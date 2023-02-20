---
title: "[MySQL] Secondary Engine 源码阅读"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Summary
- Demo：初识 MySQL Secondary Engine
- MySQL Plugin 机制
- Secondary Engine DDL
- Secondary Engine 写流程
- Secondary Engine 读流程

## Demo：初步体验 MySQL Secondary Engine

## MySQL Plugin 机制

MySQL 有着强大的 Plugin 机制，比如审计日志、查询改写，链接管理等，在 MySQL 源码中的 plugin 目录中我们能看到非常多的 plugin 实现样例，而 Secondary Engine 也是通过 MySQL 的 Plugin 机制实现的。

实现一个插件，只需要使用 `mysql_declare_plugin` 这个宏，定义好插件的名字，以及相关接口的函数指针即可，比如下面要介绍的 mock secondary engine 插件是这样定义的：

```cpp
mysql_declare_plugin(mock){
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &mock_storage_engine,
    "MOCK",
    PLUGIN_AUTHOR_ORACLE,
    "Mock storage engine",
    PLUGIN_LICENSE_GPL,
    Init,
    nullptr,
    Deinit,
    0x0001,
    nullptr,
    nullptr,
    nullptr,
    0,
} mysql_declare_plugin_end;
```

- [ ] TODO：简要说明插件接口中几个关键函数的作用

MySQL 源码中有一个 mock 的 Secondary Engine 实现，它位于 storage/secondary_engine_mock/ha_mock.cc 中，从代码注释来看这个 mock 的 Secondary Engine 主要是用来方便测试使用：

```cpp
/**
 * The MOCK storage engine is used for testing MySQL server functionality
 * related to secondary storage engines.
 *
 * There are currently no secondary storage engines mature enough to be merged
 * into mysql-trunk. Therefore, this bare-minimum storage engine, with no
 * actual functionality and implementing only the absolutely necessary handler
 * interfaces to allow setting it as a secondary engine of a table, was created
 * to facilitate pushing MySQL server code changes to mysql-trunk with test
 * coverage without depending on ongoing work of other storage engines.
 *
 * @note This mock storage engine does not support being set as a primary
 * storage engine.
 */
```

后面的所有代码研究都会基于它进行。

## Secondary Engine DDL

### Create a secondary engine table

- 如何创建 secondary engine table，元数据如何存储
- 如何修改表结构
- 如何删除 secondary engine table

对于新表：

创建一个有 secondary_engine 的 table，secondary_engine 为 rapid（heatwave）：

```sql
CREATE TABLE orders (id INT) SECONDARY_ENGINE = RAPID;
```

实测的时候，如果没有 load secondary engine 的 plugin，建表语句也能成功，show create table 也能显示出该表拥有 secondary_engine 属性。

### Load the table into secondary engine

第一次数据同步需要手工触发：

```sql
ALTER TABLE orders SECONDARY_LOAD;
```

在没有 load secondary engine plugin 时上面的语句会执行报错：`ERROR 1286 (42000): Unknown storage engine 'RAPID'`，可以从这个错误入手分析 secondary engine 的数据导入过程。把所有 ER_UNKNOWN_STORAGE_ENGINE 打上断点，Debug 可以发现这个错误是是从这里抛出来的：

```gdb
(gdb) bt
#0  secondary_engine_load_table (thd=0x7f48c8001060, table=...) at /root/code/mysql-server/sql/sql_table.cc:2671
#1  0x0000564f61a6470c in Sql_cmd_secondary_load_unload::mysql_secondary_load_or_unload (this=0x7f48c801e7a8, thd=0x7f48c8001060, table_list=0x7f48c801e138) at /root/code/mysql-server/sql/sql_table.cc:11546
#2  0x0000564f62108321 in Sql_cmd_secondary_load_unload::execute (this=0x7f48c801e7a8, thd=0x7f48c8001060) at /root/code/mysql-server/sql/sql_alter.cc:430
#3  0x0000564f61982d33 in mysql_execute_command (thd=0x7f48c8001060, first_level=true) at /root/code/mysql-server/sql/sql_parse.cc:4677
#4  0x0000564f61985014 in dispatch_sql_command (thd=0x7f48c8001060, parser_state=0x7f49445f6a90) at /root/code/mysql-server/sql/sql_parse.cc:5312
#5  0x0000564f6197a99f in dispatch_command (thd=0x7f48c8001060, com_data=0x7f49445f73e0, command=COM_QUERY) at /root/code/mysql-server/sql/sql_parse.cc:2032
#6  0x0000564f61978a16 in do_command (thd=0x7f48c8001060) at /root/code/mysql-server/sql/sql_parse.cc:1435
#7  0x0000564f61bb7a9d in handle_connection (arg=0x564f69ba51d0) at /root/code/mysql-server/sql/conn_handler/connection_handler_per_thread.cc:302
#8  0x0000564f63e7c0b6 in pfs_spawn_thread (arg=0x564f69db1310) at /root/code/mysql-server/storage/perfschema/pfs.cc:2986
#9  0x00007f498bf26609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#10 0x00007f498baf9133 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

从上面的堆栈可以分析出：MySQL 为 secondary engine 的 alter table xxx secondary_load/secondary_unload 创建了一个 Sql_cmd_secondary_load_unload 的 cmd，通过执行这个 cmd 的 execute 函数完成 secondary engine 数据的 load 和 unload：
- #2: Sql_cmd_secondary_load_unload::execute：做基本的权限检查
- #1: Sql_cmd_secondary_load_unload::mysql_secondary_load_or_unload：设置 MDL、降级事务隔离级别为 RC 等，为数据 unload 到 secondary 做准备
- #0: secondary_engine_load_table：根据 secondary_engine 的名字（rapid）寻找对应的 plugin，plugin 需要也实现 “ha_resolve_by_name” 接口。上面报错的原因是根据 secondary engine 的名字找不到对应的 plugin，于是报错了 “ERROR 1286 (42000): Unknown storage engine 'RAPID'”，如果改成 “secondary engine plugin is not found” 之类的可能会更友好些。

如果找到了对应的 plugin，则将该 plugin 转换为对应的 `handlerton`，为需要 load 的表创建一个对应的 `handler`，最终调用该 handler 的 `load_table()` 将数据 load 到 secondary engine 上。secondary engine plugin 需要自己实现该接口：
```cpp
/**
 * Loads a table into its defined secondary storage engine.
 *
 * @param table Table opened in primary storage engine. Its read_set tells
 * which columns to load.
 *
 * @return 0 if success, error code otherwise.
 */
virtual int load_table(const TABLE &table [[maybe_unused]]) {
  /* purecov: begin inspected */
  assert(false);
  return HA_ERR_WRONG_COMMAND;
  /* purecov: end */
}
```
MySQL 代码中有一个 mock 的 secondary engine 实现，它位于 storage/secondary_engine_mock/ha_mock.cc 中，在这个 mock 的 secondary engine 实现中，`load_table()` 函数仅注册该表的表名，不会同步数据，应该也查不到数据了：

```cpp
int ha_mock::load_table(const TABLE &table_arg) {
  assert(table_arg.file != nullptr);
  loaded_tables->add(table_arg.s->db.str, table_arg.s->table_name.str);
  if (loaded_tables->get(table_arg.s->db.str, table_arg.s->table_name.str) ==
      nullptr) {
    my_error(ER_NO_SUCH_TABLE, MYF(0), table_arg.s->db.str,
             table_arg.s->table_name.str);
    return HA_ERR_KEY_NOT_FOUND;
  }
  return 0;
}
```

后续的数据在发生 DML 后会自动从 innodb 同步到 rapid 这个 secondary_engine 中

参考：
- [2.2.2 Loading Data Manually](https://dev.mysql.com/doc/heatwave/en/heatwave-loading-data-manually.html)

### 哪些 DDL 允许被执行？

secondary_engine_supports_ddl()

secondary engine 中
1. notify_exclusive_mdl()
2. notify_alter_table()
3. notify_rename_table()
4. notify_truncate_table()



## Secondary Engine 写流程

- primary engine（innodb）的事务读写如何同步到 secondary engine
- secondary engine 表现出来的事务隔离级别是什么

### Binlog 同步


## Secondary Engine 读流程

- 如何 secondary engine 中优化和执行 select statement
- 如何根据 cost 判断是采用 primary engine 执行还是 secondary engine 执行
- secondary engine 的执行结果如何返回给 mysql client

### prepare_secondary_engine()

函数的调用堆栈：

```gdb
#0  PrepareSecondaryEngine (thd=0x7ffac4001060, lex=0x7ffac4004420) at /root/code/mysql-server/storage/secondary_engine_mock/ha_mock.cc:230
#1  0x00005578c04ea89f in open_secondary_engine_tables (thd=0x7ffac4001060, flags=0) at /root/code/mysql-server/sql/sql_base.cc:6706
#2  0x00005578c04ea9af in open_tables_for_query (thd=0x7ffac4001060, tables=0x7ffac4ae01d8, flags=0) at /root/code/mysql-server/sql/sql_base.cc:6740
#3  0x00005578c0687599 in Sql_cmd_dml::prepare (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:360
#4  0x00005578c06880ed in Sql_cmd_dml::execute (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:534
#5  0x00005578c05fed33 in mysql_execute_command (thd=0x7ffac4001060, first_level=true) at /root/code/mysql-server/sql/sql_parse.cc:4677
#6  0x00005578c0601014 in dispatch_sql_command (thd=0x7ffac4001060, parser_state=0x7ffb406f7a90) at /root/code/mysql-server/sql/sql_parse.cc:5312
#7  0x00005578c05f699f in dispatch_command (thd=0x7ffac4001060, com_data=0x7ffb406f83e0, command=COM_QUERY) at /root/code/mysql-server/sql/sql_parse.cc:2032
#8  0x00005578c05f4a16 in do_command (thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_parse.cc:1435
#9  0x00005578c0833a9d in handle_connection (arg=0x5578c9ea2a90) at /root/code/mysql-server/sql/conn_handler/connection_handler_per_thread.cc:302
#10 0x00005578c2af80b6 in pfs_spawn_thread (arg=0x5578c9e900b0) at /root/code/mysql-server/storage/perfschema/pfs.cc:2986
#11 0x00007ffb88011609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#12 0x00007ffb87be4133 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

Prepare_secondary_engine 的函数签名：

```cpp
/**
  Prepare the secondary engine for executing a statement. This function is
  called right after the secondary engine TABLE objects have been opened by
  open_secondary_engine_tables(), before the statement is optimized and
  executed. Secondary engines will typically create a context object in this
  function, which they can use to store state that is needed during the
  optimization and execution phases.

  @param thd  thread context
  @param lex  the statement to execute
  @return true on error, false on success
*/
using prepare_secondary_engine_t = bool (*)(THD *thd, LEX *lex);
```

这个函数用来给在 secondary engine 中执行的 query 创建需要使用的 query context，然后通过当前 query 的 LEX 注册进去，后续就可以通过 LEX 访问这个 query context 了：

```cpp
/**
  Gets the secondary engine execution context for this statement.
*/
Secondary_engine_execution_context *secondary_engine_execution_context() const {
  return m_secondary_engine_context;
}

/**
  Sets the secondary engine execution context for this statement.
  The old context object is destroyed, if there is one. Can be set
  to nullptr to destroy the old context object and clear the
  pointer.

  The supplied context object should be allocated on the execution
  MEM_ROOT, so that its memory doesn't have to be manually freed
  after query execution.
*/
void set_secondary_engine_execution_context(Secondary_engine_execution_context *context);
```

MySQL 提供的默认实现在 `Secondary_engine_execution_context `中，没有包含任何数据，secondary engine 的 实现者进需要继承该 class 即可。

### optimize_secondary_engine()

函数调用堆栈：

```gdb
#0  OptimizeSecondaryEngine (thd=0x7ffac4001060, lex=0x7ffac4004420) at /root/code/mysql-server/storage/secondary_engine_mock/ha_mock.cc:280
#1  0x00005578c0688eb5 in optimize_secondary_engine (thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:757
#2  0x00005578c0688f36 in Sql_cmd_dml::execute_inner (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:780
#3  0x00005578c0688486 in Sql_cmd_dml::execute (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:587
#4  0x00005578c05fed33 in mysql_execute_command (thd=0x7ffac4001060, first_level=true) at /root/code/mysql-server/sql/sql_parse.cc:4677
#5  0x00005578c0601014 in dispatch_sql_command (thd=0x7ffac4001060, parser_state=0x7ffb406f7a90) at /root/code/mysql-server/sql/sql_parse.cc:5312
#6  0x00005578c05f699f in dispatch_command (thd=0x7ffac4001060, com_data=0x7ffb406f83e0, command=COM_QUERY) at /root/code/mysql-server/sql/sql_parse.cc:2032
#7  0x00005578c05f4a16 in do_command (thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_parse.cc:1435
#8  0x00005578c0833a9d in handle_connection (arg=0x5578c9ea2a90) at /root/code/mysql-server/sql/conn_handler/connection_handler_per_thread.cc:302
#9  0x00005578c2af80b6 in pfs_spawn_thread (arg=0x5578c9e900b0) at /root/code/mysql-server/storage/perfschema/pfs.cc:2986
#10 0x00007ffb88011609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#11 0x00007ffb87be4133 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```


函数签名：

```C++
/**
  Optimize a statement for execution on a secondary storage engine. This
  function is called when the optimization of a statement has completed, just
  before the statement is executed. Secondary engines can use this function to
  apply engine-specific optimizations to the execution plan. They can also
  reject executing the query by raising an error, in which case the query will
  be reprepared and executed by the primary storage engine.

  @param thd  thread context
  @param lex  the statement being optimized
  @return true on error, false on success
*/
using optimize_secondary_engine_t = bool (*)(THD *thd, LEX *lex);
```

调用的入口在一开始我们看到的执行框架 `Sql_cmd_dml::execute_inner(THD *thd)` 里面：

```C++
bool optimize_secondary_engine(THD *thd) {
  if (retry_with_secondary_engine(thd)) {
    thd->get_stmt_da()->reset_diagnostics_area();
    thd->get_stmt_da()->set_error_status(thd, ER_PREPARE_FOR_SECONDARY_ENGINE);
    return true;
  }

  if (thd->secondary_engine_optimization() ==
          Secondary_engine_optimization::PRIMARY_TENTATIVELY &&
      thd->lex->m_sql_cmd != nullptr &&
      thd->lex->m_sql_cmd->is_optional_transform_prepared()) {
    // A previous preparation did a secondary engine specific transform,
    // and this transform wasn't requested for the primary engine (in
    // 'optimizer_switch'), so in this new execution we need to reprepare for
    // the primary engine without the optional transform, for likely better
    // performance.
    thd->lex->m_sql_cmd->set_optional_transform_prepared(false);
    thd->get_stmt_da()->reset_diagnostics_area();
    thd->get_stmt_da()->set_error_status(thd, ER_PREPARE_FOR_PRIMARY_ENGINE);
    return true;
  }

  const handlerton *secondary_engine = thd->lex->m_sql_cmd->secondary_engine();
  return secondary_engine != nullptr &&
         secondary_engine->optimize_secondary_engine != nullptr &&
         secondary_engine->optimize_secondary_engine(thd, thd->lex);
}
```

关键问题：

1. 怎么注册一个 secondary engine 和它对应的 optimize_secondary_engine_t() 函数实现？
2. SQL 执行时如何处理 secondary engine，计算是否可以下推？

### compare_secondary_engine_cost()

函数调用堆栈：

```gdb
#0  CompareJoinCost (thd=0x7ffac4001060, join=..., optimizer_cost=0.59999999999999998, use_best_so_far=0x7ffb406f5c74, cheaper=0x7ffb406f55b9, secondary_engine_cost=0x7ffb406f55c8) at /root/code/mysql-server/storage/secondary_engine_mock/ha_mock.cc:304
#1  0x00005578c061fb89 in Optimize_table_order::consider_plan (this=0x7ffb406f5c40, idx=0, trace_obj=0x7ffb406f56d0) at /root/code/mysql-server/sql/sql_planner.cc:2537
#2  0x00005578c0620a46 in Optimize_table_order::best_extension_by_limited_search (this=0x7ffb406f5c40, remaining_tables=1, idx=0, current_search_depth=62) at /root/code/mysql-server/sql/sql_planner.cc:2908
#3  0x00005578c061f0b5 in Optimize_table_order::greedy_search (this=0x7ffb406f5c40, remaining_tables=1) at /root/code/mysql-server/sql/sql_planner.cc:2347
#4  0x00005578c061e789 in Optimize_table_order::choose_table_order (this=0x7ffb406f5c40) at /root/code/mysql-server/sql/sql_planner.cc:2023
#5  0x00005578c05d3777 in JOIN::make_join_plan (this=0x7ffac4b01920) at /root/code/mysql-server/sql/sql_optimizer.cc:5271
#6  0x00005578c05c5db1 in JOIN::optimize (this=0x7ffac4b01920, finalize_access_paths=true) at /root/code/mysql-server/sql/sql_optimizer.cc:636
#7  0x00005578c068b781 in Query_block::optimize (this=0x7ffac4adeeb8, thd=0x7ffac4001060, finalize_access_paths=true) at /root/code/mysql-server/sql/sql_select.cc:1822
#8  0x00005578c0742d90 in Query_expression::optimize (this=0x7ffac4adedd0, thd=0x7ffac4001060, materialize_destination=0x0, create_iterators=true, finalize_access_paths=true) at /root/code/mysql-server/sql/sql_union.cc:1003
#9  0x00005578c0688f0c in Sql_cmd_dml::execute_inner (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:772
#10 0x00005578c0688486 in Sql_cmd_dml::execute (this=0x7ffac4b015d8, thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_select.cc:587
#11 0x00005578c05fed33 in mysql_execute_command (thd=0x7ffac4001060, first_level=true) at /root/code/mysql-server/sql/sql_parse.cc:4677
#12 0x00005578c0601014 in dispatch_sql_command (thd=0x7ffac4001060, parser_state=0x7ffb406f7a90) at /root/code/mysql-server/sql/sql_parse.cc:5312
#13 0x00005578c05f699f in dispatch_command (thd=0x7ffac4001060, com_data=0x7ffb406f83e0, command=COM_QUERY) at /root/code/mysql-server/sql/sql_parse.cc:2032
#14 0x00005578c05f4a16 in do_command (thd=0x7ffac4001060) at /root/code/mysql-server/sql/sql_parse.cc:1435
#15 0x00005578c0833a9d in handle_connection (arg=0x5578c9ea2a90) at /root/code/mysql-server/sql/conn_handler/connection_handler_per_thread.cc:302
#16 0x00005578c2af80b6 in pfs_spawn_thread (arg=0x5578c9e900b0) at /root/code/mysql-server/storage/perfschema/pfs.cc:2986
#17 0x00007ffb88011609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#18 0x00007ffb87be4133 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

这个函数调用栈比较深，看起来是 mysql 对 secondary_engine 的 table 做完优化后，调用这个函数来向 secondary_engine 获取该 plan 的 cost。写了几种类型的测试后发现只有在当前 JOIN 的所有 table 都是 secondary_engine 表时才会使用该函数估算 cost。mysql 目前是通过 `secondary_engine_cost_threshold` 来判断 secondary_engine 上的表能否在 secondary_engine 上执行。看起来这个估算出来的 cost 就是用来和这个变量进行对比了。

- [ ] TODO：验证这个 cost 和 `secondary_engine_cost_threshold` 的关系

另外我写了一个两表 join 的查询，其中一个表是 secondary_engine 表，另一个是普通 innodb 表，测试下来他是不能走到这个 cost estimation 函数的，那么：

- [ ] TODO：看起来一条 Query 要么全部走 secondary engine，要么全部走 innodb，不能像 TiDB 一样一个 subplan 走 TiFlash 一个 subplan 走 TiKV？需要从代码中验证一下


### secondary_engine_modify_access_path_cost()



### Integrate with Hypergraph join optimizer

secondary_engine_flags 定义了 secondary engine 中支持的算子：

```cpp
/// Bitmap which contains the supported join types and other flags
/// for a secondary storage engine when used with the hypergraph join
/// optimizer. If it is empty, it means that the secondary engine
/// does not support the hypergraph join optimizer.
SecondaryEngineFlags secondary_engine_flags;

// Capabilities (bit flags) for secondary engines.
using SecondaryEngineFlags = uint64_t;
enum class SecondaryEngineFlag : SecondaryEngineFlags {
  SUPPORTS_HASH_JOIN = 0,
  SUPPORTS_NESTED_LOOP_JOIN = 1,

  // If this flag is set, aggregation (GROUP BY and DISTINCT) do not require
  // ordered inputs and create unordered outputs. This is typically the case
  // if they are implemented using hash-based techniques.
  AGGREGATION_IS_UNORDERED = 2
};
```


### Execute query on secondary engine

### Collect results from secondary engine

## Secondary Engine 故障诊断

- secondary engine 有哪些系统变量、系统表，分别什么含义，如何维护
- secondary engine 如何排查慢查询慢在哪
- secondary engine 是否支持 explain analyze，是否支持 trace


## References

- [MySQL HeatWave User Guide](https://dev.mysql.com/doc/heatwave/en/heatwave-introduction.html)
- [MySQL · 源码阅读 · Secondary Engine](http://mysql.taobao.org/monthly/2020/11/04/)

## Appendix I: Build and run MySQL

Build and install MySQL:

```cpp
git clone --recursive https://github.com/mysql/mysql-server.git
cd mysql-server

cmake -B build -S . \
      -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=build/install \
      -DDOWNLOAD_BOOST=1 -DWITH_BOOST=. \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=1
      
cmake --build build -j `nproc`
cmake --install build
```

Create a config file, for example, `my.cnf`:

```txt
[mysqld]
basedir=/tmp/my-instance
datadir=/tmp/my-instance/data
```

Initialize MySQL storage in the first run:

```sh
rm -rf /tmp/my-instance
mkdir -p /tmp/my-instance

build/install/bin/mysqld --defaults-file=my.cnf --user=root \
                          --initialize-insecure \
```

Start MySQL instance after initialization:

```sh
build/install/bin/mysqld --defaults-file=my.cnf --user=root
```

Connect to the server:

```sh
build/install/bin/mysql -u root
```
