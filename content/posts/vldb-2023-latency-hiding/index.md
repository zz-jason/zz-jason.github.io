---
title: "[VLDB 2023] The Art of Latency Hiding in Modern Database Engines"
date: 2024-04-12T00:00:00Z
categories: ["Paper Reading", "Storage"]
draft: true
---

## 简介

这篇论文是 CoroBase 的延续，在 CoroBase 中提出了利用 C++ 20 无栈协程以及 CPU prefetch 指令减少 cache miss 提升执行效率的方法。这篇论文在  CoroBase 的基础上进一步通过异步 IO 和精心设计的协程调度算法来提升文件读写效率，进一步提升数据库系统的事务执行吞吐。

读这篇论文时重点关注两个方面的内容即可：如何利用 coroutine 和异步 IO 接口（以 io_uring 为例）优化系统吞吐，如何设计协程调度算法使系统尽可能在等待 IO 完成的同时执行纯内存事务？