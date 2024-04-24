---
title: "[VLDB 2023] Write-Aware Timestamp Tracking: Effective and Efficient Page Replacement for Modern Hardware"
date: 2023-08-31T00:00:00Z
categories: ["LeanStore", "Storage"]
draft: true
---

## 2.1. What Database Systems Use

Least Recently Used (LRU), or its variants and approximations：
1. DB2：LRU，FIFO
2. Oracle：LRU, partitions the LRU lists to reduce latch contention
3. SQL Server: LRU2, which is an instantiation of LRU-K [15], LRU2 uses the next-to-last access to find replacement candidates.
4. InnoDB: modifies LRU by inserting new pages in the mid- dle (instead of the front) of the LRU list. This ensures that large table scans do not thrash the entire buffer pool (but only a part of it).
5. PostgreSQL: Clock-Sweep, a variation of CLOCK.
6. WiredTiger: an approximation of LRU. The implementation scans the whole RAM for pages older than a certain threshold and replaces them.

## 2.2 Advanced Replacement Algorithms

LRU-K: replaces pages based on their Kth-to-last accesses timestamp. In practice, K is set to 2, which effectively means that the most recent access is ignored.

the 2Q [7] algorithm: having two lists instead of one such that pages with very different frequencies can be separated. for example, pages are first entered into a FIFO list, and only if they are accessed a second time, they are moved to an LRU list

the Adaptive Replacement Cache (ARC) [13] algorithm: relies on two lists, but can adaptively determine their sizes based on the workload.