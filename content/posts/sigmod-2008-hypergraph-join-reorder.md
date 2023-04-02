---
title: "[SIGMOD 2008] Dynamic Programming Strikes Back"
date: 2023-03-01T08:00:00Z
categories: ["Paper Reading"]
draft: true
---

MySQL 8.0.31 引入了 Hypergraph Join Optimizer，它用超图表示多表 Join，每个边连接任意个表，直接表示复杂的 Join Predicate，避免了无用的 Cross Join。它采用了《Dynamic Programming Strikes Back》中的 DPhyp 方法进行 Join Reorder，能够更好地优化多表 Join 的性能和成本。

我对这个新特性非常感兴趣，就读了《Dynamic Programming Strikes Back》这篇论文，并整理了一些 Hypergraph Join Order 算法的思路和效果，希望对读者朋友们有帮助。


## 为什么要引入 Hypergraph Join Order 算法

DPsize -> DPccp
memoization -> top-down partition search

DPccp 和 top-down partition search 没被大规模生产使用的原因：

> First, as has been argued in several places, hypergraphs must be handled by any plan generator [1, 19, 23]. Second, plan generators have to deal with outer joins and antijoins [11, 19].


## 什么是 Hypergraph

Hypergraph 由 Hypernode 和 Hyperedge 构成：
1.  Hypernode：它是一个节点集合，里面可以有一个或多个节点。例如，下图中 {R1} 是只有 R1 的 Hypernode，{R1, R2, R3} 是有 R1、R2 和 R3 的 Hypernode。
2.  Hyperedge：它是一条连接两个 Hypernode 的边。例如，下图中 {R1} 和 {R2} 之间的边是一个 Hyperedge，{R1, R2, R3} 和 {R4, R5, R6} 之间的边也是一个 Hyperedge。

在 Join Order 的上下文里，Node 指的是参与 Join 的表，Edge 指的是 Join Predicate。例如，下图中，R1 ~ R6 是 6 个表，`R1.a + R2.b + R3.c = R4.d + R5.e + R6.f` 是一个复杂的 Join Predicate，它连接了两个 Hypernode：{R1, R2, R3} 和 {R4, R5, R6}。

![Sample Hypergraph](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303040026259.png)

把所有要 Join 的表看作节点，把他们之间的 Join Predicate 看作边。那么，Join Order 问题就变成了：怎样选择节点和边，构建一个包含所有表和条件的图，使得图的总权值最小。

为了方便后续介绍 Hypergraph Join Order 算法的过程，我们还需要继续学习几个概念：
1. 子图（sg）：子图（sg）是由某些 Hypernode 和它们之间的 Hyperedge 构成的图，例如上图中 {{R1}, {R2}} 是一个 Hypernode 子集，它们只有一个 Hyperedge ({R1}, {R2})，形成了一个含 2 个 Hypernode 的子图。
3. 联通子图（csg）：联通子图是指可以划分成两个联通子图通过 Hyperedge 相连的子图，与普通连通图类似，只是把 Node 和 Edge 换成 Hypernode 和 Hyperedge。例如上图中 {{R4}, {R6}} 就不是一个联通子图，因为 {R4} 和 {R6} 之间没有 Hyperedge。
4. 联通补图（cmp）：联通补图是指该子图的一个补图，并且这个补图是联通的。以上图 {{R1}, {R2}} 代表的子图为例，{{R4}, {R5}} 代表的子图是它的一个联通补图，但 {{R4}, {R6}} 代表的子图虽然是它的补图，但因为 {{R4}, {R6}} 不联通，所以不是它的联通补图。

联通子图和它的补图们（csg-cmp-pair）：csg-cmp-pair 是一个由 csg 和 cmp 组成的联通子图对，它们之间有一条 Hyperedge 连接，表示可以把两个子查询合并成一个更大的查询。例如，如果我们有四个表 A、B、C、D 和三个条件 A.id = B.id, B.name = C.name, C.age = D.age，那么我们可以把这个查询用一个超图表示：

其中每个节点是一个表，每条边是一个条件。我们可以找到所有可能的 csg-cmp-pair，比如：

-   ({A,B}, {C,D}) 是一个 csg-cmp-pair，因为 {A,B} 和 {C,D} 都是连接的子图，并且它们之间有一条边 B.name = C.name 连接。
-   ({B,C}, {A,D}) 是另一个 csg-cmp-pair，因为 {B,C} 和 {A,D} 都是连接的子图，并且它们之间有两条边 A.id = B.id 和 C.age = D.age 连接。

通过找到所有可能的 csg-cmp-pair，我们可以用动态规划的方法来寻找最优的 Join Order。


### 那怎么枚举所有的 csg-cmp pair 呢

论文中使用 E↓ 表示超图中非包含的超边的集合。如果一个超边 A 的所有节点都包含在另一个超边 B 的节点中，那么我们就说 A 被 B 包含或者 B 包含 A。例如下图中有三个超边 ({R1}, {R2}), ({R1}, {R2, R3}), ({R1, R2}, {R3, R4})，那么：
1.  ({R1}, {R2}) 被 ({R1}, {R2, R3}) 包含；
2. ({R1}, {R2, R3}) 是一个非包含的超边，因为它没有被其他任何超边包含；
3.  ({R1, R2}, {R3, R4}) 也是一个非包含的超边。

![](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041353643.png)

Hypergraph 的算法过程其实很简单：不断扩展子图，更新更大子图的最佳 Join Order。当计算完所有子图时，整个图的最佳 Join Order 也就确定了。在这个过程中需要不断枚举子图和它的补图。

动态规划的核心是最优子结构，在 Hypergraph 中子结构就是 Hypergraph 的子图，在 DPhyp 这个算法中仅考虑 connected subgraph（缩写成 csg）：

> Definition 2 (subgraph). Let H = (V, E) be a hy- pergraph and V′ ⊆ V a subset of nodes. The node in- duced subgraph G|V′ of G is defined as G|V′ = (V′,E′) with E′ = {(u,v)|(u,v) ∈ E,u ⊆ V′,v ⊆ V′}. The node ordering on V ′ is the restriction of the node ordering of V .

> Definition 3 (connected). Let H = (V,E) be a hy- pergraph. H is connected if |V | = 1 or if there exists a par- titioning V ′, V ′′ of V and a hyperedge (u, v) ∈ E such that u⊆V′,v⊆V′′,andbothG|V′ andG|V′′ areconnected.

从图论的视角来看，一个 connected hypergraph 也可以成为一个团，因为每个边代表了一个 join predicate，那么也就意味着这个 connected hypergraph 所对应的 join plan 中可以不使用 cross join（总是存在某个 join predicate 连接两个表集）

## DPhyp

### 基本思路

主要的挑战：
1. 如何高效的遍历 Hyperedge，使得我们可以遍历出所有相邻的 Hypernode。
2. Hyperedge 指向的 Hypernode 可能包含多个节点，使得递归遍历和增长 Hypergraph 子图变的非常复杂。

为了方便的保存算法执行过程的上下文信息，论文将 Hypergraph Join Order 算法需要使用的中间数据保存在 DPhyp 这个 class 中，将算法所涉及到的 5 个子过程实现为 DPhyp 的成员函数。

算法的入口是 Solve() 函数，用来初始化动态规划的 DP 表，以及所有子状态（也就是单表的最佳 Join Order）的初始值。接着调用 EmitCsg() 和 EnumerateCsgRec()。

EnumerateCsgRec() 负责递归的枚举所有联通子图。

### Subroutine 1: Solve()

![Pseudo code for solve](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041417706.png)

1. The algorithm calls EmitCsg({v}) for single nodes v ∈ V to generate all csg-cmp-pairs ({v}, S2) via calls to EnumerateCsgCmp and EmitCsgCmp.
2. The calls to EnumerateCsgRec extend the initial set {v} to larger sets S1.

### Subroutine 2: EnumerateCsgRec()

![EnumerateCsgRec](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041423506.png)

1. For each of these subsets N, it checks whether S1 ∪ N is a connected component. This is done by a lookup into the dpTable. If this test succeeds, a new connected com- ponent has been found and is further processed by a call EmitCsg(S1 ∪ N ).
2. Then, in a second step, for all these sub- sets N of the neighborhood, we call EnumerateCsgRec such that S1 ∪ N can be further extended recursively.

![EnumerateCsgRec example](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041531471.jpg)

Take a look at step 12. This call was generated by Solve on S1 = {R2 }. The neighborhood consists only of {R3 }, since R1 is in X (R4,R5,R6 are not in X either, but not reachable). EnumerateCsgRec first calls EmitCsg, which will create the joinable complement (step 13). It then tests {R2 , R3 } for connectedness. The according dpTable entry was generated in step 13. Hence, this test succeeds, and {R2,R3} is further processed by a recursive call to Enumer- ateCsgRec (step 14). Now the expansion stops, since the neighborhood of {R2, R3} is empty, because R1 ∈ X.

### Subroutine 3: EmitCsg()

![EmitCsg](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041424606.png)

It is then responsible to generate the seeds for all S2 such that (S1 , S2 ) becomes a csg-cmp-pair

1. B_min(S1): All nodes that have ordered before the smallest element in S1 (captured by the set B_min(S1)) are removed from the neighborhood to avoid duplicate enumerations.
2. Since the neighborhood also contains min(v) for hyperedges (u, v) with |v| > 1, it is not guaranteed that S1 is connected to v.

![EmitCsg example](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041542844.jpg)

Take a look at step 20. The current set S1 is S1 = {R1, R2, R3}, and the neighborhood is N = {R4}. As there is no hyperedge connecting these two sets, there is no call to EmitCsgCmp. However, the set {R4} can be extended to a valid complement, namely {R4, R5, R6}. Properly extending the seeds of complements is the task of the call to EnumerateCmpRec in step 21.

### Subroutine 4: EnumerateCmpRec()

![EnumerateCmpRec](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041424141.png)

EnumerateCsgRec has three parameters. The second parameter is a set S2 which is connected and must be extended until a valid csg-cmp-pair is reached.

![EnumerateCmpRec example](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041554851.jpg)

Take a look at step 21 again. The parameters are S1 = {R1, R2, R3} and S2 = {R4}. The neighborhood consists of the single relation R5. The set {R4,R5} induces a connected subgraph. It was inserted into dpTable in step 6. However, there is no hyperedge connecting it to S1. Hence, there is no call to EmitCsgCmp. Next is the recursive call in step 22 with S2 changed to {R4,R5}. Its neighborhood is {R6}. The set {R4,R5,R6} induces a connected subgraph. The according test via a lookup into dpTable succeeds, since the according entry was generated in step 7. The second part of the test also succeeds, as our only true hyperedge connects this set with S1. Hence, the call to EmitCsgCmp in step 23 takes place and generates the plans containing all relations.

### Subroutine 5: EmitCsgCmp()

![EmitCsgCmp](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041425353.png)

The task of EmitCsgCmp(S1,S2) is to join the optimal plans for S1 and S2, which must form a csg-cmp-pair. For this purpose, we must be able to calculate the proper join predicate and costs of the resulting joins.

## Evaluation

目前没有太多关于 Hypergraph 的 Join Order Benchmark，所以论文自己尝试构造了几个 Benchmark：

> The general design princi- ple of our hypergraphs used in the experiments is that we start with a simple graph and add one big hyperedge to it. Then, we successively split the hyperedge into two smaller ones until we reach simple edges. As starting points, we use those graphs that have proven useful for the join ordering of simple graphs.

论文着重考虑 Cycle 和 Star Query，没有考虑 Chain 和 Clique Query：

> The behavior of join ordering algorithms on chains and cycles does not differ much: the impact of one additional edge is minor. Hence, we decided to use cycles as one starting point.
> 
> Star queries have also been proven to be very useful to illustrate different performance behaviors of join ordering algorithms. Moreover, star queries are common in data warehousing and thus deserve special attention. Hence, we also used star queries as a starting point.
>
> The last potential candidate are clique queries. However, adding hyperedges to a clique query does not make much sense, as every subset of relations already induces a connected subgraph.

论文给了 Split Hyperedge 得到更多 Hypergraph 的例子：

![Cycle and Star with initial hyperedge](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041632733.png)

> Fig. 4a shows a starting cycle-based query. It con- tains eight relations R0 , . . . , R7 . The simple edges are ({Ri},{Ri+1}) for 0 ≤ i ≤ 7 (with R7+1 = R0). We then added the hyperedge ({R0,...,R3}, {R4,...,R7}). Each of its hypernodes consists of exactly half of the re- lations. From this graph (call it G0), we derive hypergraphs G1 , . . . , G3 by successively splitting the hyperedge. This is done by splitting each hypernode into two hypernodes comprising half of the relations. That is, apart from the simple edges, G1 has the hyperedges ({R0 , R1 }, {R6 , R7 }) and ({R2, R3}, {R4, R5}). To derive G2, we split the first hyperedge into ({R0 }, {R6 }) and ({R1 }, {R7 }). G3 addi- tionally splits the second hyperedge into ({R2},{R4}) and ({R3 }, {R5 }).

比较可惜的是，这篇论文仅关注了 DPhyp 生成 Join Order 的优化时间，没有关注生成的 Join Order 实际运行时间。在 Selectivity 估算、Cost Model 都理想的情况下，DPhyp 确实能产生不错的执行计划。不过考虑到 Selectivity 估算误差，所做出来的 Join Order 的实际运行时间是否也是最优的可能还需要更多的实验来验证。不过这个就属于 Join Order 问题的其他子问题了：
1. 如何准确、高效的估算 Join Cardinality
2. 如何建立 Cost Model 以考虑不同的 Join 算子物理实现之间的代价

### Cycle-Based Hypergraphs

![Results for cycle-based hypergraphs](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041619801.png)

### Star-Based Hypergraphs

![Results for star-based hypergraphs](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041621111.png)

### Queries with Regular Graphs

![Results for star-based regular graphs](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303041622629.png)

## Non-ReOrderable Operators

### Considered Binary Operators

> Besides the fully reorderable join (B), we also consider the following operators with limited reorderability capabilities: full outer join (M), left outer join (P), left antijoin (I), left semijoin (G), and left nestjoin (T). Except for the nestjoin, these are standard operators.

nestjoin 有一种变种称之为 d-join，d-join 可以和看做是 SQL Server 中的 Apply 算子，或者 Hyper 中的 DependentJoin 算子。在子查询优化中通常用来表示关联子查询，对子查询的解关联优化非常有帮助。

### Reorderability
