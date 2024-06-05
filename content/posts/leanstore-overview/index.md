---
title: LeanStore | 架构概览，源码导读
date: 2024-05-09T00:00:00Z
categories:
  - Storage
draft: true
---
## Introduction

LeanStore 是一个面向 NVMe SSD 和多线程优化，性能接近内存数据库，数据不用全部缓存在内存中的 Larger-than-Memory 数据库。之前为了方便学习 fork 了一份 LeanStore 代码，在读代码和实验的过程中修复了发现的问题，增加了测试来保证正确性，也修改了代码风格使其更加好读，基本架构和原理还是和原来 LeanStore 一样。

这篇文章根据 [zz-jason/leanstore](https://github.com/zz-jason/leanstore) 最新代码介绍 LeanStore 的整体架构和关键模块的实现原理。许多内容在之前论文分享的文章中已经详细介绍过，但随着 LeanStore 的架构演进，部分模块的原理和实现和之前的文章会有些出入。最能反应代码实现的是《[The Evolution of LeanStore](https://dl.gi.de/server/api/core/bitstreams/edd344ab-d765-4454-9dbe-fcfa25c8059c/content)》这篇论文。

## 整体架构

![Architecture](architecture-v3.jpg)
> LeanStore Architecture

LeanStore 技术架构如上图所示。由工作线程 worker 执行用户读写请求，worker 包含了所有事务并发控制需要的数据，比如存储 MVCC 版本链的 history tree，用于可见性检查的 commit log，用于构造 wal 的 wal buffer 等。

Buffer Management。使用 B+ tree 组织和存储数据，每个 page 大小为 4KB，内存中的 page 由 buffer manager 集中管理，存储在 4KB+512B 的 buffer frame 中，通过 page evictor（也就是原 LeanStore 代码中的 page provider）进行缓存替换，为工作线程从磁盘读取 page 提供空闲 buffer frame。

Logging。工作线程会先将 wal 生成在本地的 wal buffer 中，由后台线程 group committer 不停的轮巡和收集所有工作线程的 wal buffer，使用 libaio 将 wal 写入磁盘文件，并根据事务的依赖关系判断各个 worker 上的事务是否可以 commit，通过 page 级的依赖检测实现了 remote flush avoidance（RFA）识别互相独立的事务以避免 commit 等待。

所有的数据对象和线程资源都通过 `LeanStore` 对象来管理，在 LeanStore 构造函数中初始化，在析构函数中释放所有资源。这部分代码在 src/LeanStore.cpp。

## Buffer Manager

![](buffer-manager.jpg)
> Buffer Management

Buffer Manager 由 buffer pool 和 page evictor 组成，代码路径如下：
- src/buffer-manager/BufferManager.hpp
- src/buffer-manager/BufferManager.cpp

### BufferFrame/Page

Buffer pool 是由 mmap 初始化的一段大小固定的连续内存，其的基本单元是 BufferFrame，每个 BufferFrame 大小为 4KB + 512 字节，其中的 4KB 用于存储从磁盘读取上来的 Page，512 字节用于存储 BufferFrameHeader。BufferFrameHeader 包含了 Page 的状态、锁、page id 等信息。BufferFrame、BufferFrameHeader、Page 都定义和实现在同一个代码文件中：
- src/buffer-manager/BufferFrame.hpp

### Swip

LeanStore 采用 Pointer Swizzling 存储 BufferFrame 的内存地址或 page id，Swip 的实现其实就是个 page id 和内存指针的 union，通过在最高个 bit 位上打 tag 的方式标志这个 union 的 int64 是个 page id 还是内存指针。Swip 的代码路径如下所示：
* src/buffer-manager/Swip.hpp

### Resolve Swip

LeanStore 通过 BufferManager 的 ResolveSwipMayJump() 接口将对应的 Swip 解析成 BufferFrame。

为了减少 page 读取过程中的锁竞争，LeanStore 将 page id hash 分片成了 64 个 partition，每个 partition 由 inflight IO hash table 和 free BufferFrame list 构成，buffer pool 初始化时所有的 BufferFrame 都均匀分配到了各个 partition 的 free list 中。工作线程在读取 page 时会先根据 page id 定位到对应的 partition，再获取该 partition 的 inflight IO hash table 以查询对应 page 的读取状态，所需要的 free BufferFrame 也直接从该 partition 的 free list 中获取。

Page evictor 是个不停工作的后台线程，通过缓存替换驱逐不再需要的 page 为 partition 补充 free list 保障前端工作线程有足够的 free BufferFrame 使用。Page eviction 采用 secondary chance 策略。先随机挑选一批 BufferFrame，然后根据 BufferFrame 中 page 的状态来判断是应该将其 evict 还是 cool 还是挑选它的一个 child 继续处理：
- 如果 page 处于 HOT 状态并且它在 B+ Tree 中的的所有孩子节点所处的 page 都已经 evict，那么就将该 page 状态修改为 COOL 并从当前 BufferFrame batch 中移除。COOL 状态的 page 仍然处于内存中，worker 读写到 COOL 状态的 page 后会将其状态修改为 HOT，否则它会一直维持在 COOL 状态。
- 如果 page 处于 COOL 状态，表明该 page 从之前某次 page eviction 被修改为 COOL 后一直没有任何工作线程读写过它，这一轮 page eviction 该 page 就可以直接被 evict 掉了。如果是 dirty page，则需要将脏页回写至底层的 page file。
- 如果 page 处于 HOT 状态且有某个孩子节点尚未 evict，并且其中某个节点为 HOT，则用 HOT 状态的孩子节点替换当前 BufferFrame，然后重复上面的过程

以 B+ Tree 的视角来看，page eviction 会从叶子结点层层向上 evict。Page eviction 的代码在 src/buffer-manager/PageEvictor.cpp 

## B+ Tree

Page 的实现在 src/buffer-manager/BufferFrame.hpp。page 中包含 GSN、btree id、和 payload，其中 payload 存储 BTreeNode。BTreeNode 的定义在 src/btree/core/BTreeNode.hpp 中。其内存布局如下所示：

LeanStore 的 B+ Tree 比较朴素，没有像 B-link Tree 那样存储 right sibling，也没有指向父亲节点的指针。存储了

## Synchronization Primitives

这里主要介绍内部广泛使用的 HybridLatch 和 JUMPMU。

### HybridLatch

HybridLatch 是结合 std::atomic 和 std::shared_latch 实现的可以提供三种锁模式的混合锁，这三种模式分别为：
- optimistic shared：乐观共享锁，用于读操作，读之前上锁，记录 atomic 的值，判断是否有人以 pes simistic exclusive 的方式持有锁，若无则上锁成功。读完后释放锁，检查 atomic 变量是否发生变化，检测到变化说明这段期间被锁保护的数据发生了修改，之前的读操作需要重试。
- pessimistic shared：悲观共享锁，用于读取可能经常被修改的数据，本质上就是对 shared_mutex 加了 shared lock。这类数据如果使用 optimistic shared 的方式上锁和读取会因为比较高频的并发写导致不断重试。不如直接使用 pessimistic shared 的方式先上锁，引入锁竞争，阻止其他人在读取期间修改，避免 optimistic shared 无效重试带来的额外 CPU 开销。
- pessimistic exclusive：悲观排它锁，用于写操作，也就是对 shared_mutex 加了 unique lock，在上锁时增加 atomic 的值，释放锁时再次增加 atomic 的值，使 optimistic shared 模式下上锁的并发线程能够检测到数据更新。

HybridLatch 的实现在 src/sync/HybridLatch.hpp 中：
```cpp
class alignas(64) HybridLatch {
private:
  std::atomic<uint64_t> mVersion = 0;

  std::shared_mutex mMutex;
```

在获取或释放悲观排它锁时都需要增加 mVersion，使得乐观共享锁能够感知到并发数据更新：
```cpp
  void LockExclusively() {
    mMutex.lock();
    mVersion.fetch_add(kLatchExclusiveBit, std::memory_order_release);
    LS_DCHECK(IsLockedExclusively());
  }

  void UnlockExclusively() {
    LS_DCHECK(IsLockedExclusively());
    mVersion.fetch_add(kLatchExclusiveBit, std::memory_order_release);
    mMutex.unlock();
  }
```

LeanStore 中 B+ Tree 遍历操作会先对中间节点上乐观共享锁，对叶子结点根据需要上乐观或悲观锁。这里对工作负载的假设是读多写少，B+ Tree 上很少发生 node split 和 merge，因此中间节点直接上乐观共享锁。而所有的读写操作都在叶子结点，如果要写则直接上悲观排它锁，如果要读可以先上乐观共享锁试试，如果检测到读写冲突，则重试时上悲观共享锁。这部分代码可以参考 TransactionKV 的 Lookup 接口实现。（注：TransactionKV 也就是原 LeanStore 代码中的 BTreeVI，实现了 MVCC 和 SI 隔离级别的并发控制）

关于 HybridLatch 可以参考这篇文章：[[DaMoN 2020] Scalable and Robust Latches for Database Systems](https://zhuanlan.zhihu.com/p/623976822)
### JUMPMU

前面提到在乐观共享锁模式下检测到冲突时需要重试整个读操作。从性能的角度来说，重试的 CPU 开销需要越低越好，从代码编写的角度来说，重试需要能够跨函数调用，还需把沿途在栈上分配的对象和他们管理的资源都释放掉，尤其是锁。比如 B+ Tree 遍历，可能遍历到最后一层中间节点时才发现冲突，这时候就需要从根节点开始重新遍历，之前持有的中间节点和其他数据的锁都需要全部释放掉。

目前来看只有 longjump 能够非常高效的实现跨函数调用的跳转和重试（goto 只能单文件内跳转，C++ exception 处理的 CPU 开销又很高），但是 longjump 仅释放栈上内存，不调用相关对象的析构函数，所以 RAII 编程范式在这里就失去了效果，资源也就没法在对象析构时有效的释放。为了解决这个问题。LeanStore 使用了一个全局数组来保存这些对象，并在需要重试进行 jump 之前将上次 setjump 以来所有收集到的需要释放资源的对象都去析构一遍，完成 jump 之前的资源释放。为了使栈上生成的对象都被自动加入到该数组中，只需在这些对象的构造函数中将该他们放入数组中即可。

最终，这套 JUMPMU 原语相当于使用 longjump 实现了个轻量级的异常捕获机制，编码使用它时也可以这么去类比。

JUMPMU 的代码实现在 xxx，使用方法如下：

## Logging

![](logging.jpg)
LeanStore 通过后台线程 Group Committer 将 WAL 写入磁盘。Worker 先将 WAL 写入本地的 WAL Buffer，Group Committer 不停收集所有 Worker 的 WAL Buffer，收集新增的 WAL Record，最后通过 libaio 将所有收集到的 WAL Record 一次性落盘。

LeanStore 通过类似 Lamport 时钟的 GSN 机制追踪并发事务之间的依赖关系。具体来说每个 Page 和 Worker 都各自维护了一个 GSN，每当 Worker 遍历 B+ Tree 读取 Page 时都会使用各个 Page 上的 GSN 来更新自己的 GSN，在 Worker 更新 Page 时会同时更新 Page 和 Worker 的 GSN。

事务 T1 执行过程中如果发现某个 Page 的 GSN 比当前 Worker 的 GSN 更大，并且该 Page 的上次写入者不是当前 Worker，则该事务 T1 依赖某个并发执行的事务 T，事务 T1 能提交的前提是所有依赖的并发事务的 wal 已经落盘。这个检测是在 Group Committer 中做的，Worker 在更新自己的 WAL Buffer 时也会将最新 WAL Record 对应的 GSN 记录下来。Group Committer 收集到这些 WAL Records 和 GSN，将所有 WAL Record 写入的同时也计算 min flushed GSN 作为全局 GSN 水位线，所有 GSN 小于该水位线的事务都可以安全提交。

如果事务执行过程中发现没有任何远端依赖，则它只需要等自己的 wal records 都被写入后即可提交。这个就是 LeanStore 的 Remote Flush Avoident 机制。在事务提交时，根据事务是否有远端依赖将其放入有远端依赖的 commit queue 或无远端依赖的 commit queue，针对不同的 commit queue 采用不同的方式判断是否能够 commit。

## MVCC

MVCC 的实现原理可以参考这篇文章：[[VLDB 2023] Scalable and Robust Snapshot Isolation for High-Performance Storage Engines](https://zhuanlan.zhihu.com/p/631694262)

在没有接触代码之前其实觉得上面这篇论文中描述的事务机制蛮复杂的，不过结合代码来看会比较好理解。

原来 LeanStore 代码中总共有两个 B+ Tree 实现，分别是 BTreeLL 和 BTreeVI。其实 BTreeLL 不带事务，它的所有修改操作比如 update/remove 等都不会做并发控制，仅对当前叶子结点上锁后直接修改对应 key 的值。而 BTreeVI 则实现了完整的 MVCC 事务。为了方便区分和理解，我把 BTreeLL 重命名成了 BasicKV，把 BTreeVI 重命名成了 TransactionKV。

为了避免 long-running transaction 导致的版本累积，每个 TransactionKV 都有个由 BasicKV 实现的 graveyard，用于存放 short-running transaction 已经删除但是 long-running transaction 还在使用的数据。这部分可以看 src/btree/TransactionKV.cpp 中的 `TransactionKV::Remove()` 部分。

同理 long-running transaction 在 Lookup 的时候，如果没有在当前 TransactionKV 中找到，则需要去它的 graveyard 中再找一遍。这部分可以参考 `TransactionKV::Lookup()` 部分。

TransactionKV 采用了 chained tuple 来存储某个 key 的所有版本链。在 BTree 的叶子结点中只存储最新版本的数据，在各个工作线程内使用 BasicKV 存储了所有老版本，其中的 key 为 (newer worker id, newer transaction id, newer command id)，value 为 具体的 delta。

LeanStore MVCC 的特点一个是 precise GC，一个是避免 long-running transaction 和 short-running transaction 互相影响。

## Worker

