---
title: "Essential Readings on Query Execution"
date: 2023-03-10T17:00:00+08:00
categories: ["Paper Reading"]
draft: true
---

## 0. Introduction

## 1. Execution Framework

**[Volcano - An Extensible and Parallel Query Evaluation System](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf), 1994, IEEE Transactions on Knowledge and Data EngineeringFebruary**

这篇论文主要讲了 Volcano 这个查询执行系统，它能够支持数据库查询处理的可扩展性和并行性。Volcano 基于迭代式的 Pull 模型，用包括 open、next、close 等方法的标准接口实现 SQL 算子，方便添加新功能。这篇论文是首个结合可扩展性和并行性的查询执行引擎，对数据库系统设计、查询优化、并行查询执行和资源分配等有非常重要的影响。

**[MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf), 2005, CIDR**

这篇论文主要讲了 MonetDB/X100 这个查询执行系统，它能够利用现代 CPU 的高性能，提高数据库查询处理的效率 。其中的关键设计是使用向量化的数据流模型，每次处理一个批次的列数据，结合了 Volcano-style 的流水线和 MIL 的全量物化的优点 。这样可以减少不必要的数据存取、内存占用、缓存失效和分支预测错误等开销，提高指令每周期（IPC）的效率 。

**[Efficiently Compiling Efficient Query Plans for Modern Hardware](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf), 2011, VLDB**

这篇论文主要讲了一种利用 LLVM 编译框架将查询转换为紧凑、高效的机器码的方法。利用 LLVM 的优化能力，生成的代码具有良好数据局部性、无冗余指令、无虚函数调用等特点 。可以提高现代硬件条件下的查询引擎性能，性能与针对当前查询特化手写的 C++ 代码相媲美。

**[Relaxed Operator Fusion for In-Memory Databases: Making Compilation, Vectorization, and Prefetching Work Together At Last](http://www.vldb.org/pvldb/vol11/p1-menon.pdf), 2017, VLDB**

这篇论文提出了一种叫 Relaxed Operator Fusion（ROF）的新查询执行模型。ROF 可以让代码生成、向量化和 CPU Prefetch 这三种技术在内存数据库中协同工作，提高查询执行性能。其中的代码生成和向量化在上两篇论文中有更详细的介绍，CPU Prefetch 是利用软件预取指令，在数据被使用之前将其从内存加载到 CPU Cache 中，提升缓存命中率。

## 2. Optimization Methods

向量化和代码生成是两种常用的查询执行优化技术，都是为了提高 CPU 利用率和缓存命中率，从而加速查询性能。向量化是指将查询操作以批处理的方式执行，每次处理一个 vector，而不是一个 tuple。这样可以减少函数调用和分支预测错误，提高指令级并行度，并且可以利用 SIMD 指令集进行加速。代码生成是指将查询计划转换为针对特定查询和数据特征的高效机器码的方式执行。这样可以消除虚函数调用和不必要的分支预测，提升数据局部性，也可以利用编译器优化进行加速。

在上一节中我们推荐了 [MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf) 和 [Efficiently Compiling Efficient Query Plans for Modern Hardware](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf) 这两篇关于向量化和代码生成主要思路的论文，这个章节我们将深入这两个优化方向介绍几篇经典的关于向量化和代码生成优化的其他论文。

**[Vectorization vs. Compilation in Query Execution](https://15721.courses.cs.cmu.edu/spring2023/papers/10-vect-vs-comp/p5-sompolski.pdf "Vectorization vs. Compilation in Query Execution"), 2011, DaMoN**

这篇来自 VectorWise 的论文主要讲了向量化和代码生成在查询执行中的性能比较。论文使用 Ingres VectorWise 作为实验平台，对 Project、Select 和 Hash Join 这三种常见算子进行了详细的分析和测试。论文指出，代码生成和向量化都有各自优势场景，应该结合使用。

**[Data Blocks: Hybrid OLTP and OLAP on Compressed Storage using both Vectorization and Compilation](https://15721.courses.cs.cmu.edu/spring2023/papers/10-vect-vs-comp/p311-lang.pdf "Data Blocks: Hybrid OLTP and OLAP on Compressed Storage using both Vectorization and Compilation"), 2016, SIGMOD**

这篇来自 TUM、Snowflake 和 CWI 的论文讲了一种叫 Data Blocks 的压缩列式存储格式，它可以在高性能的 HTAP 数据库中减少内存占用的同时保持极高的查询性能和事务吞吐 。Data Blocks使用了向量化和代码生成两种技术来加速冷数据的查询分析。Data Blocks 还支持轻量级的压缩方案，使 OLTP 事务可以快速地访问单个元组。

**[Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf), 2018, VLDB**

这篇来自 TUM、CMU 和 CWI 的论文通过实验比较了向量化和代码生成这两种优化方式。两种方法都很高效但各有优劣。向量化可以提高 CPU Cache 命中率降低查询处理时间，代码生成得到的 CPU 指令更少，对于 CPU Cache 内的工作负载有利。除了单线程性能，作者还研究了 SIMD 和多核并行化以及不同的硬件架构对于查询执行的影响。这篇论文能够帮助读者更深刻的了解向量化和代码生成的差异，非常值得一读。

### 2.1. Vectorization

**[Implementing Database Operations Using SIMD Instructions](http://www.cs.columbia.edu/~kar/pubsk/simd.pdf), 2002, SIGMOD**

这篇论文主要讲了如何利用 SIMD 指令来加速数据库操作的执行。作者研究了数据库中各种类型的操作，例如 Scan、Aggregate、Index Access、Join等，并展示了如何使用 SIMD 指令来加快这些操作。

**[Rethinking SIMD Vectorization for In-Memory Databases](https://15721.courses.cs.cmu.edu/spring2023/papers/08-vectorization/p1493-polychroniou.pdf), 2015, SIGMOD**

这篇论文主要讲了如何利用 SIMD 指令来优化内存数据库的查询执行。作者提出了基于 SIMD 操作（比如 gather 和 scatter）的向量化设计和实现原则。作者研究了 Selection、Hash Table、Partitioning 等操作，并将它们组合起来构建 Sort 和 Join。作者在不同的硬件架构上进行了实验，发现论文中的方法相比 start-of-the-art 向量化计算方式有数量级的性能提升。

**[Make the most out of your SIMD investments: counter control flow divergence in compiled query pipelines](https://15721.courses.cs.cmu.edu/spring2023/papers/08-vectorization/lang-vldbj2020.pdf), 2020, VLDB**

这篇论文主要讲了如何解决编译 Query Pipeline 中的控制流分歧问题。控制流分歧是指在 SIMD 指令中，不同的数据元素需要执行不同的计算分支，导致向量处理单元的利用率降低。作者提出了一些策略来减少控制流分歧的影响，例如使用部分消费模式、填充空闲 SIMD 通道、重排数据元素等。作者在不同的硬件平台和查询工作负载上进行了实验，发现使用这些策略可以显著提高查询性能。

### 2.2. Code Generation

**[Generating code for holistic query evaluation](https://15721.courses.cs.cmu.edu/spring2016/papers/krikellas-icde2010.pdf), 2010, ICDE**

这篇论文提出了一个叫 Holistic Query Evaluation 的查询执行方式，通过使用定制化的代码生成来优化查询执行。Holistic Query Evaluation 会在传统的查询计算过程中加入一个源代码生成的步骤。系统根据整个查询和硬件特性，生成高效的代码模板并动态实例化它们，然后编译和执行它们

**[CPU and Cache Efficient Management of Memory-Resident Databases](https://15721.courses.cs.cmu.edu/spring2023/papers/09-compilation/pirk-icde2013.pdf "CPU and Cache Efficient Management of Memory-Resident Databases"), in ICDE, 2013**

这篇论文主要讲了如何优化内存驻留数据库管理系统（MRDBMS）的两个资源：CPU 周期和内存带宽。作者针对 HTAP 场景，提出了一种混合或部分分解存储模型（PDSM），可以根据查询特征动态地选择最佳的数据布局。作者还提出了一种基于缓存感知的查询处理技术，可以利用缓存层次结构来减少内存访问次数和提高指令级并行度。作者在不同的硬件平台和查询工作负载上进行了实验，发现使用 PDSM 和缓存感知技术可以显著提高 MRDBMS 的性能。

**[How to Architect a Query Compiler](https://15721.courses.cs.cmu.edu/spring2023/papers/09-compilation/shaikhha-sigmod2016.pdf "How to Architect a Query Compiler"), 2016, SIGMOD**

这篇论文提出了一种通过多次优化，将最开始的 SQL 执行计划一层层编译成 DSL 表示的执行策略，直到得到最底层的 C 实现。这种方式能够解决传统的类似宏展开的代码生成方式中的复杂代码逻辑，对不同层的 DSL 进行优化后最终得到的执行代码也更加高效。

### 2.3. Prefetch

### 2.4. Parallel Execution

### 2.5. Adaptive Query Execution

**[Micro Adaptivity in Vectorwise](https://15721.courses.cs.cmu.edu/spring2023/papers/09-compilation/p1231-raducanu.pdf "Micro Adaptivity in Vectorwise"), 2013, SIGMOD**

这篇论文提出了一个叫 Micro Adaptivity 的计算框架，使得 DBMS 可以根据具体的运行环境选择最优的函数实现（论文中称之为 “flavor”），使得查询性能在不同的环境中都能稳定的快。论文描述了一系列 flavor 以及影响它们各自执行性能的因素，使用了一个学习算法选择 flavor，以及他们是如何在 Vectorwise 中实现 Micro-batch Adaptivity 的。

**[Adaptive Execution of Compiled Queries](https://15721.courses.cs.cmu.edu/spring2023/papers/09-compilation/kohn-icde2018.pdf "Adaptive Execution of Compiled Queries"), 2018, in ICDE**

这篇来自 TUM 的论文讲述了一种

### 2.6. Encoded Computation

BIPie: fast selection and aggregation on encoded data using operator specialization. In: Proceedings of the 2018 International Conference on Management of Data, SIGMOD’18, pp. 1447– 1459. ACM, New York (2018)

### 2.7. Materialization

**[Materialization Strategies in the Vertica Analytic Database: Lessons Learned](https://15721.courses.cs.cmu.edu/spring2023/papers/06-execution/shrinivas-icde2013.pdf "Materialization Strategies in the Vertica Analytic Database: Lessons Learned"), 2013, ICDE**

### 2.8. Spill to External Storage

这里介绍一些论文，它们探讨了如何在资源有限的情况下，利用外存高效地完成大规模数据的存储、处理和查询。这些论文涉及了数据模型、存储格式、数据操作、执行计划和调度策略等方面的问题和解决方案。

**[Memory-Adaptive External Sorting](https://www.vldb.org/conf/1993/P618.PDF), 1993, VLDB**

**[AlphaSort: A Cache-Sensitive Parallel External Sort](https://www.vldb.org/journal/VLDBJ4/P603.pdf), 1995, VLDB**

## 3. Optimization of SQL operators

### 3.1. Scan

**[SIMD-Scan: Ultra Fast in-Memory Table Scan using onChip Vector Processing Units](http://www.vldb.org/pvldb/vol2/vldb09-327.pdf), 2009, VLDB**

### 3.2. Filter

**[Vectorized Bloom Filters for Advanced SIMD Processors](http://www.cs.columbia.edu/~orestis/damon14.pdf), 2014, DaMoN**

### 3.3. Aggregate

**[Scalable Aggregation on Multicore Processors](https://cse.hkust.edu.hk/damon2011/proceedings/p1-ye.pdf), 2011, DaMoN**

**[High Throughput Heavy Hitter Aggregation for Modern SIMD Processors](http://www.cs.columbia.edu/~orestis/damon13.pdf), 2013, DaMoN**

### 3.4. Sort

**[AA-Sort: A New Parallel Sorting Algorithm for Multi-Core SIMD Processors](https://dl.acm.org/doi/pdf/10.5555/1299042.1299047), 2007, PACT**

**[Efficient Implementation of Sorting on Multi-Core SIMD CPU Architecture](http://www.vldb.org/pvldb/vol1/1454171.pdf), 2008, VLDB**

Fast sort on CPUs and GPUs: a case for bandwidth oblivious SIMD sort. In SIGMOD, pages 351–362, 2010.

Engineering a multi core radix sort. In EuroPar, pages 160–169, 2011.

A comprehensive study of main-memory partitioning and its application to large-scale comparison- and radix-sort. In SIGMOD, pages 755–766, 2014.

**[Origami: A High-Performance Mergesort Framework](https://www.vldb.org/pvldb/vol15/p259-arman.pdf), 2021, VLDB**

### 3.5. Window Function

**[Optimization of Analytic Window Functions](http://vldb.org/pvldb/vol5/p1244_yucao_vldb2012.pdf), 2012, VLDB**

**[Efficient Processing of Window Functions in Analytical SQL Queries](https://www.vldb.org/pvldb/vol8/p1058-leis.pdf), 2015, VLDB**

### 3.6. Common Table Expression

**[Optimization of Common Table Expressions in MPP Database Systems](http://www.vldb.org/pvldb/vol8/p1704-elhelw.pdf), 2015, VLDB**

### 3.7. Join

**[Sort vs. Hash Revisited](https://15721.courses.cs.cmu.edu/spring2023/papers/12-sortmergejoins/graefe-tkde1994.pdf "Sort vs. Hash Revisited"), 1994, TKDE**

What happens during a join? dissecting CPU and memory optimization effects. In VLDB, pages 339–350, 2000.

**[Improving Hash Join Performance through Prefetching](https://www.pdl.cmu.edu/PDL-FTP/Database/icde04.pdf), 2004, ICDE**

Sort vs. hash revisited: fast join implementation on modern multicore CPUs. In VLDB, pages 1378–1389, 2009.

Design and evaluation of main memory hash join algorithms for multi-core CPUs. In SIGMOD, pages 37–48, 2011.

How soccer players would do stream joins. In: Proceedings of the ACM SIGMOD International Conference on Management of Data, SIGMOD 2011

Massively parallel sort-merge joins in main memory multi-core database systems. PVLDB, 5(10):1064–1075, June 2012.

GPU join processing revisited. In DaMoN, 2012.

Main-memory hash joins on multi-core CPUs: Tuning to the underlying hardware. In ICDE, pages 362–373, 2013.

Multicore, main-memory joins: Sort vs. hash revisited. PVLDB, 7(1):85–96, Sept. 2013.

**[Hash joins and hash teams in Microsoft SQL Server](https://www.vldb.org/conf/1998/p086.pdf), VLDB**

**[An Adaptive Hash Join Algorithm for Multiuser Environments](https://www.vldb.org/conf/1990/P186.PDF), VLDB**

**[Skew Strikes Back: New Developments in the Theory of Join Algorithms](https://15721.courses.cs.cmu.edu/spring2023/papers/13-multiwayjoins/ngo-sigmodrec13.pdf "Skew Strikes Back: New Developments in the Theory of Join Algorithms"), 2013, SIGMOD**

**[Memory-Efficient Hash Joins](http://www.vldb.org/pvldb/vol8/p353-barber.pdf), 2014, VLDB**

**[Improving Main Memory Hash Joins on Intel Xeon Phi Processors: An Experimental Approach](https://www.vldb.org/pvldb/vol8/p642-Jha.pdf), 2015, VLDB**

**[A Seven-Dimensional Analysis of Hashing Methods and its Implications on Query Processing](https://15721.courses.cs.cmu.edu/spring2023/papers/11-hashjoins/richter-vldb2015.pdf "A Seven-Dimensional Analysis of Hashing Methods and its Implications on Query Processing"), 2015, VLDB**

**[An Experimental Comparison of Thirteen Relational Equi-Joins in Main Memory](https://15721.courses.cs.cmu.edu/spring2023/papers/11-hashjoins/schuh-sigmod2016.pdf "An Experimental Comparison of Thirteen Relational Equi-Joins in Main Memory"), 2016, SIGMOD**

**[LevelHeaded: A Unified Engine for Business Intelligence and Linear Algebra Querying](https://15721.courses.cs.cmu.edu/spring2023/papers/13-multiwayjoins/aberger-icde2018.pdf "LevelHeaded: A Unified Engine for Business Intelligence and Linear Algebra Querying"), 2018, ICDE**

**[Adopting Worst-Case Optimal Joins in Relational Database Systems](https://15721.courses.cs.cmu.edu/spring2023/papers/13-multiwayjoins/p1891-freitag.pdf "Adopting Worst-Case Optimal Joins in Relational Database Systems"), 2020, VLDB**

**[To Partition, or Not to Partition, That is the Join Question in a Real System](https://15721.courses.cs.cmu.edu/spring2023/papers/11-hashjoins/bandle-sigmod21.pdf "To Partition, or Not to Partition, That is the Join Question in a Real System"), 2021, SIGMOD**

### 3.8. Hash table

Cuckoo hashing. J. Algorithms, 51(2):122–144, May 2004.

Architecture-conscious hashing. In DaMoN, 2006.

Efficient hash probes on modern processors. In ICDE, pages 1297–1301, 2007.


## 4. Execution Scheduling

**[Self-Tuning Query Scheduling for Analytical Workloads](https://15721.courses.cs.cmu.edu/spring2023/papers/07-scheduling/wagner-sigmod21.pdf "Self-Tuning Query Scheduling for Analytical Workloads"), 2021, SIGMOD**

**[Scaling Up Concurrent Main-Memory Column-Store Scans: Towards Adaptive NUMA-aware Data and Task Placement](https://15721.courses.cs.cmu.edu/spring2023/papers/07-scheduling/p1442-psaroudakis.pdf "Scaling Up Concurrent Main-Memory Column-Store Scans: Towards Adaptive NUMA-aware Data and Task Placement"), 2015, VLDB**

Large-scale cluster management at Google with Borg, 2015

**[Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age](https://15721.courses.cs.cmu.edu/spring2016/papers/p743-leis.pdf), 2014, SIGMOD**

**[Task Scheduling for Highly Concurrent Analytical and Transactional Main-Memory Workloads](https://15721.courses.cs.cmu.edu/spring2023/papers/07-scheduling/psaroudakis_adms13.pdf "Task Scheduling for Highly Concurrent Analytical and Transactional Main-Memory Workloads"), 2013,  ADMS**

Apache Hadoop YARN: Yet Another Resource Negotiator. In Proceedings of the 4th annual Symposium on Cloud Computing, 2013

Work stealing strategies for parallel stream processing in soft real-time systems. In Proc. of the 25th Int’l Conf. on Architecture of Computing Systems, pages 172–183, 2012

OpenMP task scheduling strategies for multicore NUMA systems. Int’l Journal of High Performance Computing Applications, 26(2):110–124, 2012.

Static scheduling algorithms for allocating directed task graphs to multiprocessors. ACM Computing Surveys, 31(4):406–471, 1999.

Scheduling Threads for Low Space Requirement and Good Locality. In Proc. of the 11th Annual ACM Symposium on Parallel Algorithms and Architectures, pages 83–95, 1999.

Dynamic scheduling strategies for shared-memory multiprocessors. In Proc. of the 16th Int’l Conf. on Distributed Computing Systems, pages 208–215, 1996.

Task clustering and scheduling for distributed memory parallel architectures. IEEE Transactions on Parallel and Distributed Systems, 7(1):46–55, 1996

Using Runtime Measured Workload Characteristics in Parallel Processor Scheduling. In Workshop on Job Scheduling Strategies for Parallel Processing. Springer, 155–174., 1996

Lazy task creation: a technique for increasing the granularity of parallel programs. In Proc. of the 1990

## 5. Resource Management

这里介绍一些论文，它们探讨了如何在多租户、多查询、多任务的场景下，合理、动态、协调地分配和调度 CPU、内存、磁盘、网络等资源，以提高系统性能和可靠性的解决方案。