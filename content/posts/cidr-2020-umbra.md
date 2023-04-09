---
title: "[CIDR 2020] Umbra: A Disk-Based System with In-Memory Performance"
date: 2023-04-01T08:00:00Z
categories: ["Paper Reading"]
---

## 简介

这篇论文主要介绍了 TUM 研发的基于 SSD 的通用数据库 Umbra，可以在不牺牲性能的情况下处理任意大小的数据集。它被 TUM 看做是内存数据库 HyPer 的继任者。Umbra 有很多关键实现，比如采用不定长 page 并为其针对性设计实现了 buffer manager，采用了 pointer swizzling、versioned latch 等优化提升 Umbra 在多核上的 scalability，实现了高效的 log 和 recover 算法，和 HyPer 一样采用代码生成等。

TUM 在 HyPer 后开始了基于 SSD 的数据库系统研究，一开始的项目叫 LeanStore（LeanStore 采用定长 page），Umbra 是在 LeanStore 的基础上直接演进出来的，许多技术设计和 LeanStore 差不多，建议大家在看 Umbra 的时候也提前看看 LeanStore 的论文，一些技术实现在 LeanStore 中会讲的更细。

这篇文章假设大家已经对 LeanStore 比较熟悉了，不了解 LeanStore 的朋友也可以看看我之前写的这篇文章：TODO

## Buffer Manager

![Figure 1](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304021024851.png)

LeanStore 已经实现了足够高效的 Buffer Manager，但可惜它管理的 page 是定长的，虽然性能优异，但基于定长 page 开发的系统会引入额外的机制来处理可变大小的 tuple，带来了额外的系统复杂性和性能开销。

Umbra 选择了变长 page，如上图所示，Umbra 的 Buffer manager 将 page 分为不同的 size class 进行管理，不同 size class 的 page 大小不同，最小的 size class 是 64 KB，size class i+1 的 page 大小是 size class i 的 2 倍，这样倍增上去，最大的 page 可以把整个 buffer pool 装下。

Buffer Manager 只有一个全局 Buffer Pool，管理所有的 size class。整个 Buffer Pool 的可用内存是整体配置的，不需要单独为每个 size class 配置内存容量。初始默认 Buffer Pool 只能使用可用内存的 50%，剩下的一半供查询执行使用。

### Buffer Pool Memory Management

Buffer Pool 中支持多个 size class 的主要挑战就是如何解决内存碎片化的问题。

Umbra 通过操作系统的虚拟地址和物理地址映射来解决这个问题。操作系统通过页表将用户的虚拟内存地址转换为实际的物理地址。这使得连续的虚拟内存可以分散的存储在碎片化的物理内存中，同时虚拟内存分配也可以和物理内存分配解耦开，用户程序可以在不实际分配物理内存以及不建立虚拟内存到物理内存地址映射的情况下分配一块虚拟内存。

Umbra 的 Buffer Manager 使用 mmap 来实现上述目标。具体来说，每个 size class 都被分配了一个足够装下整个 buffer pool 大小的虚拟内存，通过配置 mmap 系统调用使其仅分配虚拟内存地址不分配实际物理内存，然后将每个 size class 内的虚拟内存按照 page size 切分成一个个 chunk，每个 buffer frame 都包含一个指向对应 chunk 的内存指针。这些内存指针创建出来后就不再改变，后续 buffer manager 在需要时就将 page 从磁盘加载到这个虚拟内存地址对应的物理内存中。

**物理内存分配**：Umbra 通过 pread 系统调用将磁盘上的 page 数据读到 buffer frame 中，此时操作系统才会实际分配物理内存（可能不连续），创建虚拟地址到这些物理内存地址的映射关系。

**物理内存释放**：当 page 从 buffer pool 中剔除时，Umbra 先通过 pwrite 系统调用将 buffer frame 中的数据写回磁盘文件，然后通过 madvise 系统调用传入 MADV_DONTNEED 标志，让操作系统回收掉 buffer frame 背后的物理内存。因为一开始的 mmap 系统调用没有实际映射磁盘文件，madvise 系统调用的开销可以忽略。

Buffer Manager 会在运行中追踪整体的物理内存使用情况确保 buffer pool 不超过配置的容量。Umbra 采用了和 LeanStore 相同的缓存替换策略，内存 page 会先放入 cooling stage 的 FIFO 队列头部，到达队列末尾时才从内存驱逐。

### Pointer Swizzling

![Figure 2: Illustration of a swizzled (top) and unswizzled (bot- tom) swip](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022007177.png)

由于页面需要序列化到磁盘中，因此在一般情况下需要通过 page ID（PID）而不是内存指针来引用它们。传统的做法是采用一个全局哈希表来存储 page ID 和其对应的内存指针，每次读写 page 时都需要获取 latch 以访问这个全局哈希表，并发很高时会有很大的 latch contention，使其成为主要性能瓶颈。

不同的是 Umbra 采用了变长 page，每个 page 位于不同的 size class 中，需要将 size class 也编码进 Swip 存储的内存地址中。

和 LeanStore  一样，Umbra 采用了 pointer swizzling 的方案来解决这个问题。内存中每个 page 通过 Swip 来访问子节点。Swip 是一个 64 位整数，最低位是 0 表示它是内存指针，最低位是 1 表示它是 page ID（对应的 page 需要 buffer manager 从 cooling stage 拿出来或从磁盘加载上来）。对于表示 page ID 的 Swip，Umbra 使用额外的 6 比特来表示其所属的 size class，剩余的 57 比特用来表示实际 page ID，也就是说 page ID 的范围是 0 到 2^57-1，也足够用了。

和 LeanStore 一样，Umbra 要求每个页面仅有一个 Swip，方便缓存替换更新 Swip 值。

### Versioned Latches

![Figure 3: Structure of the versioned latch stored in a buffer frame for synchronization of page accesses](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022008208.png)

和 LeanStore 一样，Umbra 采用了 optimistic latch 机制来尽可能避免 latch contention。不一样的是，Umbra 这里采用一个 64 位整数的原子变量实现了 versioned latch，这个 versioned latch 支持采用 exclusive、shared 或 optimistic 的模式上锁。

versioned latch 的 5 比特位用于存储锁的状态，0 表示没人上锁，1 表示 exclusive 锁，2 及以上的值表示 shared 锁。这里的 excluseive 和 shared 的语义和 RWMutex 等价。在没有被其他线程以 exclusive 模式锁定时才可以 shared 模式上锁才会成功。shared 模式下的状态位的值 -1 表示有多少个线程以 shared 模式持有该 latch，如果所有的 5 个比特还存不下，会使用额外的 64 为整数辅助存储。

versioned latch 剩余的 59 位用于存储版本计数器，每次修改被保护的数据，都需要递增计数器的值。这样当某个线程以 exclusive 模式上锁修改完 page 后，其他以 optimistic 模式上锁（实际上不会修改这个原子变脸）线程通过比对数据读取前后锁版本，锁版本发生了变化表明他们读取的 page 在这期间发生了变化，需要重试该 page 上的读操作。

和 LeanStore 一样，shared 模式下只能读不能修改 page，如果一个 page 被至少 1 个线程以 shared 模式上锁读，则这个 page 不能被 buffer manager 进行缓存替换。

与 LeanStore 不同，Umbra 为读操作支持了 shared 模式上锁，LeanStore 仅为读支持 optimistic 模式。原因是 optimistic 模式需要验证数据读取前后的锁版本，如果锁版本发生变化需要进行重试。这种机制对于那些不清楚有多少读写比重的数据，或者对于那些重试代价高昂的 OLAP 查询来说性能损失是巨大的，不如读之前就上个 shared latch。

值得注意的是 Umbra 的 versioned latch 在后面演进成了 Hybrid-Latch，发表在了 VLDB 2023 的这篇论文中，里面通过伪代码详细介绍了各个模式的 latch 实现，论文不长，推荐感兴趣的朋友阅读一下原文：《[Scalable and Robust Latches for Database Systems](https://db.in.tum.de/~giceva/papers/damon_latches.pdf?lang=de)》。

### Buffer-Managed Relations

Umbra 中的 tuple 存储在 B+ 树中，使用固定 8 字节的 tuple ID 作为 Key，tuple ID 严格单调递增。这样的 8 字节 key 设计使得 Umbra 内部节点始终使用最小可用页面大小（64 KiB），扇出固定为 8192，使得插入任意 tuple 的处理效率都差不多。

Umbra 只有在插入的 tuple 无法放入现有 page 时才会分配新的叶子结点，在能够容纳整个 tuple 的最小 size class 中分配 page，一般 tuple 不会很大，基本都会分配到 64 KiB 的 page。Umbra 中大多数叶子页面都是 64 KiB，有效的平衡了点查和范围查的读性能。

Umbra 采用 PAX 的格式组织元组：固定大小的字段以列式布局存储在页面的开头，而可变大小的字段则密集的存储在页面的末尾，在插入的时候可以按需进行 compaction。这种页面布局对于存储在磁盘上的数据并不是最优的，获取部分字段可能导致所有字段都加载到内存中，Umbra 计划在将来使用其他方案进行优化，比如冷数据采用 Hyper 中的 DataBlock 格式，参考：《[Data Blocks: Hybrid OLTP and OLAP on Compressed Storage using both Vectorization and Compilation](https://db.in.tum.de/downloads/publications/datablocks.pdf)》。

B+ 树的并发访问通过对 buffer frame 上的 versioned latch 进行 optimistic latch coupling 完成。写线程以 exclusive 模式获取 versioned latch，拆分内部页面或将元组插入到叶子页面中，读线程以 shared 模式获取 versioned latch，从叶子页面中读取 tuple 或从磁盘加载子页面。在非修改遍历期间，以 optimistic 模式获取 versioned latch。

因为 B+ tree 不再有指向兄弟节点的指针了，Umbra 根据 fence key 实现 range scan。而在 range scan 期间，为了避免代价高昂的读重试，以及不必要的整树遍历，读线程会维护父节点的 optimistic latch，当父节点的 optimistic versioned latch 没有发生变化时，读完一个叶子结点后就可以直接从父节点开始读兄弟叶子结点。

### Recovery

Umbra 采用 ARIES 恢复算法。因为采用了变长 page，在磁盘空间复用时不能让多个小 page 复用大 page 的磁盘空间。比如下面这个例子：

一个 128 KiB 的数据库文件，它当前完全被一个 128 KiB 的页面占用。将此页面加载到内存中，删除它，并创建两个新的 64 KiB 页面，这些页面重用该文件中的磁盘空间。系统崩溃时，可能只有 WAL 写入磁盘了，而实际的新页面数据还没写入。在恢复期间，ARIES 会从该文件读取第二个 64 KiB 页面的 LSN，而实际上读取上来的是已删除的 128 KiB 页面的某些数据，并错误地将其解释为了 LSN，导致后续故障恢复出现异常情况。

Umbra 解决思路比较简单，仅在 size class 内部支持磁盘空间复用。

## Further Considerations

Umbra 最大的特点就是变长 page 和对应的 buffer manager。其他模块都需要在这个基础上进行或多或少的适配工作，尤其是字符串和统计信息收集。

### String Handling

![Figure 4: Structure of the 16-byte string headers in Umbra](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022100558.png)

因为 page size 是变长的，一个 string 也不需要被拆分成多段了，Umbra 简单的将其存成 length + data 两部分：

Umbra 将字符串分为两个部分来存储：一个包含元数据的 16 字节 header 和一个包含实际数据，可变大小的 body。header 和其他固定大小的属性一样存储在页面开头的列式布局中，而实际字符串数据则存储在页面末尾，长字符串也不需分割（放不下就触发分裂，然后 buffer manager 分配一个能放下它的叶子结点）。

header 的前 4 位用于存储字符串长度，而后 12 位根据字符串长度有两种编码格式：
- 对于不长于 12 字节的短字符串，它们的数据直接存储在 header 的后 12 位内
- 后 12 位的 4 位用于存储字符串的前四个字符，允许 Umbra 快速进行一些字符串比较，剩下的 8 位存储其数据的指针或某个已知位置的偏移量

由于长字符串存储在其他物理页面上，这些页面可能没有加载到内存中，因此这些字符串需要特殊处理：Umbra 引入了三种存储类别，即持久、瞬态和临时存储。存储类别编码在字符串头部存储的偏移量或指针值的两个比特中：
- 持久存储：对持久存储的字符串引用（例如查询常量）在数据库运行时间内始终有效
- 瞬态存储：对瞬态存储的字符串引用，只有在当前工作单元正在处理时有效
- 临时存储：由查询执行实际创建的字符串（例如 UPPER 函数）具有临时存储期

### Statistics

Umbra 中支持的统计信息主要是每个表上的随机采样和每个列上可更新的 HyperLogLog。Umbra 实现了一个可扩展的在线蓄水池采样算法，参考：《[Scalable Reservoir Sampling on Many-Core CPUs](https://altan.birler.co/static/papers/2019SIGMOD_ScalableReservoirSampling.pdf)》。相比 HyPer 仅支持定期更新样本的方式，Umbra 可以用最小的开销确保优化器始终具有最新的样本。

### Compilation & Execution

Umbra 大致上采用了 HyPer 的执行策略：逻辑查询计划被转换为高效的并行机器码，然后执行以获得查询结果。Umbra 采用了比 HyPer 更细粒度的物理执行计划。HyPer 的物理执行计划本质上是一个整体代码片段，Umbra 的物理执行计划则被表示为模块化状态机。以下面的 query 为例：

```sql
select count(*) from supplier group by s_nationkey
```

对上面这个 TPC-H 查询，HyPer 会采用两个 Pipeline 来执行它，第一个 Pipeline 扫描 supplier 表并执行 group by 操作，第二个 Pipeline 扫描每个 group 的数据并打印查询输出。在 Umbra 中，这些 Pipeline 进一步分解为 step，每个 step 可以是单线程的也可以是多线程的。上述 query 的 pipeline 和 step 如下图所示：：

![Figure 5: Pipelines and the corresponding steps for a simple group-by query in Umbra](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022118774.png)

在生成的代码中，每个 step 对应一个单独的函数，可以由 Umbra 的 runtime system 调用。在查询执行时，通过这些 step 完成 pipeline 内的状态转换，step 的执行由 Umbra 的查询执行器协调。多线程 step 采用 morsel-driven 的方式执行。

这样做的好处有很多：
- 可以在每次调用 step 函数后暂停查询执行，比如系统 IO 负载超过某个阈值可以及时暂停。
- 查询执行器可以在运行时检测并行 step 是否仅包含单个 morsel，这种情况下不必将所需的工作分派到另一个线程，从而避免潜在的上下文切换开销。
- 可以轻松支持 pipeline 内的多个并行 step，如上图所示

另一个和 HyPer 不同的地方是代码不是直接生成成 LLVM IR，而是在 Umbra 中实现了一个自定义的轻量级 IR，这使 Umbra 能在不依赖 LLVM 的情况下高效（省去一些额外开销，能够比 LLVM 更高效）地生成代码。

Umbra 不会立即将 IR 编译为优化后的机器码。Umbra 采用了自适应编译策略，用来权衡每个 step 的编译和执行时间。step 的 IR 先被转换为高效的字节码，由 Umbra runtime system 解释执行。对于并行 step，自适应执行引擎会跟踪执行进展以决定编译是否有益，如果是，则将 Umbra IR 转换为 LLVM IR，然后交给 LLVM JIT 编译后执行。

## Experiments

![Figure 6: Relative speedup of Umbra over HyPer and Mon- etDB on JOB and TPCH](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022135620.png)

作者测试了 TPC-H 和 JOB 两个 Benchmark 10GB 的数据，每个查询重复五次，选取最快的重复结果（也就是充分预热后的结果）。和 Hyper 相比，Umbra 的性能提升明显，主要来自自适应的 IR 编译。特别是在 JOB 上，Umbra 的 geometric mean 提升为 3.0×，在 TPC-H 上为 1.8×。在这些查询中，HyPer 实际上在查询编译上花费的时间远远超过查询执行时间，最多达到 29×。

而如果仅看纯计算开销，Umbra 也要好于 Hyper。平均而言，在 JOB 上执行时间波动约为 30%，在 TPC-H 上波动约为 10%。JOB 上的相对差异较大，大多数是由于 HyPer 和 Umbra 选择了不同的执行计划。尽管 Umbra 的基数估计要好得多，但它偶尔会选择比 HyPer 更差的逻辑计划。这是因为 Hyper 里出现了负负得正的 Join Order 优化效果。

作者还单独测试了 buffer manager，感兴趣的朋友可以详细阅读论文这一节，本文不再展开。