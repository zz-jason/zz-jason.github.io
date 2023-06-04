---
title: "[VLDB 2022] Excalibur: A Virtual Machine for Adaptive Fine-grained JIT-Compiled Query Execution based on VOILA"
date: 2023-02-19T03:22:58Z
categories: ["Paper Reading"]
draft: true
---

这篇论文来自 CWI，CWI 产出了大量 vectorized 相关优化的论文，它们的数据库研究水平一向很高，想必这篇论文也不差。

## 简介

### 解决的问题是什么？

论文提出了一个非常实际的问题：

> In recent years, hardware has become increasingly diverse, in terms of features as well as performance. This poses a problem for complex software in general and database systems in particular. To achieve top-notch performance, we need to exploit hardware features, but do not fully know which behave best on the current, and more-so future, machines. Specializing query execution methods for many diverse hardware platforms will significantly increase database software complexity and also poses a physical query optimization problem that cannot be solved robustly with static cost models.
>
> so the question is how one can create future-proof database systems?

为了实现优秀性能的数据库，我们需要面向硬件优化，但硬件规格是多种多样的，并且随着时间的发展硬件也在不断发展，为了不断适配不同的硬件规格以及将来的硬件发展，我们需要不断优化数据库代码，使得数据库开发变得非常复杂。

### 解决的整体思路是什么？

> we propose a new query execution architecture addressing these problems. Based on the flexible domain-specific language VOILA, it can generate thousands of different flavors from a single code-base. As an abstraction, a virtual machine (VM) allows hiding physical execution details, such that the VM can transparently switch between different execution tactics within each query, applied at a fine granularity.
>
> The VM starts executing each query in full vectorized code style, but adaptively replaces (parts of) query pipelines by code fragments compiled using different execution flavors, exploring this search space and exploiting the best tactics found, casting adaptive query execution into a Multi-Armed Bandit (MAB) problem.
>
> The gist of its DSL is that it logically encodes the operations to be performed on data as well as the layout of the data structures, yet it does not specify the order or execution-style of these operations, and instead tries to create independence of these operations; thus providing freedom to execute them in different ways.

基于 VOILA 这个 DSL 实现了一个数据库虚拟机，在执行 query 时虚拟机根据 query 的特点，为 query 的某一致性过程选择合适的执行策略（LLVM JIT、向量化），打到整体性能提升的目的。要达到这个目的，需要解决这些问题：



问题 1：切换执行策略的依据是什么：

> The classic approach would have been to construct detailed physical cost models for VOILA, and solve the problem of finding the best flavor statically, before query execution starts, as a physical query optimization problem. However, such physical cost models would need to be created for every hardware type that exists, as we found that flavor performance varies strongly among different hardware [21], and would never be future-proof. Instead, **we pursue a micro-adaptive approach** [42].



问题 2：以什么粒度切换执行策略，上下游的计算过程中，不同的执行策略如何衔接：

> A crucial property we exploit is that all flavors generated from a VOILA program operate on the same data structures, which allows to switch flavors in-flight and at fine granularity: switching flavors can be done for the whole query, or only for one pipeline, or just a fragment of a pipeline.



到这我们基本上对要解决的问题和解决思路有个初步印象了，接下来我们就来详细看看 Excalibur 这个 VM 内部的详细设计吧。论文也给出了 Excalibur 的 source code，感兴趣的朋友可以结合代码一起看看这个系统是如何实现的：https://github.com/t1mm3/db_excalibur。



