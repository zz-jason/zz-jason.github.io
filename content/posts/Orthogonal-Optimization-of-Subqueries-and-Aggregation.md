---
title: "[SIGMOD 2001] Orthogonal Optimization of Subqueries and Aggregation"
date: 2022-11-25T13:47:11Z
categories: ["Paper Reading"]
draft: false
---

SQL 优化中，索引选择、Join Reorder、子查询优化是最难处理的几个问题。这是一篇 2001 年（距今 21 年了，时间过得好快）发表的经典论文，提出了 Apply 算子和一种渐进式、迭代式的子查询去关联和优化的方法。不少数据库的子查询处理都采用了类似的思路，我觉得这篇论文最有价值的地方在于对问题的分治，体现在这些方面：

1. 通过引入 Apply 算子将子查询问题独立出来，通过一系列原子的优化规则，将子查询去关联的问题划分成一个个 Apply 消除和下推的小问题，在代码实现上结合 Volcano/Cascades 优化框架可以很好的控制代码复杂度，提高代码的可理解性和可维护性。论文最后还提到一种 Segmented Apply 算子，能够消除某种模式的重复 Table Scan 来提供更好的执行性能。
2. Apply 下推和消除的过程中其他 SQL 算子也会遇到新的优化机会，比如论文提到的聚合下推、上拉、Outer Join 化简成 Inner Join 等，这些复杂问题也可以通过分治的思想由一系列小的优化规则逐个解决。

近期因为工作原因重温了这篇论文，这次我把其中一些比较关键的原理和思考整理出来，希望能帮助到大家。

## 子查询和常用执行策略





## Apply 算子的定义

文章在 1.3 小结给出了 Apply 算子的形式化定义如下，Apply 的语义启发自 LISP，简单来说可以看做是一个 nested loop join：外层驱动表先读取一行数据，然后执行内部的 subquery。和普通 Join 算子一样，Apply 连接左右两表的方式可以是 cross join、left outer join、left semi join、 left anti join 等，只要能产生满足 SQL 语义的正确结果就行。

![Apply](/images/orthogonal-optimization-of-subqueries-and-aggregation/apply.png)

下面的例子讲述了 SQL 子查询和 Apply 的转换关系：

![Q1](/images/orthogonal-optimization-of-subqueries-and-aggregation/q1.png)



它转换出来的 Apply 算子如下：

![Subquery Execution Using Apply](/images/orthogonal-optimization-of-subqueries-and-aggregation/subquery-execution-using-apply.png)

如果 Apply 里面没有关联子查询，它其实等价于 nested loop join。Apply 算子的执行过程大致如下：

1. 读取一行驱动表的数据，上面例子是 CUSTOMER 表。

1. 根据该行驱动表的数据更新 Apply 算子所在子树的关联变量，上面例子中，假设读取上来的一行数据中 C_CUSTKEY 得值是 123，那么 O_CUSTKEY=C_CUSTKEY 绑定上这个运行时读上来的值后就变成了 O_CUSTKEY=123 这样的 filter。

1. 执行 Apply 算子得到结果，返回给更上层的 SQL 算子。在上面的例子中，O_CUSTKEY=C_CUSTKEY 的 C_CUSTKEY 被更新后，Apply 算子为根的子 plan 就可以独立执行返回结果了。经过 SGb（scala group by，没有 group by key 仅返回一行结果的 group by 算子）得到一行数据，和外层 CUSTOMER 的这一行数据 join 后也仅产生一行数据，然后经过后续的 1000000<x 的过滤条件，完成后续计算。

了解了 Apply 算子的定义，那如何通过 Apply 算子进行子查询的优化和执行呢。要完成子查询优化，我们需要先将 AST 转换成 Apply 算子。Appy 算子本身是可以执行的，但通常如果内表没有索引，或者外表的数据量非常大，Apply 的执行效率会非常低，所以还需要进行 Apply 相关的优化，尽量将 Apply 转换成没有关联子查询的普通 Join，然后利用现有的 Join 优化规则进行更进一步的查询优化。

## 将 AST 改写成包含 Apply 的执行计划

![AST](/images/orthogonal-optimization-of-subqueries-and-aggregation/ast.png)

子查询会出现在 SQL 的各个地方，但通常都是发生在表达式计算的时候，比如上面例子，在构造 WHERE 子句的 SELECT 算子时发现一个 scalar expression 内嵌了一个 subquery，这个 subquery expression 里的 statement 描述了这个 subquery 代表的 scalar value 是如何计算出来的。

Apply 改写的基本思路比较简单，只需要在使用这个 Subquery 的 SQL 算子之前构造好为其提供 Subquery 结果的 Apply 算子即可。在上面这个例子中，我们需要在 SELECT 算子之前构造好 Apply，执行计划变成：CUSTOMER -> Apply -> Select，其中 CUSTOMER 作为 Apply 算子的左子树，而 Subuquery 里的语句构造出来的执行计划作为 Apply 的右子树，原先在 SELECT 中的子查询也根据需要替换成其他等价的 scalar expression，比如当 Apply 是 left outer join 时，IN Subquery 就可以替换成对右孩子（代表子查询执行结果）某个字段的 IS NOT NULL 函数。

考虑到一方面子查询可能嵌套，另一方面一个 scalar expression 也可能包含多个子查询，而且我们在改写完子查询后还需要替换成新的 scalar expression，因此最好采用 Bottom Up 的方式递归的构造子查询对应的 Apply 算子和新的用于替代当前它的新的 scalar expression：

1. Bottom-Up 的遍历 expression 树，如果发现 child expression 已经被改写了，就更新其 child 信息

1. 遍历自己，如果需要进行子查询改写，则根据子查询的类型生成相应的 Apply 算子和改写后的新 expression。构造 Apply 时将当前 subquery expression 所在 SQL 算子的 child operator 作为新 Apply 算子的左孩子，子查询对应的 plan 作为新 Apply 算子的右孩子，根据 scalar expression 的语义选择适当的 join 类型（left outer join、cross join 等）。新的 scalar expression 使用构造出来的 Apply 算子所产生的结果做为输入。


通过这样 Bottom-Up 的遍历后，我们就得到了最终的 Apply 算子和需要替换的 scalar expression，然后继续遍历 AST 的其他部分，构造剩余的 query plan，完成从 AST 到执行计划的转化。

因为整个过程是递归的，每个子查询都会遍历到并转换成相应的 Apply 算子和 scalar expression，当这种方法应用到包含多个子查询的场景时，最终会生成一棵类似下面这样的 Apply 左深树：

![Left Deep Tree](/images/orthogonal-optimization-of-subqueries-and-aggregation/left-deep-tree.png)

### IN、NOT IN 的改写

IN 可以改写成 Distinct + Apply：先通过 Apply 计算出 Left Outer Join 的结果，然后将 Selection 中的子查询改为 Apply 产生的右表上的 column 的 IsNotNull 表达式即可。如果是 NOT IN 就改写成 IsNull。

比如 where t.id in (select id from s) 就改写成了：

![In Sub-Query](/images/orthogonal-optimization-of-subqueries-and-aggregation/in-subq.png)

对子查询的结果加 Distinct 去重是为了确保驱动表的数据经过 left outer join 后不会发生膨胀，如果能 Join 上则只会有一条数据产生，如果没有 Join 上则会把右表部分填补成 NULL 后也产生一条结果。因此 Apply 的输出结果里有完整的左表数据，我们也能根据输出结果中右表部分的数据是否为 NULL 来判断是否 Join 上（也就是 IN 子查询的结果应该是 true 还是 false）。

### EXISTS、NOT EXISTS 的改写

和上面 IN、NOT IN 大致相同。

### compare predicate 的改写

保留原本的 compare operator，把相关的 parameter 用子查询的结果替换。子查询根据需要改写成 Aggregate 或者 MaxOneRow（新算子，用于确保用户没有写 group by 但是又要求结果只产生一行数据的情况）

## Apply 的下推和消除

下面是论文给出的所有 Apply 消除会用到的原子优化规则。SQL Server 采用了 Volcano/Cascades 优化框架，每个优化规则只需要匹配一个 Plan 片段即可，因此 Apply 下推直至消除所需要使用的优化规则都可以一遍遍的应用到新产生的执行计划中，直至优化结束：

![Rules](/images/orthogonal-optimization-of-subqueries-and-aggregation/rules.png)

这些公式看起来复杂，但理解起来还是比较容易的，可以把他们分为 2 类：一类是直接消除 Apply 算子的，可以认为是 Apply 消除算法的终止条件，另一类就是不断进行 Apply 下推寻找 Apply 消除机会的，我们分别来介绍他们。

### 规则 1、2：Apply 消除

规则 1：当 Apply 的左表和右表没有任何 join 条件时，不管这是个什么 join（cross join、left outer join, left semi join、left anti join），都可以将 Apply 转换成普通 Join 算子。

规则 2：对 Apply 而言，右孩子的 Filter 可以等价转换成 Apply 的 join 条件，完成这次转换后，Apply 就和普通 nested loop join 没有什么区别了，可以安全的将其转换成普通 Join 算子。

需要注意的是：

1. 规则 1 和 2 能够生效的公共条件是 Apply 的右孩子中，E 为根的执行计划中不包含 R 中的相关列。
2. 理论上来说 Apply 转换后的执行计划不一定什么情况下都更好，比如当左表的数据量很小，右表在相关联列上有索引，或者右表已经提前物化到了计算节点上时，可能 Apply 算子这种 nested loop 的执行方式会更好。这个之前在 TiDB 的一些客户场景中遇见过一些。保险的做法是把 Apply 算子代表的老执行计划也留下来，通过 CBO 来选择。当然怎么样让 CBO 选择更准确就是另一个难题了。

### 规则 3 - 9：Apply 下推，寻找消除的机会

规则 1 和 2 描述了 Apply 消除的终止条件，而规则 3-9 则描述了在各个场景下 Apply 如何下推，行成可以触发规则 1 和 2 的执行计划。

1. 规则 3 和 4 分别描述了 Apply 下推过 selection 和 pojection 的场景
2. 规则 5 和和 6 分别描述了 Apply 下推过 union [all] 和 intersect [all] 的场景
3. 规则 7 描述了 Apply 下推过 cross join 的场景，需要注意的是：
   1. 下推后 R 表在左右两个 Apply 算子中分别被读了 1 次，如果底层没有采用 DAG 的 execution model 使得 R 表可以只读一次被多个 parent 复用可能会有很大的性能损耗，使得下推后的 Apply 不如下推前执行效率好。
   2. 下推后取代原始 Apply 的是一个 inner join，为了确保 R 表的每行数据仍然只会产生一行结果，这个 inner join 的 equal condition 需要在 R 表的某个唯一索引、主键或者其他可以唯一决定一行数据的字段上（比如 row id）。
4. 规则 8-9 分别描述了 Apply 下推过 vector aggregation 和 scalar aggregation 的场景。
   1. 因为下推过 Aggregation 后要确保 R 的每行数据只产生一行结果，我们需要保证新产生的 Aggregation 的 group by key 也是某个能唯一确定 R 表一行数据的主键、唯一索引、或者类似 row id 一样的字段。
   2. 另外对于规则 9 来说，先后的 aggregation 分别是 scalar aggregation 和 vector aggregation，需要特别注意某些聚合函数对 empty set input 和 null input 的结果差异。

### Apply 下推和消除的例子

![Example](/images/orthogonal-optimization-of-subqueries-and-aggregation/example.png)

## Apply 引发的其他优化

在子查询改写成 Apply 算子，以及 Apply 算子下推、消除的过程中，会诞生聚合和 Join 算子，因此也引发了其他的优化规则。我们先来看看聚合的上拉和下推。

### 聚合的上拉和下推



### Outer Join 化简



## Segemented Apply



