---
title: "[VLDB 2023] What Modern NVMe Storage Can Do, And How To Exploit It: High-Performance I/O for High-Performance Storage Engines"
date: 2023-08-04T00:00:00Z
categories:
  - Paper Reading
  - Storage
draft: true
---

![](featured.jpg)

> 2023-07-30，鸡鸣驿，据说这里也是大话西游取景地
## 简介

这是 LeanStore 新的一篇 VLDB 论文，它在 LeanStore 的基础上继续优化，使其充分能够利用多块 NVMe SSD 的 IOPS 和带宽，提供尽可能高的读写吞吐。LeanStore 主要设计来面向 out-of-memory 的工作负载，它能充分利用 NVMe SSD 的 IOPS 和读写带宽，相比纯内存数据库来说性能差不多，同时在存储成本上又比内存便宜许多，对于那些需要极强单机性能的应用场景来说，是个非常不错的数据库选择。LeanStore 的相关介绍可以参考 《LeanStore: In-Memory Data Management beyond Main Memory》这篇论文，或者我之前写的一篇论文笔记：《[\[ICDE 2018\] LeanStore: In-Memory Data Management Beyond Main Memory](https://zhuanlan.zhihu.com/p/619669465)》

这篇论文对 LeanStore 的代码改造和架构变化并不是特别的多。最主要的贡献，一方面是讨论了如何充分发挥多块 SSD 磁盘性能的最佳实践，比如 Page Size 调整成 4KB，使用 SPDK，使用 user-threads 或者 coroutine 等。然后展示了从处理用户请求，到 Buffer Manager 内存管理，再到磁盘 IO 操作各个环节如何充分发挥多核 CPU 的并行能力，如何降低系统调用开销，最终把瓶颈落回到 SSD 上的技术方案。

## What Modern NVMe Storage Can Do

作者这里采用三星 PM1733 SSD 做了许多基础测试以评估如何达到这些 SSD 的理论 IOPS 和带宽。总共 8 块 NVMe SSD，内存数据 10GB，磁盘数据 100GB。
### Drive Scalability

![](fig-2.png)

单块 PM1733 SSD 在 4KB 数据块的随机读 IOPS 能够达到 1.5M，8 块这样的 SSD 总的 IOPS 能够达到 12M，也就是 1200 万的 IOPS。作者的测试发现，总的 IOPS 随着 SSD 的数量是线性提升的，实测下来 8 块 SSD 的总 IOPS 比官方说明的 12M 还要多，达到了 12.5M

事务工作负载通常包含大量写入，而 SSD 的读写速度又是非均衡的，因此作者进行了 SSD 的读写混合测试，上图 b 展示了这些 SSD 的总 IOPS 随着读比例的提升而提升。

### The Case for 4KB Pages

和 PMEM 不同，读写 SSD 通常以 Page 为粒度，Page Size 的选择非常关键。许多数据库系统都使用比较大的 Page Size，比如 PG、SQL Server、Shore-MT 等采用 8KB Page，MySQL 使用 16KB 的 Page，WiredTiger 更是采用了 32KB 的 Page。在之前的 LeanStore 工作中，作者发现采用 16KB 的 Page 更有利于 in-memory 的工作负载，因此 LeanStore 一开始的 Page 也是 16KB。更大的 Page Size 还有一个好处是能够减少 Buffer Pool 中的 Page Entry，降低缓存维护负担。但 Page Size 过大带来的坏处是 IO 放大，比如 16KB Page 配置下，读写 100KB 的数据引起的放大是 16KB/100=160 倍，而如果采用 4KB 的 Page 配置，读写放大就能降低为原来的 1/4，也就是 40 倍。

![](fig-3.png)

如上图所示,作者测试发现，对于 SSD 来说最佳的 Page Size 应该是 4KB，在这样的配置下它们能够提供最高的随机读 IOPS 和最低的延迟，同时也能尽可能打满 SSD 的读写带宽。

不过因为大多现有数据库的瓶颈不在 IOPS 上，仅仅将 Page Size 设置为 4KB 并不能立马看到收益，还需要配合其他优化才行。

### SSD Parallelism

![](fig-4.png)

SSD 是个内部高度并行化的设备，拥有多个 channel 连接到不同的闪存颗粒上同时进行数据读写。SSD 随机读延迟在 100us 级别，采用同步 IO 只能获得 10K 的 IOPS，要想充分利用 SSD 内部的并行 IO，需要使用异步 IO 发送尽可能多的 IO 请求给 SSD。从上图作者的测试结果来看，当同时处理 1000 个 IO 请求时能够获得非常不错的 IOPS，当同时处理 3000 个 IO 请求时才能完全发挥这 8 块 NVMe SSD 的 IOPS。

对于数据库系统来说，最大的一个挑战就是如何管理这么高的 IO 请求。

### I/O Interfaces

![](fig-5.png)

作者讨论了 4 个 Linux 上常用的 IO 库：POSIX pread/pwrite、libaio、io_uring 以及 SPDK。不管使用哪个库，在 NVMe SSD 上读写数据的最终过程都是将用户的 IO 请求发送给 NVMe 的 submission queue 中，当读写请求处理完后，会将这些事件信息发送到 completion queue 中，并根据需要进行中断处理。这里不会过多介绍前面几种 IO 接口，感兴趣的朋友可以阅读查阅相关文献了解更详细的信息。

在这些 IO 接口中，SPDK 拥有最好的性能和 CPU 消耗。SPDK 会直接在用户态分配 NVMe 的 submission 和 completion queue，它不支持中断，用户程序需要从 completion queue 中 poll 相关事件以完成 IO 请求，它完全 bypass 了操作系统内核，包括文件系统和 page cache 等。从作者的实验结果来看，SPDK 拥有最好的 IO 性能，能以最小的 CPU 消耗达到 SSD 的 IOPS 瓶颈。

### A Tight CPU Budget

我们要到达的 IOPS 目标很高，但是可用的 CPU 资源却非常有限。作者采用的 AMD CPU 是 2.5GHz 64 核的，算下来要达到 12M 的 IOPS，平均每个 IO 只有约 13k 的 CPU 时钟周期。

作者使用 fio 测试的过程中，发现 fio 因为使用了基于中断的 IO 接口，它并不能打满这些 SSD 的总 IOPS。这也从侧面说明了数据库系统要想充分利用多块 SSD 提供的 IO 能力有多么困难。

![](fig-6.png)

如上图所示，使用 io_uring 需要 32 线程才能打满这些 SSD 的 IOPS，而使用 SPDK 只需要 3 个线程。如果不用 SPDK 那么至少一半的 CPU 时钟周期都需要花费在 IO 请求上，
剩下一半的时钟周期留给了数据库其他操作比如查询处理、索引遍历、并发控制、WAL、Buffer Manager 等显然是不够的，SPDK 成了打满磁盘 IOPS 的必选项。

### Implications for High-Performance Storage Engines

LeanStore 虽然专门面向 NVMe SSD 优化，但还是不能充分发挥多块 SSD 的 IOPS，从上面的分析来看，要想充分发挥这些 SSD 的性能，还需要继续优化。

LeanStore 使用一组工作线程用来处理每个用户事务，每个工作线程对应一个操作系统线程，使用同步的  pread 接口从 SSD 读取缺失的 page，这里既有频繁的用户态-内核态上下文切换，又因为 pread 接口的原因该线程会被阻塞无法处理其他事务。正如之前实验结果看到的，我们需要上千个并发事务才能打满这些 SSD 的 IOPS，也就对应了上千个并发的工作线程，当有上千个工作线程的时候，他们对操作系统的线程争用会引发大量的上下文切换开销，反而还会降低性能。

除了处理用户事务的工作线程以外，LeanStore 还有专门的 Page Provider 线程用来寻找 code page 完成 page eviction。

## How to Exploit NVMe Storage

### Design Overview and Outline

![](fig-7.png)

要充分利用 SSD 的 IOPS，需要同时处理上千个用户事务，发送上千个 IO 请求，在传统的 thread-to-transaction 执行模式下，就意味着需要上千个线程同时运行，这显然是不可接受的。LeanStore 采用了 boost 提供的 lightweight cooperative thread，也有人把它叫做纤程（[fiber](https://en.wikipedia.org/wiki/Fiber_(computer_science))），降低了上下文切换开销。

![](fig-8.png)

如上图所示，LeanStore 在每个工作线程上实现了一个 scheduler 和对应的 user task 队列，每个 user task 对应一个纤程，通过 boost fcontext 实现了用户态的 task switch，task switch 开销在 20 个 CPU 时钟周期左右，相比内核态的上下文切换开销需要上千 CPU 始终周期来说非常轻量。使用纤程能够同时处理上千并发事务，发送上千 IO 请求到 SSD，同时对 LeanStore 代码重构来说也更加简单。

为了实现 mutli-task 调度，当发生 page fault，或者没有 free page，或者用户 task 执行完成的时候都会 yield 控制权给 scheduler，同时为了避免工作线程阻塞在锁等待上，作者修改了所有的锁实现，使得发生锁等待后最终也能将控制权移交给 scheduler。

非阻塞 IO，采用 non-preemptive task 意味着 pread/pwrite 这样的同步 IO 接口不能再用了，作者使用了 libaio、io_uring、SPDK 这样的异步 IO 接口来异步提交 IO 请求到 SSD queue 中，当 task 遇到 page fault 时，该工作线程在将 IO 请求提交到 IO backend 后，对应的 task 会将请求 yield 回 scheduler，使其继续执行后面的其他 user tasks。当 IO 结束后，工作线程最终挑选该 task 以完成后续事务执行。

## 实验结果

## 总结