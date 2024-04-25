---
title: "[VLDB 2018] Plan Stitch: Harnessing the Best of Many Plans"
date: 2024-04-22T00:00:00Z
categories: ["Paper Reading", "Query Optimization"]
draft: true
---

## 简介

这篇论文提出了 Plan Stitch，核心思想是把过去执行过的 best plan 和对应的 subplan 当做搜索空间，把过去执行的平均 cpu 时间作为 runtime cost，用 system R 类似的方式在这个搜索空间和 cost model 上缝合出（stitch）一个新的更优的 plan。

因为每个 best plan 和其 subplan 都是过去执行过的，这个缝合出的 best plan 发生 regression 的风险非常低，而且它的搜索空间局限在过往执行过的 best plan 和 subplan 上，整体优化时间也非常可控，可以直接用来增强优化器的 SQL Plan Management：在 schema change 之后，仍然可以使用过往的 best plan 得出在新的 schema 上 work 的低风险、高效的新执行计划。

利用 subplan 和 runtime cost 指导 query optimization 的 idea 应该也能进一步扩展，整体来看挺实用的。这篇文章着重介绍 Plan Stitch 的核心原理，推荐感兴趣的朋友阅读原论文。

作者在 SQL Server 实现了 Plan Stitch，对 TPC-DS 和三个真实工作负载进行了测试验证。实验结果显示 Plan Stitch 获得的执行计划可以显著降低执行成本，甚至比回滚到先前执行计划的成本更低两个数量级。

## 问题描述

通常来说，我们希望优化器能够利用当前的索引和统计信息，稳定的生成高效执行计划。但因为这样那样的原因，有时候同一个 SQL 生成的执行计划可能比之前更差，也就是发生了 Plan Regression。

这个问题在 SQL Server 中可以通过 Automatic Plan Correction (APC) 来解决，其他数据库也有类似 SQL Server APC 的解决方案，比如 Oracle 的 SQL Plan Management (SPM)。APC 的主要思路是统计该 SQL 的历史执行计划和执行时间，然后采用历史上执行效率最高的 Plan：

> Once a plan completes execution and has statisticall-significant worse execution cost compared to earlier plans of the same query observed in recent history, the server automatically forces the optimizer to execute the query with the cheaper older plan which is still valid.

类似 APC 这样的 reversion-based plan correction（RBPC）风险低，大部分情况下也非常有效，在生产环境上非常受欢迎。但它也有一定的缺陷：只能在所有执行过的 plan 中选择代价最优的。因为这些 plan 都被执行过，那我们就拥有每个 plan 在算子级别的 execution cost，我们有机会根据所有这些 plan 的 subplan 和它们的 execution cost 组合出新的代价更优的 plan，同时因为这些 subplan 都被执行过，组合出的新 plan 和过往的老 plan 一样 regression 风险都很低。

这个缺陷总结来说就是 RBPC 以 plan 粒度的修正方式限制了发现更优 plan 的可能性。如果能以 subplan 为粒度，综合所有执行过的 plan 的 subplan 和它们的 execution cost statistics，我们有机会设计出比传统 RBPC 更好的 plan correction algorithm。

## Plan Stitch 核心思路

![](20240422231138.png)
以上图 A、B、C、D 四表 join 为例。一开始它的执行计划 p1 如图 a，所有的 join 全部是 hash join，后来用户为表 B 和 D 创建了能被 nested loop join 使用的索引，优化器产生了新的执行计划 p2 如图 b。在 p2 中因为估算误差的原因优化器认为 HashJoin(HashJoin(A, B), C) 比 NLJ(NLJ(C, B), A) 的代价更大，于是得到了 p2 的执行计划。但实际执行后发现 p2 出现了性能回退，实际上 HashJoin(HashJoin(A, B), C) 的执行代价（80）比 NLJ(NLJ(C, B), A) 的执行代价（300）更低。按照传统 RBPC（比如 SQL Server APC）的策略此时该查询的 plan 应该回退到 p1。

但从 p2 的 execution cost 可以看到，对 D 表执行 NLJ 的 execution cost (50+150) 比 Hash Join 的 execution cost (100+300) 更低。如果能像 p1 那样对 A、B、C 采用 HashJoin(HashJoin(A, B), C)，像 p2 那样最后采用 NLJ((A, B, C), D)，那么就能得到一个比 p1 更优的新 plan，也就是图 c 所示的 p3。

上面这个例子展示了 plan stitch 的 idea 和目标：根据过往执行的 plan 和 execution cost 信息，得到 execution cost 最优的 plan。这个 plan 可能是某个历史上执行最优的 plan，也可能是某些执行过的 plan 的 subplan 的组合。

那怎么实现 plan stitch 呢？使用经典优化器的动态规划方法即可：对于某个 logical expression，只从历史执行计划中选择被执行过且仍旧有效的 physical expression，并使用他们的 execution cost 进行代价估算。这样递归下去即可得到最优的 stitched plan。

相比 RBPC（比如 SQL Server APC 或 Oracle SPM），plan stitch 有着相似的优点，同时也能尽可能利用历史执行计划。计算过程本身开销都很低、得到的 plan 在之前执行过程中已经被验证过是比较高效的，性能回退的风险很低。相比 RBPC 来说，即使某个 plan 因为 schema change 的原因整体不可用了，但只要它的某个 subplan 仍然可用，这样的 subplan 也可以被 plan stitch 利用上，能够最大限度的利用历史执行计划。

相比基于 query feedback 的查询优化来说，plan stitch 将搜索空间限制在过去执行过的物理执行计划上，不会像基于 query feedback 的查询优化那样产生新的、未执行过的 subplan。plan stitch 的这个特点使它比 feedback 机制更适合那些对 plan 稳定性要求高的场景中。

值得注意的是，即使没有 plan regression，也可以在日常使用 plan stitch 基于历史生成更优的执行计划。考虑到 plan stitch 过程本身开销很低，plan stitch 完全可以默认开启，作为优化器的一个辅助机制。

对于参数化的 sql 或者 prepare 语句，plan stitch 和 RBPC 一样存储和使用所有 plan 的 avg execution cost。也可以根据需要采用带权重的 cost 计算方式。

![](20240422232225.png)
Plan Stitch 整体架构如上图所示，plan stitch 作为 sql server 优化器的一个扩展组件，使用 sql server 已有的 query store 功能存储历史 plan、获取算子级别的 execution cost 等信息，通过 plan guide、USE PLAN hint 等功能 force 优化器采用 stitched plan。

从测试结果来看 plan stitch 在广度上能比 RBPC 优化更多的 query，在深度上能大幅减少 query 的 execution cost：经过 RBPC 优化后的查询中，83% 都能经过 plan stitch 进一步优化，减少至少 10% 的 execution cost，甚至有些 query 能减少 2 个数量级的 execution cost。同时，plan stitch 能使用 invalid plan 的 subplan，能优化的 plan 是 RBPC 的 20 倍。

## Plan Stitch 搜索空间

## Plan Stitch 代价估算

## Plan Stitch 搜索算法

![](20240422232440.png)
如上图所示，Plan Stitch 的搜索算法和 System R 优化器的搜索算法非常相似
## 实现细节

## 实验结果

