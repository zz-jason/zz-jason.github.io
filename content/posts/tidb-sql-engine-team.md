---
title: "TiDB SQL Engine Team：纯手工打磨前沿的优化器和执行引擎｜PingCAP 招聘季"
date: 2020-03-25T06:26:16Z
draft: false
---

“SQL at SCALE”（出自 [PingCAP 官网](https://pingcap.com/)）是我们对 TiDB 的一个精简概括，而我们 TiDB SQL Engine Team 正是负责这 3 个单词中的 “SQL” 部分，其重要性可见一斑。SQL 在数据库中的大致处理流程可以简短概括为查询优化和执行，这期间涉及到 SQL Parser、优化器、统计信息和执行引擎等模块，他们就是 TiDB SQL Engine Team 目前所负责的模块。接下来我会用简短的篇幅向大家介绍 SQL Engine 的背景知识，以及我们在做的事情，面临的挑战等。

## 关于查询优化

优化器是 SQL 引擎的大脑，负责查询优化。查询优化的主要工作概括起来很简单：搜索可行的执行计划，从中挑一个最好的。但要做好这两件事却是整个分布式数据库中最难的地方。

1979 年 Selinger 发布了 "[Access Path Selection in a Relational Database Management System](https://courses.cs.duke.edu//compsci516/cps216/spring03/papers/selinger-etal-1979.pdf)"，正式拉开了 Cost Based Optimization 的帷幕，这篇论文也被视为 CBO 优化器的圣经。在这之后陆续出现了 [Starburst](https://people.eecs.berkeley.edu/~brewer/cs262/23-lohman88.pdf)（1988 年），[Volcano Optimizer Generator](https://15721.courses.cs.cmu.edu/spring2017/papers/14-optimizer1/graefe-icde1993.pdf)（1993 年）和 [Cascades Framework](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/Cascades-graefe.pdf)（1995 年） 等，每年数据库三大顶会中也能看到不少查询优化相关的论文，整个优化器领域可谓是蓬勃发展。但即使如此，优化器也仍然有很多问题未能得到很好的解决，比如：

1. Guy Lohman 2014 年在 “[Is Query Optimization a “Solved” Problem?](https://wp.sigmod.org/?p=1075)” 中详细讲述的 SQL 算子结果集估算的难题。简单来说，要估算某个表需要扫多少行数据比较容易，但是要再估算更上层的 SQL 算子，比如 Join 或者 Join 之后再 Group By 的结果集有多大，这个就很难了。可以想象的是，估算误差会随着层数的增加而被放大，这个放大有时候是数量级的。此外还会出现负负得正的情况：明明估算错了，但是执行计划却是对的，纠正估算误差后，执行计划反而不对了。

2. Viktor Leis 等人在 2015 年的论文 [How Good Are Query Optimizers, Really?](http://www.vldb.org/pvldb/vol9/p204-leis.pdf) 中讨论了优化器的另一朵乌云：Join Order。如果枚举所有可行的 Join Order，光是考虑左深树，N 个表的 Join 就可能有 N! 种执行计划。目前大家普遍采用一种妥协的方案：当参与 Join 的表比较少时用动态规划来确定 Join 的顺序，表比较多的时候用贪心或者遗传算法（PG 用的模拟退火）来做。但是采用什么样的动态规划和搜索算法也仍然处在热烈的研究中，而算子结果集的估算误差又进一步让这个问题雪上加霜，难上加难。

作为一个从头到尾完全自己手写的优化器，TiDB 优化器的发展历史也算精彩：一开始我们是 Selinger 的 System R 模型，但是它的扩展性不是很好，搜索空间有限，维护成本也高，于是我们调研后，决定开发 Cascades 模型的新优化器（具体请参考：[十分钟成为 Contributor 系列| 为 Cascades Planner 添加优化规则](https://pingcap.com/zh/blog/10mins-become-contributor-20191126) 和 [揭秘 TiDB 新优化器：Cascades Planner 原理解析](https://pingcap.com/zh/blog/tidb-cascades-planner)）。在开发 Cascades Planner 的同时，我们还在做着另外一件非常重要的事情，提升优化器的稳定性：

1. 优化器的稳定性非常重要。去年之前我们经常遇到选错索引，或者干脆不选索引的问题。这个对业务的影响非常大，有时候一个慢查询可能拖垮整个集群，很多用户都吐槽过这个问题。后来调查研究后，我们引入了 Skyline Pruning 的剪枝优化，极大地提升了优化器选择索引的稳定性。参考：[Proposal: Support Skyline Pruning](https://github.com/pingcap/tidb/blob/master/docs/design/2019-01-25-skyline-pruning.md)。

2. 优化器的稳定性非常重要。要稳定的做出好的执行计划，统计信息非常非常关键。以前我们收集统计信息需要整个表都扫描一遍，扫的过程中用蓄水池算法做抽样。小表这样做没啥问题，大表也这样做就不行了：一方面担心对正在运行的业务造成影响，另一方面这种方式也很低效。于是我们结合 TiKV 的存储特点引入了 Fast Analyze，极大的提升了统计信息的搜集速度，也降低了对业务负载的影响。参考：[PR/10214](https://github.com/pingcap/tidb/pull/10214)。

3. 优化器的稳定性非常重要。即使我们做了各种优化，解了各种 Bug，仍然会出现执行计划不优的问题。有条件的用户还可以改一改 SQL，那没条件的呢？比如 SQL 是通过第三方工具自动拼接的怎么改？为了解决这些问题，我们决定引入 [SQL Plan Management](https://github.com/pingcap/tidb/projects/19)，先实现了给 SQL 绑定执行计划的功能，使得不用更改业务也能抢救 SQL 的执行计划（[Issue/8935](https://github.com/pingcap/tidb/issues/8935)）；为了能够应对更多业务场景，更加细粒度的控制优化行为，我们还丰富了 SQL Hint 集合（[Issue/12304](https://github.com/pingcap/tidb/issues/12304)）；为了让 SQL 执行计划不会变差，我们为 SQL 确定了 Plan 的 Baseline，并且再往前走一步，我们做了 Baseline 的自动演进，使得执行计划不但不会变坏，而且只会变的越来越好。

重要的事情重复 3 遍：优化器的稳定性非常重要。

除了稳定性之外，还有性能问题：

- 如何在尽量短的时间内消耗尽量少的硬件资源找到最佳执行计划？
- 而目前 TiDB 正在 HTAP 之路上迈出坚实步伐，如何自动识别一条 SQL 是 AP 还是 TP 查询？
- 如何为 TP 查询选择合理的索引？
- 又如何为 AP 查询做出一个高效的分布式执行计划？

可以预见，在这条道路上，优化器又将迎接新的困难和挑战，不断自我演进。

## 关于查询执行

我的第一份工作从执行引擎开始，对它的感情异常深厚。执行引擎的目标是尽量利用计算资源，正确且快速的完成执行计划所描述的计算任务。光有看起来很完美的执行计划，却没有高效的执行引擎，整个 SQL 引擎也是废的。

执行引擎也是一个热门的研究领域。最经典的执行模型当属 1994 年 Goetz Graefe 发表的 [Volcano 迭代器模型](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf)，至今仍被广大数据库使用。原因很简单：接口抽象度高，扩展性好，实现起来简单。在数据量不大的 TP 请求中，这种模型足够用了。不过后来大家发现，随着数据量的上升，这玩意的执行性能很差：每完成一条数据的计算，要额外花费的很多 CPU 指令，计算效率非常低。于是有了后来的两大优化方向：Vectorization 和 Compilation，各自的代表分别为：2005 年 Marcin Zukowski 的 ”[MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf)” 和 2011 年 Thomas Neumann 的 “[Efficiently Compiling Efficient Query Plans for Modern Hardware](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf)”。

除了执行框架，如何利用 CPU 硬件特性优化各种执行算子也被广泛的讨论和研究。比如 2013 年的 “[Multi-Core, Main-Memory Joins: Sort vs. Hash Revisited](http://www.vldb.org/pvldb/vol7/p85-balkesen.pdf)” 这篇论文详细的探讨和对比了 Hash Join 和 Merge Join 的实现和性能，2015 年的 “[Rethinking SIMD Vectorization for In-Memory Databases](http://www.cs.columbia.edu/~orestis/sigmod15.pdf)” 这篇论文详细讨论了如何利用 SIMD 指令提升 SQL 算子性能。此外，底层软硬件技术的革新带来更多的优化机会，比如还有一系列论文来讨论如何适配 NUMA 架构，提升算子执行性能等。

作为一个从头到尾完全自己手写的执行引擎，TiDB 执行引擎的发展也非常丰富多彩：一开始我们使用的是传统 Volcano 迭代器模型，后来我们和社区同学在 TiDB 2.0 版本中将其优化成了向量化模型（[Issue/5261](https://github.com/pingcap/tidb/issues/5261)），得到了巨大的性能提升：[TPC-H 50G, TiDB 2.0 VS 1.0](https://github.com/pingcap/docs-cn/blob/master/v2.1-legacy/benchmark/tpch.md)。之后我们和社区同学优化了聚合算子，重构了整个聚合函数的执行框架，执行性能又取得了飞跃的发展（Issue/6952）。再之后，我们和社区同学优化了表达式执行框架，使得表达式执行效率得到了 10 倍的性能提升，这期间 “[10x Performance Improvement for Expression Evaluation Made Possible by Vectorized Execution and the Community](https://en.pingcap.com/blog/10x-performance-improvement-for-expression-evaluation-made-possible-by-vectorized-execution/)” 这篇文章还占据了 Hacker News 的首页和 DZone Database 头版头条。

稳定性和易用性也非常重要。为了解决用户 OOM 的问题，我们先后引入了内存追踪和记录的机制，后来干脆让算子落盘真正解决内存使用过多的问题，另外我们也在优化排查问题的调查工具，方便在出问题时快速定位和 workaround。

如前文所说，目前 TiDB 正在 HTAP 之路上迈出坚实的步伐。执行引擎将在新的征程上肩负着新的使命。在分布式数据库中，广义上的执行引擎需要考虑更多的事情：任务如何调度？shuffle 如何优化？目前三套执行引擎（TiDB、TiKV、TiFlash）三套代码的维护成本如何降低？这些问题都等待着我们去探索和解决，可以预见，在这条道路上，执行引擎又将迎接新的困难和挑战，不断自我演进。

## 期待你的加入

很开心，TiDB 的优化器和执行引擎是从零开始由我们的小伙伴们纯手工打造的，我们有很大的自由度来发挥自己的创造力；很紧张，上面这些列出来的种种问题我们都会遇到；很荣幸，我们能够和业界大牛、广大开源爱好者们一起来攻克这些难题；也很有成就感，我们能在广大 TiDB 用户的业务中看到这些改进为他们带来的价值。

我们热爱开源，相信开源能够为我们的产品带来巨大的收益，也愿意为开源奉献，非常期待同样热爱开源的你的加入。如果你：

- 热爱和相信开源，聪明且有激情；
- 敢于挑战上面那些难题，突破极限；
- 熟悉分布式系统、优化器和执行引擎的实现，熟悉 CPU 硬件特性；
- 有团队带领经验（加分项）。
- 那么我们就加入我们吧，一起向这些难题发起挑战，构建一个前沿、稳定的优化器和高效易用的执行引擎。

欢迎联系：zhangjian@pingcap.com