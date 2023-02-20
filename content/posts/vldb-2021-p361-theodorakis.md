---
title: "[VLDB 2021] Scabbard: Single-Node Fault-Tolerant Stream Processing"
date: 2023-02-19T06:27:23Z
categories: ["Paper Reading"]
draft: false
---

这篇论文解决的问题：因为单机磁盘 IO 受限，通常单节点流处理系统的故障恢复依赖于像 kafka 这样的上游分布式系统，使得单节点流处理系统的部署和运维变的非常复杂。

> due to the limited I/O bandwidth of a single-node, it becomes infeasible to persist all stream data and operator state during execution. Instead, single-node SPEs rely on upstream distributed systems, such as Apache Kafka, to recover stream data after failure, necessitating complex cluster- based deployments.



提出的解决办法：基于 operator 的 selectivity 对那些会 discard 数据的 operator 进行落盘，达到降低磁盘 IO 的目的，使得单节点流处理系统能够自己在单机上进行故障恢复。

> Within the operator graph, Scab- bard determines when to persist streams based on the selectivity of operators: by persisting streams after operators that discard data, it can substantially reduce the required I/O bandwidth.



流处理系统的应用场景：

> Stream processing enables applications ranging from real-time credit card **fraud detection** [36] to **click-stream analytics** [2, 20, 44], and **live mining** of sensor data [28, 29].



分布式流处理系统运维复杂，像 RDMA 这样低延迟高带宽的硬件能力又无法完全在性能上体现出来：

> To accommodate growing data amounts, distributed stream pro- cessing engines (SPEs) such as Flink [19] and Spark Streaming [108] scale out processing to a cluster of nodes through appropriate data partitioning [19, 108] – at substantial operational cost.
>
> While high-speed networking such as RDMA [16, 59] provides 200 Gbps per-port bandwidth with microsecond latencies [11], which allows for fast stream ingestion and remote storage [63], existing cluster-based SPEs cannot saturate these fast interconnects [109].



随着硬件规格的提升和新硬件的出现，单机流处理系统的处理能力和处理速度已经能够胜任大多数计算场景。并且因为不再需要考虑分布式处理，单机流计算系统结合 JIT 后相比分布式流计算系统使用更少的资源，运维负担也更低：

> With the rise of parallel hardware, such as multi-core CPUs and GPUs, we witness scale-up designs for single-node SPEs [65, 76, 77, 96, 110] that rival the performance of cluster-based deployments.
>
> In contrast, single-node SPEs yield up to an order of magnitude higher per- formance with fewer resources and lower maintenance costs [86]. Such high execution efficiency is achieved by avoiding abstractions for distributed processing and incorporating techniques such as just-in-time (JIT) code generation [47, 96].



为什么生产环境中单机流处理系统不多：due to a lack of fault-tolerance mechanisms that guarantee correct results after system failure [6, 53, 94].



![Figure 1: Data ingestion rates for stream queries in a single-node SPE (LightSaber) vs. a persistent message queue (Apache Kafka)](/images/vldb-2021-p361/figure-1.png)

为什么不用分布式流处理系统相同的持久化方式：

> While the same persistence approaches could be used for single- node SPEs, relying on an external cluster-optimized system for persistence, such as Kafka, **counteracts the benefits of a single-node deployment**.
>
> A single Kafka node cannot support the performance requirements of modern single-node SPEs.
>
> a single Kafka node can only ingest data streams at rates that are several orders of magnitude lower than LightSaber’s query performance and does not even saturate the SSD bandwidth (indicated by a dashed line). While it is possible to scale out the Kafka deployment to increase its throughput lin- early through stream partitioning, **this requires a large cluster (with associated maintenance costs) just to support a single SPE node**.



前人做过一些尝试，但是发现性能瓶颈出现在了单机的磁盘 IO 上：

> A strawman solution is to design a “self-contained” fault-tole- rance mechanism for a single-node SPE in which the SPE persists all input data streams (and temporary processing state) to stable storage to recover processing after failure. We observe that, for such an approach, disk I/O bandwidth becomes the limiting factor for a majority of queries in Fig. 1, capping performance to 950 MB/s. While I/O bandwidth can be increased through hardware solutions (e.g., NVMe SSDs [106] or RAID [82]), this also increases costs.



