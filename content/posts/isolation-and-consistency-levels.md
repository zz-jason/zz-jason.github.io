---
title: "Isolation Levels and Consistency Models in Distributed Transactions"
date: 2022-01-28T08:24:41Z
draft: true
---

![consistency models](/images/isolation-and-consistency-levels/consistency-models.png)

## Isolation Levels

## Consistency Models

Jepsen 在他的博客 《[Consistency Models](https://jepsen.io/consistency)》 中很好的讲述了什么是一致性模型，各个一致性模型分别可能出现什么异常现象，一个应用程序通常需要什么样的一致性模型。

### 为什么需要一致性模型？

为什么有了事务隔离级别后，我们还需要关心一致性级别呢？

确实单机事务只需要考虑事务隔离级别就行了。但是在分布式系统的分布式事务中，因为涉及到数据的多副本、多版本，以及最重要的，各个节点存在着时钟误差，导致即使是最高的 Serializabe 隔离级别也可能出现违背事实情况的异常现象，因此在分布式系统的分布式事务中我们不仅需要考虑隔离级别，还需要考虑一致性模型。CockroachDB 的博客《[Living Without Atomic Clocks | Where CockroachDB & Spanner Diverge](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/)》 中关于这一点是这么讲述的：

> In a non-distributed database, serializability implies linearizability for transactions because a single node has a monotonically increasing clock (or should, anyway!). If transaction T1 is committed before starting transaction T2, then transaction T2 can only commit at a later time.
>
> In a distributed database, things can get dicey. It’s easy to see how the ordering of causally-related transactions can be violated if nodes in the system have unsynchronized clocks.

![Causally Related Transactions](/images/isolation-and-consistency-levels/causally-related-transactions.png)

文中基于上图举了一个例子用来说明这个异常现象。假设：
1. 在 N1 和 N2 节点上各自提交事务 T1（把 x 的值写为 1）和 T2（把不同于 x 的 y 的值写为 2）
2. T1 和 T2 各自从 N1 和 N2 的本地时钟获取事务提交的时间戳
3. N1 的时间是准确的，N2 的时钟相比 N1 晚 100ms

在上帝视角的外界看来（这也是我们希望看到的完美结果），x 的更新早于 y：

1. T1 在 t=150ms 的时刻提交，把 x 更新成 1
2. T2 在 T1 提交结束后开始并在 t=200ms 的时刻提交把 y 的值更新成了 2。

但实际上在这个系统中 T2 的时钟比 T1 晚 100ms，给 y 分配的时间戳是 t=100ms，而给 x 分配的时间戳是 150ms，造成了 x 的更新晚于 y 的假象。这个系统之后如果读 x 和 y 的值可能得到这样的结果：
1. x=0, y=2，如果这个只读事务的时间戳在 t=100ms 和 t=150ms 之间的话
2. x=1, y=2，如果这个只读事务的时间戳在 t=150ms 之后的话

可以看到，上面这个例子完全满足事务的 Serializabe 隔离级别，但是还是出现了不符合直觉的异常现象（这个例子中展现的一致性模型满足 sequential 但是不满足 linearizable）。


《[Demystifying Database Systems, Part 4: Isolation levels vs. Consistency levels](https://fauna.com/blog/demystifying-database-systems-part-4-isolation-levels-vs-consistency-levels)》 把隔离级别和一致性模型讲的很透彻。关于这个问题，文章是这么回答的：

> the serializability guarantee allows transactions to “go back in time” --- as long as the final result is equivalent to some serial order.
>
> If you want guarantees for nonconcurrent transactions（注：在分布式系统中）, for example, you want to be sure that a later transaction reads the writes of an earlier transaction, you need consistency guarantees.

回头来看，隔离级别和一致性模型在分布式系统中不是

### 什么是一致性模型（Consistency Model）？

#### 稍微形式化的解释

在博客 《[Consistency Models](https://jepsen.io/consistency)》 中，作者定义了两个关键的概念来帮助我们理解什么是一致性模型，先来看一个关于 History 的概念：

> A history is a collection of operations, including their concurrent structure.

一致性模型可以通过 history 来描述，同事务隔离级别一样，一致性模型也用来描述允许出现哪些操作历史，不同的一致性级别所允许的异常现象不一样：

> A consistency model is a set of histories. We use consistency models to define which histories are “good”, or “legal” in a system. When we say a history “violates serializability” or “is not serializable”, we mean that the history is not in the set of serializable histories.

一致性模型也有强弱关系，例如 linearizability 就是比 sequential 更强更严格的一致性模型。

> We say that consistency model A implies model B if A is a subset of B. For example, linearizability implies sequential consistency because every history which is linearizable is also sequentially consistent. This allows us to relate consistency models in a hierarchy.


#### 稍微直白点的解释

在 《[Demystifying Database Systems, Part 3: Introduction to Consistency Levels](https://fauna.com/blog/demystifying-database-systems-introduction-to-consistency-levels)》  中明确的指出，Consistency Level 中的 Consistency 指的是 CAP 中的 C，不是 ACID 中的 C：

> When we talk about consistency levels, we’re really referring the **C** of **CAP**. In this context, perfect consistency --- usually referred to as “strict consistency” --- would imply that the system ensures that all reads reflect all previous writes --- no matter where those writes were performed. Any consistency level below “perfect” consistency enables situations to occur where a read does not return the most recent write of a data item. [Side note: the **C** of **CAP** in the original CAP paper refers to something called “atomic consistency” which is slightly weaker than strict consistency but still considered “perfect” for practical purposes. We’ll discuss the difference later in this post.]

另外，《[Demystifying Database Systems, Part 4: Isolation levels vs. Consistency levels](https://fauna.com/blog/demystifying-database-systems-part-4-isolation-levels-vs-consistency-levels)》 中给了一个更直白的解释，consistency 是从客户端的角度来观测和定义的：

> Database consistency is defined in different ways depending on the context, but when a modern system offers multiple consistency levels, they define consistency in terms of the client view of the database. If two clients can see different states at the same point in time, we say that their view of the database is inconsistent (or, more euphemistically, operating at a “reduced consistency level”).

### 有哪些一致性模型？
