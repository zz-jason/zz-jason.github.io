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

PG 中，WAL 和 B Tree 在内存中都是通过一个个的 Page Buffer 来存储。这些数据最终落到磁盘上时会切割成一个个 segment，每个 segment 对应一个磁盘文件。WAL 和 BTree 的 block size 都是 8KB：

```cpp
#define BLCKSZ 8192
#define XLOG_BLCKSZ 8192
```

## B Tree 的创建和初始化

```c
extern void btbuildempty(Relation index);
extern IndexBuildResult *btbuild(Relation heap, Relation index,
								 struct IndexInfo *indexInfo);
```

```c
/*
 *	btbuildempty() -- build an empty btree index in the initialization fork
 */
void
btbuildempty(Relation index)
{
	Page		metapage;

	/* Construct metapage. */
	metapage = (Page) palloc(BLCKSZ);
	_bt_initmetapage(metapage, P_NONE, 0, _bt_allequalimage(index, false));

	/*
	 * Write the page and log it.  It might seem that an immediate sync would
	 * be sufficient to guarantee that the file exists on disk, but recovery
	 * itself might remove it while replaying, for example, an
	 * XLOG_DBASE_CREATE* or XLOG_TBLSPC_CREATE record.  Therefore, we need
	 * this even when wal_level=minimal.
	 */
	PageSetChecksumInplace(metapage, BTREE_METAPAGE);
	smgrwrite(RelationGetSmgr(index), INIT_FORKNUM, BTREE_METAPAGE,
			  (char *) metapage, true);
	log_newpage(&RelationGetSmgr(index)->smgr_rlocator.locator, INIT_FORKNUM,
				BTREE_METAPAGE, metapage, true);

	/*
	 * An immediate sync is required even if we xlog'd the page, because the
	 * write did not go through shared_buffers and therefore a concurrent
	 * checkpoint may have moved the redo pointer past our xlog record.
	 */
	smgrimmedsync(RelationGetSmgr(index), INIT_FORKNUM);
}
```

btbuildempty：仅构造 B Tree 的 meta page。PG 每个内存中的 block 都有一个 block number，meta page 的 block number 为 0。构造过程主要是下面 2 步：

1. 在内存中新建并初始化一个 8KB 的 metapage，通过 `smgrwrite()` 将其写入到磁盘文件，在记录完 wal 后通过 `smgrimmedsync()` 显式的 fsync 持久化 metapage 到磁盘中。
2. 通过 `log_newpage()` 为这个 metapage 记录 wal，写入到 wal 文件中。

### `smgrwrite()`：写入一个 8KB block

PG 通过 `Relation` 来表示一个 B tree，每个 `Relation` 对应一个 `SMgrRelation`（SMgr 应该是 storage manager 的缩写）。在为该 Relation 写 BTree page 或 WAL 的时候都需要通过 `RelationGetSmgr()` 获取它的 SMgrRelation。

在 `smgrwrite()` 中我们会发现，PG 会调用其对应的 `SMgrRelation` 的 `mdwrite()` 函数写入一个 8KB 的内存 block 到文件中。（这里的写入如果没有调用 fsync，可能仅仅是 buffer 在文件系统的缓存中，并没有落盘）。

```c
/*
 *	smgrwrite() -- Write the supplied buffer out.
 *
 *		This is to be used only for updating already-existing blocks of a
 *		relation (ie, those before the current EOF).  To extend a relation,
 *		use smgrextend().
 *
 *		This is not a synchronous write -- the block is not necessarily
 *		on disk at return, only dumped out to the kernel.  However,
 *		provisions will be made to fsync the write before the next checkpoint.
 *
 *		skipFsync indicates that the caller will make other provisions to
 *		fsync the relation, so we needn't bother.  Temporary relations also
 *		do not require fsync.
 */
void
smgrwrite(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum,
		  char *buffer, bool skipFsync)
{
	smgrsw[reln->smgr_which].smgr_write(reln, forknum, blocknum,
										buffer, skipFsync);
}
```

上面的 `smgr_write()` 是个函数指针，最终会调用到 `mdwrite()` 这个函数来。这个函数看起来比较长，但逻辑比较简单。先通过 block number 定位到具体的 segment 和在该 segment 中的 pos，然后在该 pos 写入 `buffer` 中的 8KB 数据：

```c
/*
 *	mdwrite() -- Write the supplied block at the appropriate location.
 *
 *		This is to be used only for updating already-existing blocks of a
 *		relation (ie, those before the current EOF).  To extend a relation,
 *		use mdextend().
 */
void
mdwrite(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum,
		char *buffer, bool skipFsync)
```

### `log_newpage()`：为创建 page 的操作记录 WAL

```c
XLogRecPtr
log_newpage(RelFileLocator *rlocator, ForkNumber forknum, BlockNumber blkno,
			Page page, bool page_std)
{
	int			flags;
	XLogRecPtr	recptr;

	flags = REGBUF_FORCE_IMAGE;
	if (page_std)
		flags |= REGBUF_STANDARD;

	XLogBeginInsert();
	XLogRegisterBlock(0, rlocator, forknum, blkno, page, flags);
	recptr = XLogInsert(RM_XLOG_ID, XLOG_FPI);

	/*
	 * The page may be uninitialized. If so, we can't set the LSN because that
	 * would corrupt the page.
	 */
	if (!PageIsNew(page))
	{
		PageSetLSN(page, recptr);
	}

	return recptr;
}
```



写 WAL 主要包含三个步骤：

1. XLogBeginInsert()：主要逻辑就是将 begininsert_called 设置为 true，XLogBeginInsert() 仅能被 call 一次，多次调用会报错 "XLogBeginInsert was already called"
2. XLogRegisterBlock()：注册需要写入的 WAL block，
3. XLogInsert()



## B Tree 的插入

```c
extern bool btinsert(Relation rel, Datum *values, bool *isnull,
					 ItemPointer ht_ctid, Relation heapRel,
					 IndexUniqueCheck checkUnique,
					 bool indexUnchanged,
					 struct IndexInfo *indexInfo);
```

## B Tree 的读取
