---
title: "[MySQL] Optimizer Introduction"
date: 2022-11-14T06:07:55Z
categories: ["MySQL"]
draft: true
---

## MySQL SQL 执行概览

DML 执行的控制逻辑：

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

整个 SQL 执行分为这些阶段：

1. Query_expression::optimize()：优化当前 Query 得到物理执行计划
2. Sql_cmd_dml::accumulate_statement_cost() 计算物理执行计划的累加 cost
3. Sql_cmd_dml::optimize_secondary_engine() 针对第二存储引擎进行二次优化
4. Query_expression::execute() 执行当前 Query



我们先来看看 MySQL 查询优化过程。

## 第一部分：Optimize a query

查询优化的入口是 Query_expression::optimize() 函数。MySQL 将 SQL parse 后切分成了多个 query block 用 `Query_block` 表示，所有的 query block 都存储在 `Query_expression` 中，在 query optimize 中逐个优化所有的 query block：

```cpp
bool Query_expression::optimize(THD *thd, TABLE *materialize_destination,
                                bool create_iterators,
                                bool finalize_access_paths) {
...

  for (Query_block *query_block = first_query_block(); query_block != nullptr;
       query_block = query_block->next_query_block()) {
    thd->lex->set_current_query_block(query_block);

    // LIMIT is required for optimization
    if (set_limit(thd, query_block)) return true; /* purecov: inspected */

    if (query_block->optimize(thd, finalize_access_paths)) return true;

    /*
      Accumulate estimated number of rows.
      1. Implicitly grouped query has one row (with HAVING it has zero or one
         rows).
      2. If GROUP BY clause is optimized away because it was a constant then
         query produces at most one row.
     */
    if (contributes_to_rowcount_estimate(query_block))
      estimated_rowcount += (query_block->is_implicitly_grouped() ||
                             query_block->join->group_optimized_away)
                                ? 1
                                : query_block->join->best_rowcount;

    estimated_cost += query_block->join->best_read;

    // TABLE_LIST::fetch_number_of_rows() expects to get the number of rows
    // from all earlier query blocks from the query result, so we need to update
    // it as we go. In particular, this is used when optimizing a recursive
    // SELECT in a CTE, so that it knows how many rows the non-recursive query
    // blocks will produce.
    //
    // TODO(sgunders): Communicate this in a different way when the query result
    // goes away.
    if (query_result() != nullptr) {
      query_result()->estimated_rowcount = estimated_rowcount;
      query_result()->estimated_cost = estimated_cost;
    }
  }

...

```



### Optimize a query block

```cpp
/**
  Optimize a query block and all inner query expressions

  @param thd    thread handler
  @param finalize_access_paths
                if true, finalize access paths, cf. FinalizePlanForQueryBlock
  @returns false if success, true if error
*/

bool Query_block::optimize(THD *thd, bool finalize_access_paths) {
  DBUG_TRACE;

  assert(join == nullptr);
  JOIN *const join_local = new (thd->mem_root) JOIN(thd, this);
  if (!join_local) return true; /* purecov: inspected */

  /*
    Updating Query_block::join requires acquiring THD::LOCK_query_plan
    to avoid races when EXPLAIN FOR CONNECTION is used.
  */
  thd->lock_query_plan();
  join = join_local;
  thd->unlock_query_plan();

  if (join->optimize(finalize_access_paths)) return true;

  if (join->zero_result_cause && !is_implicitly_grouped()) return false;

  for (Query_expression *query_expression = first_inner_query_expression();
       query_expression;
       query_expression = query_expression->next_query_expression()) {
    // Derived tables and const subqueries are already optimized
    if (!query_expression->is_optimized() &&
        query_expression->optimize(thd, /*materialize_destination=*/nullptr,
                                   /*create_iterators=*/false,
                                   /*finalize_access_paths=*/true))
      return true;
  }

  return false;
}
```



Query_block 的优化则主要通过  [Join::optimize()](https://dev.mysql.com/doc/dev/mysql-server/latest/group__Query__Optimizer.html#gadc6dcb5e0bac2637b7d9de0ff376c0f4) 函数完成：

> Optimizes one query block into a query execution plan (QEP.)
>
> This is the entry point to the query optimization phase. This phase applies both logical (equivalent) query rewrites, cost-based join optimization, and rule-based access path selection. Once an optimal plan is found, the member function creates/initializes all structures needed for query execution. The main optimization phases are outlined below:
>
> 1. Logical transformations:
>    - Outer to inner joins transformation.
>    - Equality/constant propagation.
>    - Partition pruning.
>    - COUNT(*), MIN(), MAX() constant substitution in case of implicit grouping.
>    - ORDER BY optimization.
> 2. Perform cost-based optimization of table order and access path selection. See JOIN::make_join_plan()
> 3. Post-join order optimization:
>    - Create optimal table conditions from the where clause and the join conditions.
>    - Inject outer-join guarding conditions.
>    - Adjust data access methods after determining table condition (several times.)
>    - Optimize ORDER BY/DISTINCT.
> 4. Code generation
>    - Set data access functions.
>    - Try to optimize away sorting/distinct.
>    - Setup temporary table usage for grouping and/or sorting.



### Query Resolver

看到的一个堆栈如下：

```
#0  Query_block::prepare (this=0x7fa430120f68, thd=0x7fa430001050, insert_field_list=0x0) at /home/jian.z/code/mysql-server/sql/sql_resolver.cc:184
#1  0x000055fa1161af87 in Sql_cmd_select::prepare_inner (this=0x7fa430124e98, thd=0x7fa430001050) at /home/jian.z/code/mysql-server/sql/sql_select.cc:479
#2  0x000055fa1161a98c in Sql_cmd_dml::prepare (this=0x7fa430124e98, thd=0x7fa430001050) at /home/jian.z/code/mysql-server/sql/sql_select.cc:395
#3  0x000055fa1161b251 in Sql_cmd_dml::execute (this=0x7fa430124e98, thd=0x7fa430001050) at /home/jian.z/code/mysql-server/sql/sql_select.cc:534
#4  0x000055fa1158f3cd in mysql_execute_command (thd=0x7fa430001050, first_level=true) at /home/jian.z/code/mysql-server/sql/sql_parse.cc:4677
#5  0x000055fa1159172a in dispatch_sql_command (thd=0x7fa430001050, parser_state=0x7fa4d42459f0) at /home/jian.z/code/mysql-server/sql/sql_parse.cc:5312
#6  0x000055fa11586df5 in dispatch_command (thd=0x7fa430001050, com_data=0x7fa4d4246340, command=COM_QUERY) at /home/jian.z/code/mysql-server/sql/sql_parse.cc:2032
#7  0x000055fa11584da3 in do_command (thd=0x7fa430001050) at /home/jian.z/code/mysql-server/sql/sql_parse.cc:1435
#8  0x000055fa117cf563 in handle_connection (arg=0x55fa1a34c550) at /home/jian.z/code/mysql-server/sql/conn_handler/connection_handler_per_thread.cc:302
#9  0x000055fa13af9764 in pfs_spawn_thread (arg=0x55fa1a2c6c60) at /home/jian.z/code/mysql-server/storage/perfschema/pfs.cc:2986
#10 0x00007fa4fce87b43 in start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:442
#11 0x00007fa4fcf18bb4 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:100
```

推测 Query Resolver 的入口是 Query_block::prepare() 函数



### Query Planner

```
Optimize_table_order::optimize_straight_join
Optimize_table_order::choose_table_order
```





## References

* https://dev.mysql.com/doc/dev/mysql-server/latest/
