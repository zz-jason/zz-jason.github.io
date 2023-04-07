---
title: "[VLDB 2023] Scalable and Robust Latches for Database Systems"
date: 2023-04-05T00:00:00Z
categories: ["Paper Reading"]
---

## Introduction

Efficient and scalable synchronization is one of the key require- ments for systems that run on modern multi-core processors.

但是虽然有很多类型的 Lock，但是却没有人详细研究过什么样的 Lock 适用于数据库的工作负载，主要原因是数据库的工作负载范围很大，有 write-heavy OLTP 事务，也有 read-only 的 OLAP 查询，甚至是两者都有的 HTAP 负载。

在设计 Umbra 的时候，作者开始调研不同的锁机制和他们之间的效率差异。首先第一个发现是不可能有一种锁机制在所有工作负载和所有硬件环境中都是最优的。How- ever, we noticed that there are some re-occurring best practices for locking and synchronization。于是作者们首先总结了数据库在 Lock 上的需求，然后 address them by analyzing and evaluating different locking techniques accordingly。

那对数据库友好的 Lock 需要具备什么样的功能呢？

读是需要又快有能 scale 的：
>  In general, most database workloads, even OLTP transac- tions, mostly read data, and thus reading should be fast and scalable.

在 LLVM JIT 的情况下，纯粹使用 OS 提供的 Lock 不太行：
> Many modern in-memory database systems compile queries to efficient machine code to keep the latency as low as possible [26]. A lock should therefore integrate well with query compilation and avoid external function calls. This requirement makes pure OS- based locks unattractive for frequent usage during query execution.

因为 Lock 需要保护一些细粒度的数据结构，Lock 本身的空间开销应该尽量小：
> To protect fine-granular data like index nodes, or hash table buckets, the lock itself should be space efficient. This does not necessarily mean minimal, but it should also not waste unreasonable amount of space. For instance, a std::mutex (40-80 bytes) would almost double the size required for an ART node [17].

Lock 还需要能够高效的处理锁竞争：
> While we assume that heavy contention is usually rare in a well-designed DBMS, some workloads make it unavoidable. The lock should, thus, handle contention gracefully without sacrificing its fast, uncondented path.

同时最好也要能够周期性的检查是否有被 cancel：
> for example, that the user wants to cancel a long-running query, but the working thread is currently sleeping while waiting for a lock. Waiting too long can lead to an unpleasant user experience. For this reason, it would be desirable if the lock’s API would allow one to incorporate periodic cancellation checks while a thread is waiting.

## Locking Technioques

在数据库中，我们对 lock 的需求是既要最小化 overhead 又要最大化 scalability。最近的一些研究结果表明，除了在 write-heavy 的场景里 pessimistic locking 的优势会更大以外，其他场景中 Optimistic Locking 相比 pessimistic locking 或者 lock-free 都有更好的性能和其他优势。这里作者罗列了几个常见的 Optimistic locking 和它们的优缺点，详细介绍了目前在 Umbra 中使用的 Hybrid Lock 设计和实现细节。

### Optimistic Locking

![Listing 1: Optimistic Locking](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304052144085.png)

Optimistic Locking 的基本思路是检查在读数据的时候没有其他线程正在修改它。Optimistic locking 会维护一个版本号，每次修改数据都增加这个版本号。读的时候如果发现 encode 在版本号中的 lock 比特位为 1，或者读取数据后版本号发生了变化，这次读操作就需要重试。上图的伪代码展示了一个可能 fallback 到 pessimistic lock 的 optimistic lock 实现，推测 `isLocked(preVersion)` 就是在检查 lock 比特位。

Optimistic Locking 和 Pessimistic Lock 的性能差异原因主要出现在 cache line 上：

> Optimistic locking avoids atomic writes and its cache line stays in shared mode. Pessimistic locks must always modify the cache line and thus their performance is bound by cache-coherency latencies

Optimistic locking 特别适合用在经常读的热点数据上，比如 B Tree 的 root 节点。使用时需要注意几个问题：
1. Optimistic locking 在碰到写的时候会重试正在进行的读操作，需要确保重试时不会发生异常。比如读取某个 B Tree 节点，可能别的线程触发了 B Tree Merge 操作导致当前要读的节点被删除了，需要确保重试时访问这个节点不会出现访问 null pointer 的情况。LeanStore 和作者提到的 Adaptive Radix Tree 可以通过 Epoch 机制来确保不会发生这个问题
2. 注意上面伪代码传入的是一个 readCallBack，这个 callback 每次重试都会被调用，如果在里面更新一些值，比如求 count，那么可能就会因为重试得到错误结果。最安全的做法是仅通过这个 callback 获取值后就 buffer 住，等整个读操作返回后才用读到的值去做下一步的计算
3. 当很多线程并发写时，optimistic lock 很容易被饿死，所以就像伪代码描述的那样，在经历了最大重试次数后需要能够 fallback 到 pessimistic lock

### Speculative Locking (HTM)

Speculative locking 是 Intel 在硬件上支持的一种 optimistic locking。即使是多个写线程也可以同时持有这个 lock，只要它们没有出现写冲突。这个 lock 有很多限制：

> All conflicts are detected on L1-cache line granularity (usually 64 bytes) and the addresses of the joint read/write-set must fit into L1 cache. Additionally, the critical section should be short to avoid interrupts or context switches and must avoid certain system calls.

Speculative locking 这种基于硬件的 lock 最主要的缺点就是它和硬件绑定，不够通用。只有比较新的 Intel 和 ARM 处理器才支持类似的功能。考虑到上面提到的使用限制，以及一些处理器可能没有这个功能，使用者通常都需要实现一个 fallback 到传统 lock 的机制来应对这些问题。

### Hybrid Locking

![Table1:Qualitativeoverview](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304062216400.png)

重读少写的场景可以使用 optimistic locking 获取最好的性能，读写混合的场景就需要用到 pessimistic locking 了。从上面的表格来看，不管是 write-heavy 还是 write-only 场景，使用 shared lock 都是一个更好的 pessimistic locking 选择。

要在不同场景都获得最好的性能，就需要根据上下文对同一个数据上不同的锁。比如 B Tree，访问 read-contended 上层节点就可以使用 optimistic locking，而访问叶子结点去 scan 上面的数据时就最好使用 pessimistic locking 避免代价高昂的冲突重试。

![Figure 2: Hybrid-Lock](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304062235593.png)

为了达到这个目标，作者设计了一个 Hybrid-Lock。如上图所示，这个 Hybrid-Lock 内部同时包含 RWMutex 和 atomic\<uint64_t\> 分别用来做 Pessimistic Lock 和 Optimistic Lock。

理论上也可以把这个 RWMutex 和 version 通过一个 64 位整数来实现。把他们分开后，实现起来简单直接不容易出问题，为 Hybrid-Lock 实现各种 RWMutex 已有的接口也比较简单，比如下图 HybridLock 的 lockShared()、unlockShared()、lockExclusive()、unlockExclusive() 就直接直接使用了内部的 RWMutex()。

还是以 B Tree 为例考虑这么一种情况：一个读线程通过调用 tryReadOptimistically() 来访问某个中间节点，tryReadOptimistically() 刚开始执行时没有任何其他线程读写这个节点，该函数顺利执行到了 readCallback()，但在 readCallback() 执行中另一个线程通过 lockExcluseive() 对这个节点上锁，修改这个节点的内容，最后通过 unlockExclusive() 释放锁。如果 unlockExclusive() 先释放了 RWMutex 的 lock 而没有完成 version +1，那么读线程因为是先检查 lock 再检查 version 就可能读到错误的值。

需要特别注意 unlockExclusive() 里面两个操作的顺序。因为 tryReadOptimistically() 在执行完 readCallBack 后先检查 RWMutex 再检查 version，unlockExclusive() 里面就需要先 version +1 再释放 RWMutex，确保读操作一定能够检测到这个读写冲突并重试。在 Intel 平台上也可以使用 `CMPXCHG16B` 指令来同时更新 version 和释放锁。

readOptimisticIfPossible() 和一开始在 Optimistic Locking 中看到的伪代码工作机制稍微有点区别。它会在 tryReadOptimistically() 失败后直接从回退到 pessimistic locking 模式，使用 lockShared() 和 unLockShared() 完成这次读操作。

Hybrid-Lock 再加上后面提出的 ParkingLot 的 lock contention 处理策略就是目前 Umbra 中使用的锁实现，替代了之前提到的 Versioned Latch 方案。下面就是 Hybrid Lock 的伪代码实现。

![Listing 2: Hybrid Locking](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304052230691.png)

## Contention Handling

锁冲突是很难避免的，比如在 write-heavy 场景所有线程都使用 pessimistic locking 时，Hybrid Lock 中的 RWMutex 上就可能发生锁冲突。如何高效处理锁冲突，并且在线程等锁期间能够识别到 query 被 cancel 及时停止 query 执行呢？

这个章节作者分析了 Hybrid Lock 可能的 RWMutex 实现，最终采用了 Parking Lot 的方案。

### Busy-Waiting/Spinning

![Figure 3: False-sharing](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304061335157.png)

Spinning 是一种常见的处理方式，和 [Spinlock](https://en.wikipedia.org/wiki/Spinlock) 一样。Spinning 的一些缺点：
1. Spinning can lead to priority inversion, as spinning threads seem very busy to a scheduler they might receive higher priority than a thread that does useful work. Especially in the case of over-subscription, this can cause critical problems
2. Heavy spinning wastes resources and energy [6] and increases cache pollution, which is caused by additional bus traffic. 作者通过 cache line 的例子详细解释了这个问题：Following the MESI-protocol, every atomic write needs to invalidate all existing copies in other cores. Ideally, a core owns a cache line exclusively and does not need to send any invalidation messages. However, if other threads are spinning on the same lock, they constantly request this cache line, causing contention. The negative effects are worst when the waiting thread does write-for-ownership cycles, as those cause expensive invalida- tion messages. For this reason, a waiting thread should use the test-test-and-set pattern and only do the write-for-ownership cycle when it sees that the lock is available. In other words, it only reads the lock state in the busy loop to keep the lock’s cache line in shared mode.
3. 和 lock 处于同一 cache line 的数据也会受到影响，也就是 false-sharing 的问题。spinning can still lead to cache pollution when the protected data is on the same cache line as the lock itself (cf. Figure 3). By spinning on the lock the waiting thread 𝑇𝑤𝑎𝑖𝑡 constantly loads the cache line in shared mode. Whenever the lock owning 𝑇h𝑎𝑠𝐿𝑜𝑐𝑘 updates the protected data, it must invalidate 𝑇𝑤𝑎𝑖𝑡 ’s copy of the cache line. Having to send these invalidation messages, slows down 𝑇h𝑎𝑠𝐿𝑜𝑐𝑘 and increases the time spent in the critical section.

虽然现有方案通过 backoff 策略可以缓解 spinlock 的这些问题，但这些策略本身也不是很完美：
> there exist several backoff strate- gies that add pause instructions to put the CPU into a lower power state, or call sched_yield to encourage the scheduler to switch to another thread. However, since the scheduler cannot guess when the thread wants to proceed, yielding is generally not recommended as its behavior is largely unpredictable [31].

### Local Spinning using Queuing

![Figure 4: Queuing lock](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304061340149.png)

spinning 带来的 cache contention 问题在 NUMA 架构上会更严重。为了解决这个问题，一些 spinlock 的实现只 spin 这个 lock 的 thread-local 副本，比如 MCS-lock 或者 Krieger et al. 在 [13, 25] 提出的 read-write mutex。

这种 lock 的工作方式：
> When acquiring a lock, every thread creates a thread-local instance of the lock structure including its lock state and a next pointer to build a linked list of waiting threads.3 Then, it exchanges the next pointer of the global lock, making it point to its own local instance. If the previous next entry was nil, the lock acquisition was successful. Otherwise, if the entry already pointed to another instance, the thread enqueues itself in the wait- ing list by updating the next-pointer of the found instance (current tail) to itself.

### Ticket Spinlock

ticket spinlock 是另一种 spinlock，guarantees fairness without using queues。

它的工作方式：
> maintaining two counters: next-ticket and now-serving. A thread gets a ticket using an atomic fetch_and_add and waits until its ticket number matches that of now-serving.

除了能够保证 fairness 以外，它还有其他优点：
> this also enables more precise backoff in case of contention by estimating the wait time. The wait time can be estimated by multiplying the position in the queue and the expected time spent in the critical section. Mellor-Crummey and Scott argue that it is best to use the minimal possible time for the critical section, as overshooting in backoff will delay all other threads in line due to the FIFO nature


### Kernel-Supported ParkingLot

![Figure 5: Parking Lot](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304061355223.png)

上面提到的 spinlock 始终存在 over-subscription 或者 waste of energy 的问题。因此很多库的锁实现（比如 pthread mutex）都基于 Linux 内核提供的 kernel-level locking 来 suspend 当前这个线程直到拿到这个 lock 为止。

但内核上的系统调用开销是很高的，所以也有一些自适应的锁实现，仅锁冲突的时候才调用 kernel 阻塞当前线程，比如 Linux 提供的 futex。

基于 futex 的思路，WebKit 提出了一个叫 Parking Lot 的自适应锁。也是 Umbra 目前正在使用的锁实现。Parking Lot 用一个全局哈希表来存储 lock 到 wait queue 的映射关系。和 Linux futex 不一样，这种实现方式更加通用，可移植性强，不依赖其非标准的或者特定平台的系统调用。它也更加灵活，比如在 Parking 的时候可以执行某个 callback 函数。Umbra 利用这个特性在锁等待时检查查询是否被取消，Page 是否已经被缓存替换等。

上图描述了 Umbra 中实现的 Parking Lot 锁。当线程获取锁后会将 lock bit (L) 设置为 1。当另一个线程再次获取锁时，它会在 parking lot 中等待。此时它会把锁的 wait bit (W) 设置为 1 表示有人正在等锁，然后使用这个锁的地址在哈希表中找到该锁对应的 parking space。如果仍旧满足用户自定义的 wait condition，该线程开始等待这个 condition variable。当第 1 个线程释放锁后，它发现 wait bit (W) 为 1 知道有其他线程正在等锁，它会找到这个锁对应的 parking space，将所有等待的线程都唤醒。为了避免 parking space 的 data race 问题，每个 parking space 都有一个 mutex 来保护。

关于 lock bit 和 wait bit，作者在论文的 2.3 小结介绍完 Hybrid Lock 后有个补充说明，放到这里我们了解到 parking lot 后就比较容易理解了：
- wait bit：encode 在 Hybrid-Lock 的 RWMutex 上，用来表示有其他线程等锁
- lock bit：encode 在 Hybrid-Lock 的 version 上，检测到有锁后当前线程需要调用下面伪代码中的 `park()` 函数进入 parking 状态，直到被 condition variable 唤醒。

Parking lot 本质上就是个固定 512 槽位的哈希表，因为冲突的锁数量最多不会超过使用的线程数，所以 512 个槽位就足够用了。采用拉链法解决哈希冲突。

当执行用户 query 的线程在 parking space 中等待时，每 10ms 会被唤醒检查当前 query 是否被取消了，以便停止等待及时结束当前 query 的执行。

下面是 parking lot 的伪代码。虽然非常简单直接，但使用需要特别小心。需要确保造成线程等待的信息不会丢失导致线程无限期的等待下去，任何修改这些数据的线程都需要检查锁的 wait bit，在必要时唤醒所有等待的线程。作者这里没举例子，我能想到的一个场景是 B Tree Node 被缓存替换的场景。

![Listing 3: Parking Lot Implementation](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304061357401.png)

论文到这（第 5 页）核心内容就结束了，后面花了大量的篇幅介绍和展示作者的测试结果。

## Evaluation

### TPC-C and TPC-H

### Lock Granularity

### Space Consumption

### Efficiency of Lock Acquisition

### Contention Handling Strategies

## RELATED WORK

## CONCLUSION