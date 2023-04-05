---
title: "[SIGMOD 1979] Access Path Selection in a Relational Database Management System"
date: 2020-09-12T16:13:29Z
categories: ["Paper Reading"]
draft: true
---

## 简介

《[Access Path Selection in a Relational Database Management System](https://courses.cs.duke.edu/compsci516/cps216/spring03/papers/selinger-etal-1979.pdf)》 是 Selinger 1979 年发表在 SIGMOD 上的论文，主要讲了早起 System R 查询优化器的设计和实现原理。

System R 是第一个 cost based optimizer（CBO），它采用了一种加权的 cost 估算方式综合考虑 CPU、IO、执行时间等维度，根据算子的 interesting order 采用 bottom-up 的搜索方式求解动态规划的最优子结构。它也提出了一种枚举所有 join order 的启发式算法，使得优化器能够在比较快的时间内找到一个还不错的 join order。

System R 优化器对后续数据库的设计有着很多启发和影响，是优化器领域必读论文之一。希望这篇论文分享能够帮助到感兴趣的朋友们。

本文后续的内容组织方式和论文差不多，我们会简单了解 System R 中一条 SQL 的执行过程， 接着熟悉一下 System R 的存储引擎接口，这对我们理解 interesting order 和 cost 估算有帮助，然后我们再看一条 SQL 是如何被 System R 优化的。

## 一条 SQL 的处理过程

System R 中一条 SQL 的处理过程总共分为 4 个阶段：parsing、optimization、code generation 以及 execution。parsing 主要将 SQL 文本转成 AST，optimization 主要完成类型