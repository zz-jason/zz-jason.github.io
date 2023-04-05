---
title: "[TODS 2010] A Survey of B-Tree Locking Techniques"
date: 2023-04-04T08:00:00Z
categories: ["Paper Reading"]
draft: true
---

## Introduction

单线程的 B Tree 比较好实现，但是涉及到多线程并发，以及在 B Tree 之上支持事务很难：
> The basic data structure and algorithms for B-trees are well understood. Search, insertion, and deletion are often implemented by college students as programming exercises, including split and merge operations for leaves and interior nodes. Accessing B-tree nodes on disk storage using a buffer pool adds fairly little to the programming effort. Variable-length records within fixed-length B-tree nodes add moderate complexity to the code, mostly book- keeping for lengths, offsets, and free space. It is far more challenging to enable correct multithreaded execution and even transactional execution of B-tree operations.

B Tree 的 concurrency control 算法中有非常多的名词，需要正确的区分才能更好的实现：
> The plethora of terms associated with concurrency control in B-trees may seem daunting, including row-level locking, key value locking, key range lock- ing, lock coupling, latching, latch coupling, and crabbing, the last term applied to both root-to-leaf searches and leaf-to-leaf scans. A starting point for clarify- ing and simplifying the topic is to distinguish protection of the B-tree structure from protection of the B-tree contents, and to distinguish separation of threads against one another from separation of user transactions against one another. These distinctions are central to the treatment of the topic here. Confusion between these forms of concurrency control in B-trees unnecessarily increases the efforts for developer education, code maintenance, test development, test execution, and defect isolation and repair.

### Historical Background

### Overview

## Preliminaries

本文假设所有数据像 B* Tree 或 B+ Tree 那样存储在叶子结点：
> Most work on concurrency control and recovery in B-trees assumes what Bayer and Schkolnick [1977] call B∗-trees and what Comer [1979] calls B+-trees, that is, all data records are in leaf nodes and keys in nonleaf or “interior” nodes act merely as separators enabling search and other operations but not carrying logical database contents. We follow this tradition here and ignore the original design of B-trees with data records in both leaves and interior nodes.

因为和 Locking 关系不大，这篇文章也忽略了这些方面：
- attempting to merge an overflowing node with a sibling rather than splitting it immediately.
- We ignore whether or not underflow is recognized and acted upon by load balancing and merging nodes
- whether or not empty nodes are removed immediately or ever
- whether or not leaf nodes form a singly or doubly linked list using physical pointers (page identifiers) or logical boundaries (fence keys equal to separators posted in the parent node during a split)
- whether suffix truncation is employed when posting a separator key [Bayer and Unterauer 1977]
- whether prefix truncation or any other compression is employed on each page, and the type of information associated with B-tree keys

Most of these issues have little or no bearing on locking in B-trees, with the exception of sibling pointers, as indicated in the following where appropriate.

## Two Forms of B-Tree Locking

B-tree locking, or locking in B-tree indexes, means two things.
- First, it means concurrency control among concurrent database transactions querying or mod- ifying database contents and its representation in B-tree indexes.
- Second, it means concurrency control among concurrent threads modifying the B-tree data structure in memory, including in particular images of disk-based B-tree nodes in the buffer pool.

### Locks and Latches

![Fig 2. Locks and latches](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202304041327784.png)

数据库中的 “Lock” 用来隔离两个并发的用户事务：
- Locks separate transactions using read and write locks on pages, on B-tree keys, or even on gaps (open intervals) between keys.
- By default, locks participate in deadlock de- tection and are held until end-of-transaction.
- Locks also support sophisticated scheduling, for example, using queues for pending lock requests and delaying new lock acquisitions in favor of lock conversions
- This level of sophistication makes lock ac- quisition and release fairly expensive, often thousands of CPU cycles, some of those due to cache faults in the lock manager’s hash table and its linked lists.

比较不幸的是，OS 使用 "Lock" 这个词来表示数据库中的 “Latch”：Unfortunately, the literature on operating systems and programming environments usually uses the term locks for the mechanisms that in database systems are called latches.
- Latches separate threads accessing B-tree pages cached in the buffer pool, the buffer pool’s management tables, and all other in-memory data structures shared among multiple threads.
- Since the lock manager’s hash table is one of the data structures shared by many threads, latches are required while in- specting or modifying a database system’s lock information.
- With respect to shared data structures, even threads of the same user transaction conflict if one thread requires a write latch.
- Latches are held only during a critical sec- tion, that is, while a data structure is read or updated. Deadlocks are avoided by appropriate coding disciplines, for example, requesting multiple latches in carefully designed sequences. Deadlock resolution requires a facility to roll back prior actions, whereas deadlock avoidance does not.
- Latch acquisition and release may require tens of instructions only, usually with no additional cache faults since a latch can be embedded in the data structure it protects. For images of disk pages in the buffer pool, the latch can be embedded in the descriptor structure that also contains the page identifier, etc.

作者后面举了一个 non-clustered index 的例子来说明数据库中 Latch 和 Lock 的区别。

### Recovery and B-Tree Locking

The difference between latches and locks is important not only during normal transaction processing but also during recovery from a system crash.

### Lock-Free B-Trees

Lock-Free 中的 Lock 在不同的上下文中含义不一样：
- it might mean that the in-memory format is such that pessimistic concurrency control in form of latches is not required. Multiple in-memory copies and atomic updates of in-memory pointers might enable such an implemen- tation. Nonetheless, user transactions and their access to data must still be coordinated, typically with locks such as key range locking on B-tree entries.
- On the other hand, a lock-free B-tree implementation might refer to avoid- ance of locks for coordination of user transactions. A typical implementation mechanism would require multiple versions of records based on multiversion concurrency control. However, creation, update, and removal of versions must still be coordinated for the data structure and its in-memory image. In other words, even if conflicts among read transactions and write transactions can be removed by means of versions instead of traditional database locks, modifica- tions of the data structure still require latches.

## Protecting A B-Tree’s Physical Structure

The simplest form of a latch is a “mutex” (mutual exclusion lock).

Since latches protect in-memory data structures, the data structure repre- senting the latch can be embedded in the data structure being protected. Latch duration is usually very short and amenable to hardware support and encapsu- lation in transactional memory

Since locks often protect data not even present in memory, and sometimes not even in the database, there is no in-memory data structure within which the data structure representing the lock can be embedded. Thus, database systems em- ploy lock tables. Since a lock table is an in-memory data structure accessed by multiple concurrent execution threads, access to the lock table and its compo- nents is protected by latches embedded in the data structures.

### Issues

1. First, a page image in the buffer pool must not be modified (written) by one thread while it is interpreted (read) by another thread.
2. Second, while following a pointer (page identifier) from one page to another, for example, from a parent node to a child node in a B-tree index, the pointer must not be invalidated by another thread, for instance, by deallocating a child page or balancing the load among neighboring pages.
3. Second, while following a pointer (page identifier) from one page to another, for example, from a parent node to a child node in a B-tree index, the pointer must not be invalidated by another thread, for instance, by deallocating a child page or balancing the load among neighboring pages.
4. Fourth, during a B-tree insertion, a child node may overflow and require an insertion into its parent node, which may thereupon also overflow and require an insertion into the child’s grandparent node. In the most extreme case, the B-tree’s old root node overflow must split, and be replaced by a new root node. Going back from the leaf towards the B-tree root works well in single-threaded B-tree implementations, but it introduces the danger of deadlocks between root-to-leaf searches and leaf-to-root B-tree modifications, yet latches rely on deadlock avoidance rather than deadlock detection and resolution.

### Lock Coupling

The second issue given earlier requires retaining the latch on the parent node until the child node is latched. This technique is traditionally called “lock cou- pling” but a better term is “latch coupling” in the context of transactional database systems.

### B-link Tree

### Load Balancing and Reorganization

### Protecting A B-Tree’s Logical Contents

### Key Range Locking

### Key Range Locking and Ghost Records

### Key Range Locking as Hierarchical Locking

### Locking in Nonunique Indexes

### Increment Lock Modes

## Future Directions
