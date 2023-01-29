---
title: "[PostgreSQL] B Tree 源码阅读"
date: 2023-01-28T13:32:24Z
categories: ["PostgreSQL"]
draft: true
---

PostgreSQL 的 B Tree 是一个并发安全、能够灾难恢复的 B Link Tree，主要参考下面两篇论文：

- Efficient Locking for Concurrent Operations on B-Trees
- A Symmetric Concurrent B-Tree Algorithm

代码目录：
- 接口定义：src/include/access/nbtree.h
- 接口实现：src/backend/access/nbtree/*.c

## B Tree 的创建和初始化

```c
extern void btbuildempty(Relation index);
extern IndexBuildResult *btbuild(Relation heap, Relation index,
								 struct IndexInfo *indexInfo);
```

btbuildempty：仅构造 B Tree 的 meta page，



## B Tree 的插入

```c
extern bool btinsert(Relation rel, Datum *values, bool *isnull,
					 ItemPointer ht_ctid, Relation heapRel,
					 IndexUniqueCheck checkUnique,
					 bool indexUnchanged,
					 struct IndexInfo *indexInfo);
```

## B Tree 的读取
