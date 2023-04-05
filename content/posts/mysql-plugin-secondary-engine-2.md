---
title: "[MySQL] Secondary Engine 源码阅读 2：Secondary Engine 读流程"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Summary

1. Overview of MySQL query procession
2. The prepare stage
3. The optimize stage
4. The execute stage

## 1. Overview of MySQL query processing

MySQL 的查询优化和处理由 `Sql_cmd_dml::execute(THD *thd)` 函数驱动，MySQL 在这个函数中准备执行需要的上下文信息，通过 `prepare()` 和 `execute_inner()` 这两个函数完成查询的 prepare、optimize 和 execute 工作，最后再清理当前查询执行结束后的上下文状态，收尾结束当前查询的执行。

读流程的函数调用关系：
- Sql_cmd_dml::execute()
	- Sql_cmd_dml::prepare()
		- **handlerton::prepare_secondary_engine()**
	- Sql_cmd_dml::execute_inner()
		- JOIN::optimize()
			- FindBestQueryPlan
				- **handlerton::secondary_engine_modify_access_path_cost()**
			- **handlerton::compare_secondary_engine_cost()**
		- optimize_secondary_engine()
			- **handlerton::optimize_secondary_engine()**
		- Query_expression::execute()
			- Query_expression::ExecuteIteratorQuery()
				- **JOIN::override_executor_func()**
	
### Sql_cmd_dml::execute()

SQL 的执行最外层在 `Sql_cmd_dml::execute(THD *thd)` 中，这里面会进行执行前的上下文准备，调用相关函数完成查询的执行，清理 SQL 执行完后的上下文信息，这篇文章中我们重点关注下面 2 个部分：
1. prepare(thd)：为后面的 SQL 优化与执行准备必要的信息，也包括 open 需要的 table
2. execute_inner(thd)：完成 SQL 的优化与执行

> From new bing:
>
> [Sql_cmd_dml是一个抽象类，它表示所有支持的数据操作（DML）语句的基类](https://dev.mysql.com/worklog/task/?id=5094)[1](https://dev.mysql.com/worklog/task/?id=5094)[。Sql_cmd_dml::execute()函数是一个纯虚函数，它的主要功能是执行DML语句，并检查用户是否具有相关的权限](https://dev.mysql.com/doc/dev/mysql-server/8.0.25/classSql__cmd__dml.html)[2](https://dev.mysql.com/doc/dev/mysql-server/8.0.25/classSql__cmd__dml.html)。
>
> [不同类型的DML语句（如INSERT，UPDATE，DELETE等）都有自己的子类，它们重写了Sql_cmd_dml::execute()函数来实现具体的逻辑。例如，Sql_cmd_insert_select::execute()函数会执行插入选择（INSERT … SELECT）语句，并调用mysql_insert_select()函数来完成操作](https://dev.mysql.com/worklog/task/?id=5094)[1](https://dev.mysql.com/worklog/task/?id=5094)。

精简后的代码如下：

```cpp
bool Sql_cmd_dml::execute(THD *thd) {
  ...
  if (!is_prepared()) {
    if (prepare(thd)) goto err;
  } else {
    ...
  }
  ...
  // Perform statement-specific execution
  if (execute_inner(thd)) goto err;
```

## 2. The prepare stage

### Sql_cmd_dml::prepare() && prepare_secondary_engine()

`Sql_cmd_dml::prepare()` 的主要作用是进行查询优化与计算前的初始化操作，比如权限检查、准备 DML、扩展视图等，精简后的代码如下：

```cpp
bool Sql_cmd_dml::prepare(THD *thd) {
  ...
  if (precheck(thd)) goto err;
  
  ...
  if (open_tables_for_query(
          thd, lex->query_tables,
          needs_explicit_preparation() ? MYSQL_OPEN_FORCE_SHARED_MDL : 0)) {
  }
  
  ...
  if (lex->set_var_list.elements && resolve_var_assignments(thd, lex))
    goto err; /* purecov: inspected */
  
  ...
  {
    ...
    if (prepare_inner(thd)) goto err;
  }
```

Secondary Engine 上的准备工作发生在 `open_tables_for_query()` 里面，这里 MySQL 会初始化 DML，初始化当前查询涉及到的 table 相关变量和对象：

> From new bing:
>
> [open_tables_for_query是一个内部函数，它的主要功能是打开一个查询所涉及的所有表，并将它们放入表缓存中](https://dev.mysql.com/doc/refman/8.0/en/show-open-tables.html)[1](https://dev.mysql.com/doc/refman/8.0/en/show-open-tables.html)[。表缓存是一个用于存储已经打开的非临时表的数据结构，它可以提高查询性能，避免重复打开和关闭表](https://www.cyberciti.biz/faq/installing-mysql-server-on-ubuntu-22-04-lts-linux/)[2](https://www.cyberciti.biz/faq/installing-mysql-server-on-ubuntu-22-04-lts-linux/)。
> 
> [open_tables_for_query函数会根据查询的语法树（LEX对象）来确定需要打开哪些表，并调用open_table()函数来逐个打开它们](https://dev.mysql.com/doc/refman/8.0/en/table-cache.html)[3](https://dev.mysql.com/doc/refman/8.0/en/table-cache.html)[。如果遇到错误或者超过了最大文件描述符限制，函数会返回失败](https://www.cyberciti.biz/faq/installing-mysql-server-on-ubuntu-22-04-lts-linux/)[2](https://www.cyberciti.biz/faq/installing-mysql-server-on-ubuntu-22-04-lts-linux/)。
>
> [如果您想要查看当前在表缓存中的表，您可以使用SHOW OPEN TABLES命令，并指定一个数据库名或者一个模式匹配](https://dev.mysql.com/doc/refman/8.0/en/show-open-tables.html)[1](https://dev.mysql.com/doc/refman/8.0/en/show-open-tables.html)。例如：
> 
> ```sql
> SHOW OPEN TABLES FROM test LIKE 't%';
> ```
>
> 这个例子中，会显示test数据库中以t开头的所有已经打开的表。


```cpp
bool open_tables_for_query(THD *thd, TABLE_LIST *tables, uint flags) {
  ...
  if (open_tables(thd, &tables, &thd->lex->table_count, flags,
                  &prelocking_strategy))
    goto end;

  if (open_secondary_engine_tables(thd, flags)) goto end;

  return false;
  ...
}
```

在 `open_secondary_engine_tables()` 中检查当前 query 是否能在 Secondary Engine 中执行，如果能的话则设置当前 query 的 Secondary Engine handle（也就是用户实现的 Secondary Engine 插件），将 Primary Engine 中的 TABLE 对象替换为 Secondary Engine 的对象，并且通过 Secondary Engine Plugin 为当前 Query 准备对应的 Secondary Engine Context：

> From new bing:
>
>[open_secondary_engine_tables是一个内部函数，它的主要功能是打开一个查询所涉及的所有次级存储引擎（secondary engine）的表，并将它们放入表缓存中](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-prepared-statements-instances-table.html)[1](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-prepared-statements-instances-table.html)[。次级存储引擎是一种可以与主要存储引擎（primary engine）协同工作的存储引擎，用于提供更高效的查询处理能力](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-extensions-table.html)[2](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-extensions-table.html)。
>
>[open_secondary_engine_tables函数会根据查询的语法树（LEX对象）来确定需要打开哪些次级存储引擎的表，并调用open_table()函数来逐个打开它们](https://dev.mysql.com/doc/refman/8.0/en/information-schema-table-constraints-extensions-table.html)[3](https://dev.mysql.com/doc/refman/8.0/en/information-schema-table-constraints-extensions-table.html)[。如果遇到错误或者超过了最大文件描述符限制，函数会返回失败](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)[4](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)。
>
> [目前，MySQL 8.0支持的次级存储引擎有HeatWave（RAPID），它是MySQL数据库服务（MySQL Database Service）和HeatWave服务的一部分，用于加速分析型查询](https://dev.mysql.com/doc/refman/8.0/en/table-cache.html)[5](https://dev.mysql.com/doc/refman/8.0/en/table-cache.html)[。您可以使用INFORMATION_SCHEMA.TABLES_EXTENSIONS表来查看一个表是否有次级存储引擎属性](https://dev.mysql.com/doc/refman/8.0/en/innodb-table-import.html)[6](https://dev.mysql.com/doc/refman/8.0/en/innodb-table-import.html)。例如：
>
> ```sql
> SELECT * FROM INFORMATION_SCHEMA.TABLES_EXTENSIONS WHERE TABLE_NAME = 't1';
> ```
> 
> 这个例子中，会显示t1表是否有次级存储引擎属性，以及该属性的值。

```cpp
static bool open_secondary_engine_tables(THD *thd, uint flags) {
  // 进行一系列检查，判断当前 Query 是否能使用 Secondary Engine
  ...

  // 将 plugin 注册为当前 sql_cmd 的 Secondary Engine Handle
  auto hton = plugin_data<const handlerton *>(secondary_engine_plugin);
  sql_cmd->use_secondary_storage_engine(hton);

  // Replace the TABLE objects in the TABLE_LIST with secondary tables.
  Open_table_context ot_ctx(thd, flags | MYSQL_OPEN_SECONDARY_ENGINE);
  TABLE_LIST *tl = lex->query_tables;

  ...
  for (; tl != nullptr; tl = tl->next_global) {
    if (open_table(thd, tl, &ot_ctx)) {
      ...
      return true;
    }
    assert(tl->table->s->is_secondary_engine());
    tl->table->file->ha_set_primary_handler(primary_table->file);
  }

  // Prepare the secondary engine for executing the statement.
  return hton->prepare_secondary_engine != nullptr &&
         hton->prepare_secondary_engine(thd, lex);
}
```

当一切准备就绪，当前 Query 能够被 Secondary 执行时，MySQL 最后会调用 Secondary Engine 插件实现的 `prepare_secondary_engine()` 接口，这个接口的定义如下：

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

这个接口用来给在 secondary engine 中执行的 query 创建需要使用的 query context，然后通过当前 query 的 LEX 注册进去，后续就可以通过 LEX 访问这个 query context 了。MySQL 提供的默认实现在 `Secondary_engine_execution_context` 中，没有包含任何数据，secondary engine 的实现者仅需要继承该 class 即可。MySQL 代码中 mock 的实现如下：

```cpp
static bool PrepareSecondaryEngine(THD *thd, LEX *lex) {
  ...
  auto context = new (thd->mem_root) Mock_execution_context;
  ...
  lex->set_secondary_engine_execution_context(context);

  // Disable use of constant tables and evaluation of subqueries during
  // optimization.
  lex->add_statement_options(OPTION_NO_CONST_TABLES |
                             OPTION_NO_SUBQUERY_DURING_OPTIMIZATION);

  return false;
}
```

这个 `Mock_execution_context` 定义如下：

```cpp
class Mock_execution_context : public Secondary_engine_execution_context {
  ...
};
```

后续在 Secondary Engine 的查询优化和执行过程中就可以利用这个 context 存储和获取需要的上下文信息。

## 3. The optimize stage

#### Sql_cmd_dml::execute_inner()

经过 prepare 后，SELECT 语句的执行就由 `Sql_cmd_dml::execute_inner(THD *thd)` 驱动了，里面包含了查询优化和执行的完整流程，Secondary Engine 读流程的接口也是在这些阶段中被调用：
1. `unit->optimize()`：第 1 步，完成查询在 MySQL 中的优化
2. `optimize_secondary_engine()`：第 2 步，完成在 Secondary Engine 中的优化
3. `unit->execute()`：最后一步，完成查询的执行

> From new bing:
> 
> Sql_cmd_dml::execute_inner是一个虚函数，它的主要功能是执行一个DML（数据操作语言）语句的内部部分，包括查询优化和执行[1](https://dev.mysql.com/doc/dev/mysql-server/latest/classSql__cmd__show__master__status.html)[。DML语句是指对数据库中的数据进行增删改查的语句，如SELECT、INSERT、UPDATE、DELETE等](https://dev.mysql.com/doc/refman/8.0/en/sql-data-manipulation-statements.html)[2](https://dev.mysql.com/doc/refman/8.0/en/sql-data-manipulation-statements.html)。
> 
> [Sql_cmd_dml::execute_inner函数会根据查询表达式（query expression）的类型来调用不同的实现方法。如果查询表达式只包含一个查询块（query block），则调用Sql_cmd_dml::execute_inner_single_table()函数；如果查询表达式包含多个查询块，用UNION连接，则调用Sql_cmd_dml::execute_inner_union()函数](https://dev.mysql.com/doc/dev/mysql-server/8.0.22/classSql__cmd__show__errors.html)[3](https://dev.mysql.com/doc/dev/mysql-server/8.0.22/classSql__cmd__show__errors.html)。
> 
> [每个派生自Sql_cmd_dml的类都需要重写该函数，以实现特定DML语句的逻辑](https://dev.mysql.com/worklog/task/?id=5094)[4](https://dev.mysql.com/worklog/task/?id=5094)[。例如，Sql_cmd_show_master_status类重写了该函数，以显示复制主服务器状态信息](https://dev.mysql.com/doc/dev/mysql-server/latest/classSql__cmd__show__master__status.html)[1](https://dev.mysql.com/doc/dev/mysql-server/latest/classSql__cmd__show__master__status.html)。
>
> [如果您想要在mysql客户端中执行一个SQL文件中的DML语句，您可以使用以下命令](https://dev.mysql.com/doc/refman/8.0/en/mysql-batch-commands.html)[5](https://dev.mysql.com/doc/refman/8.0/en/mysql-batch-commands.html)：
> 
> ```bash
> mysql db_name < text_file
> ```
> 
> 这个例子中，会从text_file文件中读取SQL语句，并在db_name数据库中执行它们。

代码比较短，逻辑整体上比较清晰：

```cpp
bool Sql_cmd_dml::execute_inner(THD *thd) {
  Query_expression *unit = lex->unit;

  if (unit->optimize(thd, /*materialize_destination=*/nullptr,
                     /*create_iterators=*/true, /*finalize_access_paths=*/true))
    return true;

  // Calculate the current statement cost.
  accumulate_statement_cost(lex);

  // Perform secondary engine optimizations, if needed.
  if (optimize_secondary_engine(thd)) return true;

  // We know by now that execution will complete (successful or with error)
  lex->set_exec_completed();
  if (lex->is_explain()) {
    if (explain_query(thd, thd, unit)) return true; /* purecov: inspected */
  } else {
    if (unit->execute(thd)) return true;
  }

  return false;
}
```

#### compare_secondary_engine_cost()

- [ ] TODO: 如何根据 cost 判断是采用 primary engine 执行还是 secondary engine 执行

MySQL 有两个 Join Reorder 算法：Hypergraph 和 Greedy。Hypergraph 是 MySQL 8.0 新引入的目前还没有默认采用，默认情况下还是使用的 Greedy 的 Join Reorder。这个 Join Reorder 算法会尝试枚举可能的 Join Order 和在这个 Order 下各个表的 Access Path，得到一个 Join Order  后 MySQL 会为其估算 cost，保留所有 Join Order 中代价最小的那个作为最终执行计划。这部分代码在 `Optimize_table_order::greedy_search(table_map remaining_tables)` 中，代码的注释中有比较详细的算法介绍和 pseudo code，感兴趣的朋友可以详细看看，这里篇幅所限不再展开。

在得到一个 Join Order 为其估算 cost 后，如果当前 Join Group 可以采用 Secondary Engine 执行，则会调用 Secondary Engine 的 cost 估算和比较函数，MySQL 也会把自己估算出来的 cost 传过去。Secondary Engine 可以有自己的 cost model 为当前 Join Order 估算新的 cost，也可以直接使用 MySQL 估算的 cost。

MySQL 调用 Secondary Engine 的 `compare_secondary_engine_cost()` 的地方位于 `Optimize_table_order::consider_plan()` 的如下代码片段：

```cpp
/*
  If the statement is executed on a secondary engine, and the secondary engine
  has implemented a custom cost comparison function, ask the secondary engine
  to compare the cost. The secondary engine is only consulted when a complete
  join order is considered.
*/
if (idx + 1 == join->tables) {  // this is a complete join order
  const handlerton *secondary_engine = SecondaryEngineHandlerton(thd);
  if (secondary_engine != nullptr &&
      secondary_engine->compare_secondary_engine_cost != nullptr) {
    double secondary_engine_cost;
    if (secondary_engine->compare_secondary_engine_cost(
            thd, *join, cost, &use_best_so_far, &cheaper,
            &secondary_engine_cost))
      return true;
    chosen = cheaper;
    trace_obj->add("secondary_engine_cost", secondary_engine_cost);

    // If this is the first plan seen, it must be chosen.
    assert(join->best_read != DBL_MAX || chosen);
  }
}
```

什么情况下 `secondary_engine` 为 nullptr 呢？
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

#### optimize_secondary_engine()

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

## 4. The execute stage

### override_executor_func

mock Secondary Engine 没有如何将当前 query 交给 Secondary Engine 执行的例子，不过细心的朋友不难发现在 Secondary Engine Flag 的地方有这么一个注释：

```cpp
// Capabilities (bit flags) for secondary engines.
using SecondaryEngineFlags = uint64_t;
enum class SecondaryEngineFlag : SecondaryEngineFlags {
  SUPPORTS_HASH_JOIN = 0,
  SUPPORTS_NESTED_LOOP_JOIN = 1,

  // If this flag is set, aggregation (GROUP BY and DISTINCT) do not require
  // ordered inputs and create unordered outputs. This is typically the case
  // if they are implemented using hash-based techniques.
  AGGREGATION_IS_UNORDERED = 2,

  /// This flag can be set to signal that a secondary storage engine will not
  /// use MySQL's executor (see JOIN::override_executor_func). In this case, it
  /// doesn't need MySQL's execution data structures, like internal temporary
  /// tables, filesort objects or iterators. If the flag is set,
  /// FinalizePlanForQueryBlock() will not make any changes to the plan, and
  /// CreateIteratorFromAccessPath() will not be called.
  USE_EXTERNAL_EXECUTOR = 3,
};
```

在 `USE_EXTERNAL_EXECUTOR` 的注释中，MySQL 提到了一个叫 `JOIN::override_executor_func` 的关键函数，搜索代码发现使用它的地方仅有 `Query_expression::ExecuteIteratorQuery(THD *thd)` 这么一个地方。

`Query_expression::ExecuteIteratorQuery(THD *thd)` 是 MySQL 执行查询的函数。在这里，它通过检查当前 query block 中的 `override_executor_func` 函数指针是否为空来判断当前 query 是否放到 Secondary Engine 中执行：

```cpp
bool Query_expression::ExecuteIteratorQuery(THD *thd) {
  ...
  // Hand over the query to the secondary engine if needed.
  if (first_query_block()->join->override_executor_func != nullptr) {
    thd->current_found_rows = 0;
    for (Query_block *select = first_query_block(); select != nullptr;
         select = select->next_query_block()) {
      if (select->join->override_executor_func(select->join, query_result)) {
        return true;
      }
      thd->current_found_rows += select->join->send_records;
    }
    const bool calc_found_rows =
        (first_query_block()->active_options() & OPTION_FOUND_ROWS);
    if (!calc_found_rows) {
      // This is for backwards compatibility reasons only;
      // we have documented that without SQL_CALC_FOUND_ROWS,
      // we return the actual number of rows returned.
      thd->current_found_rows =
          std::min(thd->current_found_rows, select_limit_cnt);
    }
    return query_result->send_eof(thd);
  }
  ...
```

这个函数的定义在 sql_optimizer.h 中，这个函数的第 1 个参数是一个 Query Block 的物理执行计划，第 2 个参数是执行这个物理执行计划后返回的结果需要写回的地方：

```cpp
/// A hook that secondary storage engines can use to override the executor
/// completely.
using Override_executor_func = bool (*)(JOIN *, Query_result *);
Override_executor_func override_executor_func = nullptr;
```

现在看来，Secondary Engine 只需要实现一个自己的 `Override_executor_func`，在合适的时机将 `override_executor_func` 设置成 Secondary Engine 里 Query Processing 的函数指针就行了。而最佳的时机就是在完成 `optimize_secondary_engine()` 后。MySQL 在后续执行查询的时候会拿着这个 Secondary Engine 的 executor function 完成查询的执行，并且将结果写到函数的第 2 个参数指定的 `Query_result` 中。

以上就是 MySQL 如何将 Query Plan 交给 Secondary Engine 再优化，如何将最终物理执行计划交给 Secondary Engine 执行并取回结果的完整过程。


