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
static void _bt_insertonpg(
    Relation     rel,
    BTScanInsert itup_key,
    Buffer       buf,
    Buffer       cbuf,
    BTStack      stack,
    IndexTuple   itup,
    Size         itemsz,
    OffsetNumber newitemoff,
    int          postingoff,
    bool         split_only_page
) {
```

```c
XLogBeginInsert();
XLogRegisterData((char*)&xlrec, SizeOfBtreeInsert);

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



我们在这里加个断点，看看被 XLogRegisterXXX 的 buf 和 itup 是从哪里构造出来的：

```gdb
(gdb) bt
#0  _bt_insertonpg (rel=0x7f07bf40d938, itup_key=0x561b37cdfc10, buf=277, cbuf=0, stack=0x561b37cdfc98, itup=0x561b37cefb00, itemsz=16, newitemoff=94, postingoff=0, split_only_page=false) at ../src/backend/access/nbtree/nbtinsert.c:1233
#1  0x0000561b36ca3cab in _bt_doinsert (rel=0x7f07bf40d938, itup=0x561b37cefb00, checkUnique=UNIQUE_CHECK_NO, indexUnchanged=false, heapRel=0x7f07bf419ba0) at ../src/backend/access/nbtree/nbtinsert.c:264
#2  0x0000561b36cace4a in btinsert (rel=0x7f07bf40d938, values=0x7ffff4b93890, isnull=0x7ffff4b93870, ht_ctid=0x561b37cdf8f8, heapRel=0x7f07bf419ba0, checkUnique=UNIQUE_CHECK_NO, indexUnchanged=false, indexInfo=0x561b37cdfa80) at ../src/backend/access/nbtree/nbtree.c:191
#3  0x0000561b36c9fe9d in index_insert (indexRelation=0x7f07bf40d938, values=0x7ffff4b93890, isnull=0x7ffff4b93870, heap_t_ctid=0x561b37cdf8f8, heapRelation=0x7f07bf419ba0, checkUnique=UNIQUE_CHECK_NO, indexUnchanged=false, indexInfo=0x561b37cdfa80) at ../src/backend/access/index/indexam.c:193
#4  0x0000561b36e48eef in ExecInsertIndexTuples (resultRelInfo=0x561b37cee080, slot=0x561b37cdf8c8, estate=0x561b37cedc20, update=false, noDupErr=false, specConflict=0x0, arbiterIndexes=0x0) at ../src/backend/executor/execIndexing.c:416
#5  0x0000561b36e8d9c5 in ExecInsert (context=0x7ffff4b93b40, resultRelInfo=0x561b37cee080, slot=0x561b37cdf8c8, canSetTag=true, inserted_tuple=0x0, insert_destrel=0x0) at ../src/backend/executor/nodeModifyTable.c:1146
#6  0x0000561b36e919f0 in ExecModifyTable (pstate=0x561b37cede78) at ../src/backend/executor/nodeModifyTable.c:3814
#7  0x0000561b36e564b3 in ExecProcNodeFirst (node=0x561b37cede78) at ../src/backend/executor/execProcnode.c:464
#8  0x0000561b36e4a9e2 in ExecProcNode (node=0x561b37cede78) at ../src/include/executor/executor.h:262
#9  0x0000561b36e4d229 in ExecutePlan (estate=0x561b37cedc20, planstate=0x561b37cede78, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, dest=0x561b37cf0a18, execute_once=true) at ../src/backend/executor/execMain.c:1633
#10 0x0000561b36e4af73 in standard_ExecutorRun (queryDesc=0x561b37cfce70, direction=ForwardScanDirection, count=0, execute_once=true) at ../src/backend/executor/execMain.c:364
#11 0x0000561b36e4ae03 in ExecutorRun (queryDesc=0x561b37cfce70, direction=ForwardScanDirection, count=0, execute_once=true) at ../src/backend/executor/execMain.c:308
#12 0x0000561b3708a9f1 in ProcessQuery (plan=0x561b37cf08c8, sourceText=0x561b37c16530 "insert into t select floor(random() * 10000 + 1)::int, floor(random() * 10000 + 1)::int from t;", params=0x0, queryEnv=0x0, dest=0x561b37cf0a18, qc=0x7ffff4b93fa0) at ../src/backend/tcop/pquery.c:160
#13 0x0000561b3708c3fd in PortalRunMulti (portal=0x561b37c912f0, isTopLevel=true, setHoldSnapshot=false, dest=0x561b37cf0a18, altdest=0x561b37cf0a18, qc=0x7ffff4b93fa0) at ../src/backend/tcop/pquery.c:1277
#14 0x0000561b3708b96e in PortalRun (portal=0x561b37c912f0, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x561b37cf0a18, altdest=0x561b37cf0a18, qc=0x7ffff4b93fa0) at ../src/backend/tcop/pquery.c:791
#15 0x0000561b37084bef in exec_simple_query (query_string=0x561b37c16530 "insert into t select floor(random() * 10000 + 1)::int, floor(random() * 10000 + 1)::int from t;") at ../src/backend/tcop/postgres.c:1237
#16 0x0000561b3708998a in PostgresMain (dbname=0x561b37c140f0 "template1", username=0x561b37c4f0c0 "root") at ../src/backend/tcop/postgres.c:4565
#17 0x0000561b36fcb23f in BackendRun (port=0x561b37c41ad0) at ../src/backend/postmaster/postmaster.c:4461
#18 0x0000561b36fcaaeb in BackendStartup (port=0x561b37c41ad0) at ../src/backend/postmaster/postmaster.c:4189
#19 0x0000561b36fc71f8 in ServerLoop () at ../src/backend/postmaster/postmaster.c:1779
#20 0x0000561b36fc6aad in PostmasterMain (argc=3, argv=0x561b37c120a0) at ../src/backend/postmaster/postmaster.c:1463
#21 0x0000561b36ecbb7f in main (argc=3, argv=0x561b37c120a0) at ../src/backend/main/main.c:196
```



从函数调用栈可以发现，itup 是在 `btinsert()` 中根据用户插入的一行数据 `values` 构造出来的，后续调用 `_bt_doinsert()` 将插入 itup 插入到该索引所属的 B Tree 中，完成插入后，itup 也在这里被销毁，释放内存：

```c
bool btinsert(
    Relation         rel,
    Datum*           values,
    bool*            isnull,
    ItemPointer      ht_ctid,
    Relation         heapRel,
    IndexUniqueCheck checkUnique,
    bool             indexUnchanged,
    IndexInfo*       indexInfo
) {
  bool       result;
  IndexTuple itup;

  /* generate an index tuple */
  itup = index_form_tuple(RelationGetDescr(rel), values, isnull);
  itup->t_tid = *ht_ctid;

  result = _bt_doinsert(rel, itup, checkUnique, indexUnchanged, heapRel);
  pfree(itup);
  return result;
}
```



而 XLogRegisterBuffer(0, buf, REGBUF_STANDARD) 中的 buf 代表了需要该 B Tree 叶子结点所在的 buffer id，（它实际上是个整数类型），在 src/backend/access/transam/README 中我们能找到更多关于这个函数的作用说明：

```
void XLogRegisterBuffer(uint8 block_id, Buffer buf, uint8 flags);

    XLogRegisterBuffer adds information about a data block to the WAL record.
    block_id is an arbitrary number used to identify this page reference in
    the redo routine.  The information needed to re-find the page at redo -
    relfilelocator, fork, and block number - are included in the WAL record.

    XLogInsert will automatically include a full copy of the page contents, if
    this is the first modification of the buffer since the last checkpoint.
    It is important to register every buffer modified by the action with
    XLogRegisterBuffer, to avoid torn-page hazards.

    The flags control when and how the buffer contents are included in the
    WAL record.  Normally, a full-page image is taken only if the page has not
    been modified since the last checkpoint, and only if full_page_writes=on
    or an online backup is in progress.  The REGBUF_FORCE_IMAGE flag can be
    used to force a full-page image to always be included; that is useful
    e.g. for an operation that rewrites most of the page, so that tracking the
    details is not worth it.  For the rare case where it is not necessary to
    protect from torn pages, REGBUF_NO_IMAGE flag can be used to suppress
    full page image from being taken.  REGBUF_WILL_INIT also suppresses a full
    page image, but the redo routine must re-generate the page from scratch,
    without looking at the old page contents.  Re-initializing the page
    protects from torn page hazards like a full page image does.

    The REGBUF_STANDARD flag can be specified together with the other flags to
    indicate that the page follows the standard page layout.  It causes the
    area between pd_lower and pd_upper to be left out from the image, reducing
    WAL volume.

    If the REGBUF_KEEP_DATA flag is given, any per-buffer data registered with
    XLogRegisterBufData() is included in the WAL record even if a full-page
    image is taken.
```

以及 XLogRegisterBufData 的详细说明：

```
void XLogRegisterBufData(uint8 block_id, char *data, int len);

    XLogRegisterBufData is used to include data associated with a particular
    buffer that was registered earlier with XLogRegisterBuffer().  If
    XLogRegisterBufData() is called multiple times with the same block ID, the
    data are appended, and will be made available to the redo routine as one
    contiguous chunk.

    If a full-page image of the buffer is taken at insertion, the data is not
    included in the WAL record, unless the REGBUF_KEEP_DATA flag is used.
```

简单来说，如果 buf 所代表的 B Tree 节点是第一次被修改，Postgres 会把该节点的整个 page 都写到 WAL 中，通过 XLogRegisterBufData 注册的数据会被忽略（因为已经它们对 page 的修改包含在整个 page 中了）。如果该 page 不是 checkpoint 后的第一次修改，那么只有后面通过 XLogRegisterBufData() 注册的索引数据才会被写入到 WAL 中。这些逻辑判断都发生在 `XLogInsert()` 中。

构造和写入 WAL 通常包含如下几步：

```
Constructing a WAL record
-------------------------

A WAL record consists of a header common to all WAL record types,
record-specific data, and information about the data blocks modified.  Each
modified data block is identified by an ID number, and can optionally have
more record-specific data associated with the block.  If XLogInsert decides
that a full-page image of a block needs to be taken, the data associated
with that block is not included.

The API for constructing a WAL record consists of five functions:
XLogBeginInsert, XLogRegisterBuffer, XLogRegisterData, XLogRegisterBufData,
and XLogInsert.  First, call XLogBeginInsert().  Then register all the buffers
modified, and data needed to replay the changes, using XLogRegister*
functions.  Finally, insert the constructed record to the WAL by calling
XLogInsert().

	XLogBeginInsert();

	/* register buffers modified as part of this WAL-logged action */
	XLogRegisterBuffer(0, lbuffer, REGBUF_STANDARD);
	XLogRegisterBuffer(1, rbuffer, REGBUF_STANDARD);

	/* register data that is always included in the WAL record */
	XLogRegisterData(&xlrec, SizeOfFictionalAction);

	/*
	 * register data associated with a buffer. This will not be included
	 * in the record if a full-page image is taken.
	 */
	XLogRegisterBufData(0, tuple->data, tuple->len);

	/* more data associated with the buffer */
	XLogRegisterBufData(0, data2, len2);

	/*
	 * Ok, all the data and buffers to include in the WAL record have
	 * been registered. Insert the record.
	 */
	recptr = XLogInsert(RM_FOO_ID, XLOG_FOOBAR_DO_STUFF);
```

## B Tree Checkpoint



## 附录  1: build, install, and debug Postgres

```sh
cd $POSTGRES_HOME

CFLAGS='-DWAL_DEBUG' meson setup --prefix=`pwd`/build/install --buildtype=debug build .
meson compile -C build -j `nproc`
meson install -C build
```

`meson ` 会默认在 build 目录生成 `compile_commands.json` 文件，可以用来配置 vscode c/c++ configuration 的 "compileCommands"，方便代码跳转。



启动和连接 postgres：

```sh
$POSTGRES_HOME/build/install/bin/postgres -D /tmp/pg-test/ > pg.out 2>&1
$POSTGRES_HOME/build/install/bin/psql -d template1
```

