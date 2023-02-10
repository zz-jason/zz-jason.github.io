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
2. XLogRegisterBlock()：注册需要写入的 WAL block，将需要记录 WAL 的 page 注册在 `registered_buffers[0]` 内。对于这个新 page 来说，它的所有信息：page 的完整内容、block number、所属的 relation 文件信息都可以通过注册的 `registered_buffers[0]` 获取到。

```c
void
XLogRegisterBlock(uint8 block_id, RelFileLocator *rlocator, ForkNumber forknum,
				  BlockNumber blknum, Page page, uint8 flags)
{
	registered_buffer *regbuf;

	Assert(begininsert_called);

	if (block_id >= max_registered_block_id)
		max_registered_block_id = block_id + 1;

	if (block_id >= max_registered_buffers)
		elog(ERROR, "too many registered buffers");

	regbuf = &registered_buffers[block_id];

	regbuf->rlocator = *rlocator;
	regbuf->forkno = forknum;
	regbuf->block = blknum;
	regbuf->page = page;
	regbuf->flags = flags;
	regbuf->rdata_tail = (XLogRecData *) &regbuf->rdata_head;
	regbuf->rdata_len = 0;

	regbuf->in_use = true;
}
```

3.  XLogInsert()：负责 WAL 的写入。这个函数比较长，最关键的部分是两个步骤：
    1. 通过 `XLogRecordAssemble()` 将刚才注册的 `registered_buffers[0]` 构造成 `XLogRecData` 链表。
    2. 通过 `XLogInsertRecord()`写入构造好的 `XLogRecData` 链表。



`XLogRecordAssemble()`



`XLogInsertRecord()`

## B Tree 的插入

```c
extern bool btinsert(Relation rel, Datum *values, bool *isnull,
					 ItemPointer ht_ctid, Relation heapRel,
					 IndexUniqueCheck checkUnique,
					 bool indexUnchanged,
					 struct IndexInfo *indexInfo);
```

## B Tree 的读取



## B Tree WAL 日志

B Tree 的 WAL 类型定义在 src/include/access/nbtxlog.h 中，它们代表了所有 B Tree 操作过程中会记录的 WAL 日志类型：

```c
/*
 * XLOG records for btree operations
 *
 * XLOG allows to store some information in high 4 bits of log record xl_info field
 */
#define XLOG_BTREE_INSERT_LEAF        0x00 /* add index tuple without split */
#define XLOG_BTREE_INSERT_UPPER       0x10 /* same, on a non-leaf page */
#define XLOG_BTREE_INSERT_META        0x20 /* same, plus update metapage */
#define XLOG_BTREE_SPLIT_L            0x30 /* add index tuple with split */
#define XLOG_BTREE_SPLIT_R            0x40 /* as above, new item on right */
#define XLOG_BTREE_INSERT_POST        0x50 /* add index tuple with posting split */
#define XLOG_BTREE_DEDUP              0x60 /* deduplicate tuples for a page */
#define XLOG_BTREE_DELETE             0x70 /* delete leaf index tuples for a page */
#define XLOG_BTREE_UNLINK_PAGE        0x80 /* delete a half-dead page */
#define XLOG_BTREE_UNLINK_PAGE_META   0x90 /* same, and update metapage */
#define XLOG_BTREE_NEWROOT            0xA0 /* new root page */
#define XLOG_BTREE_MARK_PAGE_HALFDEAD 0xB0 /* mark a leaf as half-dead */
#define XLOG_BTREE_VACUUM             0xC0 /* delete entries on a page during vacuum */
#define XLOG_BTREE_REUSE_PAGE         0xD0 /* old page is about to be reused from FSM */
#define XLOG_BTREE_META_CLEANUP       0xE0 /* update cleanup-related data in the metapage */
```

编译 PG 时加上 CFLAGS='-DWAL_DEBUG' 打开 PostgreSQL 的 WAL Debug 功能，启动 PG，创建一个带有 B Tree index 的表，插入数据即可看到 PostgreSQL 日志中打印的 WAL Debug 信息。仅看 B Tree 的 WAL，部分日志如下：

```txt
984:1973:2131:2023-02-08 23:02:31.356 CST [947945] LOG:  INSERT @ 0/156B8D0:  - Btree/INSERT_LEAF: off 349
986:1977:2135:2023-02-08 23:02:31.356 CST [947945] LOG:  INSERT @ 0/156B958:  - Btree/INSERT_LEAF: off 238
988:1981:2139:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156B9E0:  - Btree/INSERT_LEAF: off 223
990:1985:2143:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156BA68:  - Btree/INSERT_LEAF: off 142
992:1989:2147:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156BAF0:  - Btree/INSERT_LEAF: off 91
994:1993:2151:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156BB78:  - Btree/INSERT_LEAF: off 260
996:1997:2155:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156BBF8:  - Btree/DEDUP: nintervals 2
997:1999:2157:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156C948:  - Btree/SPLIT_L: level 0, firstrightoff 198, newitemoff 191, postingoff 0
998:2001:2159:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156C990:  - Btree/INSERT_UPPER: off 2
1000:2005:2163:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CA18:  - Btree/INSERT_LEAF: off 48
1002:2009:2167:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CAA0:  - Btree/INSERT_LEAF: off 134
1004:2013:2171:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CB28:  - Btree/INSERT_LEAF: off 193
1006:2017:2175:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CBB0:  - Btree/INSERT_LEAF: off 92
1008:2021:2179:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CC38:  - Btree/INSERT_LEAF: off 63
1010:2025:2183:2023-02-08 23:02:31.357 CST [947945] LOG:  INSERT @ 0/156CCC0:  - Btree/INSERT_LEAF: off 58
```

### XLOG_BTREE_INSERT_LEAF

这是最常见的 B Tree WAL 日志，大部分日志内容都是这个。XLOG_BTREE_INSERT_LEAF 日志只有在 `_bt_insertonpg()` 函数中会出现：

```c
static void _bt_insertonpg(Relation rel,
                           BTScanInsert itup_key,
                           Buffer buf,
                           Buffer cbuf,
                           BTStack stack,
                           IndexTuple itup,
                           Size itemsz,
                           OffsetNumber newitemoff,
                           int postingoff,
                           bool split_only_page)
```

```c
if (isleaf && postingoff == 0) {
  /* Simple leaf insert */
  xlinfo = XLOG_BTREE_INSERT_LEAF;
} else { ... }
XLogRegisterBuffer(0, buf, REGBUF_STANDARD);
if (postingoff == 0) {
  /* Just log itup from caller */
  XLogRegisterBufData(0, (char*)itup, IndexTupleSize(itup));
} else { ... }

recptr = XLogInsert(RM_BTREE_ID, xlinfo);

if (BufferIsValid(metabuf))
  PageSetLSN(metapg, recptr);
if (!isleaf)
  PageSetLSN(BufferGetPage(cbuf), recptr);

PageSetLSN(page, recptr);

```

XLOG_BTREE_INSERT_LEAF 日志的写入分为如下几步：

1. xlinfo = XLOG_BTREE_INSERT_LEAF：设置 xlinfo 为 XLOG_BTREE_INSERT_LEAF
2. XLogRegisterBuffer(0, buf, REGBUF_STANDARD)：注册 WAL buffer 用于写入
3. XLogRegisterBufData(0, (char*)itup, IndexTupleSize(itup))：注册 itup 用于写入
4. recptr = XLogInsert(RM_BTREE_ID, xlinfo)：写入 WAL，记录 LSN 到 recptr 中。
5. 为相关 page 设置 LSN

