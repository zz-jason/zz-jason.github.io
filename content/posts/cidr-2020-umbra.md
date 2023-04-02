---
title: "[CIDR 2020] Umbra: A Disk-Based System with In-Memory Performance"
date: 2023-04-01T08:00:00Z
categories: ["Paper Reading"]
---

## Buffer Manager

![Figure 1](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304021024851.png)

如上图所示，Buffer manager 将页面分为不同的 size class 进行管理，每个 buffer frame 需要从对应的 size class 中映射对应的物理内存：

- Size class：Umbra 按页面大小组织数据库页面，最小 64KiB，第 i + 1 类页面大小是第 i 类的两倍。理论上一个 page 可以达到整个缓冲池的大小
- Buffer manager 维护一个大小可配的 Buffer pool，Buffer pool 中可以加载属于任何 size class 的页面。buffer pool 占用可用物理内存的一半，将另一半作为查询执行的临时内存。Umbra buffer manager 的外部接口与其他传统的 Buffer manager 没有明显区别

### Buffer Pool Memory Management

在单个 buffer pool 中支持多个页面大小的主要挑战是如何解决虚拟内存碎片化的问题。

操作系统内核维护了一个页表，以透明地将用户空间进程使用的虚拟地址转换为实际 RAM 中的物理地址。Umbra 利用这个特点解决了内存 fragmentation 的问题：

- buffer manager 使用 mmap 为每个 size class 分配一个单独的虚拟内存块，每个内存区域都足够大，理论上可以容纳整个缓冲池。配置 mmap 为 MAP_PRIVATE，创建私有匿名映射，使它仅保留一段连续的虚拟地址范围，但这些虚拟地址尚未占用任何物理内存。
- 每个虚拟内存区域都被分成页面大小的块，每个块都创建一个 buffer frame，其中包含指向相应页面虚拟地址指针，这些虚拟地址指针在 buffer manger 的整个生命周期内保持不变。

由于页面大小在各个 size class 中是固定的，并且每个 size class 保留了一个单独的虚拟地址范围，因此与 size class 相关联的虚拟地址空间不会发生碎片化，只有物理内存地址会有预期中的碎片化。

当 Buffer Frame 活跃时，Buffer Manager 使用 pread 系统调用将数据从磁盘读入内存。这些数据存储在与 Buffer Frame 相关联的虚拟内存地址处，此时操作系统会从这些虚拟地址创建一个实际的映射到物理内存。

Buffer Manager 会跟踪当前保存在内存中的所有页面的总大小，根据需要将页面驱逐到磁盘来确保它们的总大小永远不会超过缓冲池的最大大小。当 Buffer Frame 被驱逐时，首先使用 pwrite 系统调用将上面的更改写回磁盘上的页面数据，接着内核就可以重用之前相关联的物理内存。在 Linux 上，可以通过将 MADV_DONTNEED 标志传递给 madvise 系统调用来实现这一点。而由于 buffer manager 中使用的内存映射没有由任何实际文件支持，因此 madvise 调用几乎不会产生开销。

Umbra 基本上采用了与 LeanStore 相同的替换策略，unpin 页面后不会立即将其从内存中驱逐。这些冷却页面随后被放置在 FIFO 队列中，并在到达队列末尾时最终被驱逐。

### Pointer Swizzling

![Figure 2: Illustration of a swizzled (top) and unswizzled (bot- tom) swip](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022007177.png)

由于页面需要序列化到磁盘中，因此在一般情况下需要通过 page id（PID）而不是内存指针来引用它们。传统的做法是采用一个全局 hash table 存储 page id 到内存指针的映射，每次读写 page 时都需要加锁访问这个全局 hash table，并发很高时会有很大的 lock contention，使其成为主要性能瓶颈。

Umbra 采用一个称为 Swip 的 64 位整数来封装 page id。通过最低位来判断这是一个内存指针还是一个 page id。如果最低位是 0 则表示内存指针，称其为 swizzled；如果是 1 则表示 page id，称其为 unswizzled。对于表示 page id 的 unswizzled Swip，Umbra 使用额外的 6 比特来表示其所属的 size class，剩余的 57 比特用来表示 page id。

Umbra 要求每个页面仅有一个 Swip，这样加载或驱逐页面时就不需要花大力气寻找其他 Swip 来更新了。对于 B+ 树来说，每个叶子节点也不能再存储它的兄弟节点。论文接下来的 versioned latch 章节就讲了如何加速 B+ 树的读写操作。

### Versioned Latches

![Figure 3: Structure of the versioned latch stored in a buffer frame for synchronization of page accesses](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022008208.png)

为了尽可能避免锁竞争，Umbra 采用和 leanstore 相同的乐观锁机制：
- 每个活动的 buffer frame 都包含一个 versioned latch，支持以 exclusive、shared、或者 optimistic 的模式获取
- versioned latch 是一个 64 位整数的原子变量，通过原子变量进行读写。其中 5 位用于存储锁状态，其余 59 位用于存储锁版本计数器
- 对 versioned latch 的状态位而言，0 表示没人上锁；原子性的修改为 1 表示 exclusive 锁。exclusive 模式最多只允许一个线程同时持有该锁，语义类似于全局互斥量
- 对 versioned latch 的版本计数器而言，任何页面修改都会递增的推进计数器的值

shared latch 的状态位比较特殊，在没有被其他线程以 exclusive 模式锁定时，才可以使用 shared 模式上锁。共享锁的状态位大于 1，存储有多少个线程持有该 latch，n+1 的值表示有 n 个线程正以共享模式持有该锁。如果所有的 5 个比特还存不下，会使用额外的 64 为整数辅助存储。

对于共享锁来说，以下操作都无法进行：
- 在持有共享锁时不允许修改页面。
- 共享锁 pin 住了 buffer manager 中关联的页面，防止其被卸载。

虽然读操作先获取了共享锁但仍有可能会失败，因为另一个线程可能会获取独占锁并同时修改页面。因此，在释放共享锁时必须进行一次验证：如果锁版本计数器发生了变化，则页面在上锁后已被并发修改，读操作需要重新启动。

另外当访问 page 时，可能出现数据没加载的情况。因为保留给 buffer frame 的虚拟内存区域始终保持有效，而读取带有 MADV_DONTNEED 标记的内存区域只会导致读到 0 字节，不会分配额外的物理内存。

乐观锁的设计初衷是优化高并发读场景，减轻高并发下的锁竞争。

与 Leanstore 不同，Umbra 还额外支持了共享锁定，主要原因是：Umbra 中的操作者通常不知道关系的分页性质。如果我们只为读者支持乐观锁定，则管道中的每个操作者都必须包含额外的验证逻辑，以防止页面在处理过程中被驱逐。此外，在 OLAP 查询中，如果其他查询写入相同的关系，则避免了频繁的验证失败。

### Buffer-Managed Relations

Umbra 中的元组（Tuple）存储在 B+ 树中，使用固定 8 字节的元组标识符（Tuple ID）作为 Key，元组标识符严格单调递增。这样的 8 字节 key 设计使得 Umbra 内部节点始终使用最小可用页面大小（64 KiB），扇出固定为 8192，使得插入任意 tuple 的处理效率都差不多。

Umbra 只有在插入期间元组无法适合现有页面时才会分配叶子结点，分配时选择可以容纳整个元组的最小页面大小作为新叶子结点的 size clase，一般来说元组不会很大，会分配到 64 KiB 的页面。Umbra 中的大多数叶子页面都是 64 KiB，有效的平衡了点查和范围查的读写性能。

Umbra 采用 PAX 的格式组织元组：固定大小的属性以列式布局存储在页面的开头，而可变大小的属性则密集存储在页面的末尾，支持可以按需压缩页面。这种页面布局对于存储在磁盘上的数据并不是最优的， Umbra 计划在将来使用其他方案，比如冷数据采用 Hyper 中的 DataBlock 格式。

B+ 树的并发访问通过对 buffer frame 上的 versioned latch 进行 optimistic latch coupling 完成。写者仅以独占模式获取 versioned latch 以拆分内部页面或将元组插入到叶子页面中，而读者仅以共享模式获取 versioned latch 以从叶子页面中读取元组或从磁盘加载子页面。在非修改遍历期间，锁以乐观模式获取，并在必要时进行验证。为了避免频繁地重试读操作、遍历整个 B+ 树，Umbra 在表扫描期间在当前叶子节点的父节点上维护一个乐观锁。只要这个乐观锁保持有效，即没有并发修改父节点，那就可以直接定位到对应叶子节点。

### Recovery

ARIES 的日志和恢复机制基本满足 Umbra 的需求，只需考虑一些特殊情况，比如下面这个例子：

考虑一个 128 KiB 的数据库文件，它当前完全被一个 128 KiB 的页面占用。将此页面加载到内存中，删除它，并创建两个新的 64 KiB 页面，这些页面重用该文件中的磁盘空间。系统崩溃时，可能只有相应的 WAL 写入磁盘，而实际的新页面数据还没写入。在恢复期间，ARIES 会从该文件读取第二个 64 KiB 页面的日志序列号，而实际上读取上来的是已删除的 128 KiB 页面的某些数据，并错误地将其解释为了日志序列号。

Umbra 解决思路比较简单，仅在 size class 内部支持磁盘复用。

## Further Considerations

### String Handling

![Figure 4: Structure of the 16-byte string headers in Umbra](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022100558.png)

因为 page size 是变长的，一个 string 也不需要被拆分成多段了，Umbra 简单的将其存成 length + data 两部分：

Umbra 将字符串分为两个部分来存储：一个包含元数据的 16 字节 header 和一个包含实际数据可变大小的 body。header 和其他固定大小的属性一样存储在页面开头的列式布局中，而实际字符串数据则存储在页面末尾，长字符串也不需分割（放不下就触发分裂，然后 buffer manager 分配一个能放下它的叶子结点）。

header 的前 4 位用于存储字符串长度，而后 12 位根据字符串长度有两种编码格式：
- 对于不长于 12 字节的短字符串，它们的数据直接存储在 header 的后 12 位内
- 后 12 位的 4 位用于存储字符串的前四个字符，允许 Umbra 快速进行一些字符串比较，剩下的 8 位存储其数据的指针或某个已知位置的偏移量

由于长字符串存储在其他物理页面上，这些页面可能没有加载到内存中，因此这些字符串需要特殊处理：Umbra 引入了三种存储类别，即持久、瞬态和临时存储。存储类别编码在字符串头部存储的偏移量或指针值的两个比特中：
- 持久存储：对持久存储的字符串引用（例如查询常量）在数据库运行时间内始终有效
- 瞬态存储：对瞬态存储的字符串引用，只有在当前工作单元正在处理时有效
- 临时存储：由查询执行实际创建的字符串（例如 UPPER 函数）具有临时存储期

### Statistics

Umbra 中支持的统计信息主要是每个表上的随机采样和每个列上可更新的 HyperLogLog。Umbra 实现了一个可扩展的在线蓄水池采样算法，相比 HyPer 仅支持定期更新样本的方式，Umbra 可以用最小的开销确保优化器始终具有最新的样本。

找时间学习学习这个在线蓄水池采样算法。

### Compilation & Execution

Umbra 大致上采用了 Hyper 的执行策略：逻辑查询计划被转换为高效的并行机器码，然后执行以获得查询结果。Umbra 在 Hyper 的基础上进一步优化了一些细节。

Umbra 采用了比 HyPer 更细粒度的物理执行计划。HyPer 的物理执行计划本质上是一个整体代码片段，Umbra 的物理执行计划则被表示为模块化状态机。以下面的 query 为例：

```sql
select count(*) from supplier group by s_nationkey
```

对上面这个 TPC-H 查询，Hyper 会采用两个 Pipeline 来执行它，第一个 Pipeline 扫描 supplier 表并执行 group by 操作，第二个 Pipeline 扫描每个 group 的数据并打印查询输出。在 Umbra 中，这些 Pipeline 进一步分解为 step，每个 step 可以是单线程的也可以是多线程的。上述 query 的 pipeline 和 step 如下图所示：：

![Figure 5: Pipelines and the corresponding steps for a simple group-by query in Umbra](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022118774.png)

这种执行方式和现在 Clickhouse、DuckDB 等基于 push 的模型非常像：在生成的代码中，每个 step 对应一个单独的函数，可以由 Umbra 的 runtime system 调用。在查询执行时，通过这些 step 完成 pipeline 内的状态转换，step 的执行由 Umbra 的查询执行器协调。多线程 step 采用 morsel-driven 的方式执行。

这样做的好处有很多：
- 可以在每次调用 step 函数后暂停查询执行，比如系统 IO 负载超过某个阈值可以及时暂停。
- 查询执行器可以在运行时检测并行 step 是否仅包含单个 morsel，这种情况下不必将所需的工作分派到另一个线程，从而避免潜在的上下文切换开销。
- 可以轻松支持 pipeline 内的多个并行 step，如上图所示

另一个和 Hyper 不同的地方是代码不是直接生成成 LLVM IR，而是在 Umbra 中实现了一个自定义的轻量级 IR，这使 Umbra 能在不依赖 LLVM 的情况下高效（省去一些额外开销，能够比 LLVM 更高效）地生成代码。

Umbra 不会立即将 IR 编译为优化后的机器码。Umbra 采用了自适应编译策略，用来权衡每个 step 的编译和执行时间。step 的 IR 先被转换为高效的字节码，由 Umbra runtime system 解释执行。对于并行 step，自适应执行引擎会跟踪执行进展以决定编译是否有益，如果是，则将 Umbra IR 转换为 LLVM IR，然后交给 LLVM JIT 编译后执行。

## Experiments

Umbra 主要和 Hyper 以及 MonetDB 做性能对比，接着对关键特点进行 micro benchmark

测试环境：
- Intel Core i7-7820X CPU with 8 physical and 16 logical cores running at 3.6 GHz.
- The system contains 64 GiB of main memory
- all database files are placed on a Samsung 960 EVO SSD with 500 GiB of storage space.

### Systems Comparison

主要测试 JOB 和 TPC-H 10GB 数据，测试方法：每个查询重复五次，选取最快的重复结果，也就是选取充分预热后的执行时间。

![Figure 6: Relative speedup of Umbra over HyPer and Mon- etDB on JOB and TPCH](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304022135620.png)

和 Hyper 相比，Umbra 的性能提升明显，主要来自自适应的 IR 编译。特别是在 JOB 上，Umbra 的 geometric mean 提升为 3.0×，在 TPC-H 上为 1.8×。在这些查询中，HyPer 实际上在查询编译上花费的时间远远超过查询执行时间，最多达到 29×。

而如果仅看纯计算开销，Umbra 也要好于 Hyper。平均而言，在 JOB 上执行时间波动约为 30%，在 TPC-H 上波动约为 10%。JOB 上的相对差异较大，大多数是由于 HyPer 和 Umbra 选择了不同的执行计划。尽管 Umbra 的基数估计要好得多，但它偶尔会选择比 HyPer 更差的逻辑计划。这是因为 Hyper 里出现了负负得正的 Join Order 优化效果。

MonetDB 相比而言就差的比较多了。在许多查询中，特别是更复杂的 JOB 查询中，MonetDB 在查询优化和代码生成方面花费了大量时间。MonetDB 偶尔也会选择糟糕的查询计划，导致查询执行时间缓慢。即使有好的计划，在 MonetDB 中执行查询通常也不如在 HyPer 和 Umbra 中高效。

### Microbenchmarks

为了证明 buffer manager 的开销很小，先禁用了 buffer manager 中的页面驱逐，这消除了将源自数据库页面的字符串实体化到内存中的必要性。然后实现了另一个存储层，它只是将表数据存储在扁平的内存映射文件中，而没有任何 buffer manager，消除了 buffer manager 本身的开销以及使用 B+ 树存储表数据的影响。

重跑 JOB 和 TPC-H 测试，对比结果：在避免字符串实例化的情况下，性能只在两个方向上略微波动，平均不到 2%；如果完全绕过  buffer manager，则平均波动不到 6%。

测试 buffer manager 的 IO 性能：生成一个包含 2.5 亿个随机 8 字节浮点值的表。随后计算这些值的总和，使用冷数据库和操作系统缓存进行测试。当绕过 buffer manager 时，Umbra 依赖操作系统将包含数据的扁平内存映射文件内容加载到内存中，达到了 1.15 GiB/s 的读取吞吐量，接近于 SSD 上可用随机访问带宽的最大值。由于 Umbra 中查询执行具有高度并行化的特性，并且不对线程之间处理页面的顺序施加任何约束，因此对 SSD 的读取访问实际上是基本随机的。当实际利用 buffer manager 时，Umbra 实现了几乎相同的读取吞吐量 1.13 GiB/s。

## 总结

这篇论文比较惊艳的地方在于使用了变长 page、针对性的实现了支持变长 page 的 buffer manager。pointer swizlling 和对应的 b+ tree 并发优化主要来自之前 LeanStore 上的研究。比较遗憾是只看见了针对 TPC-H 和 JOB 的测试。好奇 Umbra 在 OLTP 方面的表现，如果能有 YCSB 的纯读、读多写少的混合负载，以及 TPC-C 上的测试结果就更好了。