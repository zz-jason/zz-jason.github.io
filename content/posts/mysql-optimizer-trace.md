---
title: "[MySQL] 通过 Optimizer Trace 理解查询优化"
date: 2023-03-31T00:00:00Z
categories: ["MySQL"]
draft: true
---

以 TPC-H Q4 为例分析 MySQL 8.0.31 查询优化的过程。这里我们主要关注几个问题：

1. Q4 的 exists 子查询是如何优化成 semi join 的
2. nested loop semi join 是如何执行的

在分析过程中，我们会使用到 MySQL 的两个 debug 工具：

1. optimizer trace
2. DEBUG

## TPC-H Q4 以及 MySQL 的执行计划

TPC-H Q4 如下：

```sql
SELECT
    O_ORDERPRIORITY,
    COUNT(*) AS ORDER_COUNT
FROM
    ORDERS
WHERE
    O_ORDERDATE >= DATE '1993-12-01'
    AND O_ORDERDATE < DATE '1993-12-01' + INTERVAL '3' MONTH
    AND EXISTS (
        SELECT
            *
        FROM
            LINEITEM
        WHERE
            L_ORDERKEY = O_ORDERKEY
            AND L_COMMITDATE < L_RECEIPTDATE
    )
GROUP BY
    O_ORDERPRIORITY
ORDER BY
    O_ORDERPRIORITY;
```

其执行计划如下：

```text
-> Sort: ORDERS.O_ORDERPRIORITY
    -> Table scan on <temporary>
        -> Aggregate using temporary table
            -> Nested loop semijoin  (cost=391601.89 rows=216090)
                -> Filter: ((ORDERS.O_ORDERDATE >= DATE'1993-12-01') and (ORDERS.O_ORDERDATE < <cache>((DATE'1993-12-01' + interval '3' month))))  (cost=160543.89 rows=165020)
                    -> Table scan on ORDERS  (cost=160543.89 rows=1485477)
                -> Filter: (LINEITEM.L_COMMITDATE < LINEITEM.L_RECEIPTDATE)  (cost=1.32 rows=1)
                    -> Index lookup on LINEITEM using PRIMARY (L_ORDERKEY=ORDERS.O_ORDERKEY)  (cost=1.32 rows=4)
```

Q4 的主要目标是找出所有在 '1993-12-01' 和 '1993-12-04' 之间所有 L_COMMITDATE < L_RECEIPTDATE 的订单，按 order priority 分组求每组的总数。在过滤订单数据时使用了一个 exists 子查询：

```sql
EXISTS (
    SELECT
        *
    FROM
        LINEITEM
    WHERE
        L_ORDERKEY = O_ORDERKEY
        AND L_COMMITDATE < L_RECEIPTDATE
)
```

从上面的执行计划可以看到，这个子查询被 MySQL 优化成了 semi join：先用 orders 表 left semi join lineitem 表，然后对 semi join 的结果求聚合，最后对聚合的结果排序。上面的执行计划透露的一些信息我们不是很明确，需要在接下来的代码分析中搞明白：

1. 为什么 semi join 的 join predicate `L_ORDERKEY=ORDERS.O_ORDERKEY` 被推到了 lineitem 表的 index lookup 上？mysql 的 nested loop semijoin 是如何执行的？
2. `((ORDERS.O_ORDERDATE >= DATE'1993-12-01') and (ORDERS.O_ORDERDATE < <cache>((DATE'1993-12-01' + interval '3' month))))` 被推到了 orders 表的 table scan 以后，这个 filter pushdown 是在什么时候完成的，如何知道哪些 filter 可以下推，哪些不行？
3. Aggregate 后面有个 `using temporary table` 的含义是什么，mysql 的 aggregate 是如何执行的？
4. 每个算子最后的 `cost=0.35 rows=1` 是什么，在什么阶段，如何估算的？

我对 MySQL 优化器代码不是很熟悉，不过 MySQL 有 optimizer trace 的功能，我们接下来看看是否能从 optimizer trace 中发现一些有用信息，然后再根据 trace 中输出的关键信息去看对应的代码，反推 MySQL 的查询优化过程。

## MySQL Optimizer Trace 简介

MySQL optimizer trace 的详细介绍可以参考 MySQL 的开发文档：[The Optimizer Trace](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_OPT_TRACE.html)，为了使用这个 trace，我们需要执行 query 之前在当前 session 中将 `optimizer_trace` 开关打开，执行完 query 后 mysql 将 trace 信息保存在 information_schema.optimizer_trace 表中，只需要查看这个表就能看到所有的 trace 信息了：

```sql
set @@optimizer_trace="enabled=on";
source q4.sql
select * from information_schema.optimizer_trace\G
```

另外 MySQL 提供了 OPTIMIZER_TRACE_FEATURES 这个变量，用于控制追踪哪些类型的信息，默认 MySQL 会追踪所有类型的 trace 数据，当我们有明确的关注点，希望减少 trace 信息中的噪音时，可以使用这个变量来精准 trace。我们在代码中搜索 `Opt_trace_array` 能发现有大量不同的 trace 信息，他们都可以通过 OPTIMIZER_TRACE_FEATURES 来控制。

## MySQL DEBUG 简介

## Optimization Steps of Q4

通过 optimizer trace 的结果看到，这条 Query 的执行在 MySQL 中先后经历了下面 3 个阶段：
- join_preparation
- join_optimization
- join_execution

### 阶段  1：join_preparation

通过 Optimizer Trace 我们可以看到，Q4 在 join_preparation 阶段经过了 4 个 step，在最后一个 step 已经被改写成了  semi join，关联表达式 `O_ORDERKEY=L_ORDERKEY` 也被放入了 join predicate 中：

```sql
/* select#1 */ select `ORDERS`.`O_ORDERPRIORITY` AS `O_ORDERPRIORITY`,count(0) AS `ORDER_COUNT` from `ORDERS` semi join (`LINEITEM`) where ((`ORDERS`.`O_ORDERDATE` >= DATE'1993-12-01') and (`ORDERS`.`O_ORDERDATE` < (DATE'1993-12-01' + interval '3' month)) and (`LINEITEM`.`L_COMMITDATE` < `LINEITEM`.`L_RECEIPTDATE`) and (`ORDERS`.`O_ORDERKEY` = `LINEITEM`.`L_ORDERKEY`)) group by `ORDERS`.`O_ORDERPRIORITY` order by `ORDERS`.`O_ORDERPRIORITY`
```

Q4 在 join_preparation 阶段经历的 4 个 step 分别是：
- 对内部的 Query_block 进行 `join_preparation`
- 暂时没有名字，不过能看出来内部的 Query_block 已经被 expand 到外部 Query_block 中了
- transformation_to_semi_join：将 in 子查询转换为 semi join
- transformations_to_nested_joins：将 semi join 转换为 nested semi join

问题：
1. in、exists 等所有子查询的改写都是发生在这一阶段吗？
2. mysql 有哪些子查询改写策略？
3. 为啥要在 prepare 阶段将 semi join 转换为 nested join？

### 阶段 2：join_optimize

这个阶段的 step 就比较多了：
- condition_processing
- substitute_generated_columns
- table_dependencies
- ref_optimizer_key_uses
- pulled_out_semijoin_tables
- rows_estimation
- execution_plan_for_potential_materialization
- considered_execution_plans
- attaching_conditions_to_tables
- optimizing_distinct_group_by_order_by
- finalizing_table_conditions
- refine_plan
- considering_tmp_tables

问题：
1. 这么多优化 step，MySQL 的优化过程有先做什么后做什么的原则吗？
2. where condition 中的 filter 是如何下推到各个 child 上的？

Query_block 的优化发生在位于 sql_optimizer.cc 的 JOIN::optimize() 中，我们看见的所有 `join_optimize` 相关的 trace 信息都是在这个 800 行左右的函数内记录的。

#### step 1：condition_processing

由 JOIN::optimize() 驱动 optimize_cond() 函数完成，入参为：
- 当前 JOIN 的 where_cond：存储对 JOIN 结果的 predicate
- 当前 JOIN 的 cond_equal：存储 multi equal item
- 当前 Query_block 的 top_join_list
- 当前 Query_block 的 cond_value

```cpp
  if (where_cond || query_block->outer_join) {
    if (optimize_cond(thd, &where_cond, &cond_equal,
                      &query_block->top_join_list, &query_block->cond_value)) {
      error = 1;
      DBUG_PRINT("error", ("Error from optimize_cond"));
      return true;
    }
    if (query_block->cond_value == Item::COND_FALSE) {
      zero_result_cause = "Impossible WHERE";
      best_rowcount = 0;
      create_access_paths_for_zero_rows();
      goto setup_subq_exit;
    }
  }
```

optimize_cond() 主要作用：
- equality_propagation，通过 `build_equal_items()` 完成
- constant_propagation，通过 `propagate_cond_constants()` 完成
- trivial_condition_removal，通过 `remove_eq_conds()` 完成

#### step 2：substitute_generated_columns

optimizer trace 里 Q4 经历的第 2 个优化 step 就是 substitute_generated_columns，但实际上 MySQL 会依次进行：
1. where 和 having 里面 predicate 的 condition_processing
2. partition pruning
3. optimize_aggregated_query

然后才进入到这个 step 中，由 JOIN::optimize() 驱动 substitute_gc() 函数完成：
```cpp
  if ((where_cond || !group_list.empty() || !order.empty()) &&
      substitute_gc(thd, query_block, where_cond, group_list.order,
                    order.order)) {
    // We added hidden fields to the all_fields list, count them.
    count_field_types(query_block, &tmp_table_param, query_block->fields, false,
                      false);
  }
```

因为 Q4 没有 generated column，所以 substitute_g() 没对 where_cond 做任何处理，optimizer trace 中这个 step 的信息也为空。

#### step 3：table_dependencies

这个仅仅是 MySQL 用来输出当前 JOIN list 中所有 join table 自身的 map bit 和它所依赖的 table 的 map bit。MySQL 采用状态压缩的方式，将每个 table 用 uint64_t 的一个比特位来表示，第 i 位为 1 表示第 i 个 table 被引用。

JOIN::optimize() 完成其他优化后开始优化 join order，这一步通过 make_join_plan() 完成：

```cpp
// Set up join order and initial access paths
THD_STAGE_INFO(thd, stage_statistics);
if (make_join_plan()) {
  if (thd->killed) thd->send_kill_message();
  DBUG_PRINT("error", ("Error: JOIN::make_join_plan() failed"));
  return true;
}
```

而在 `make_join_plan()` 中进一步调用 `trace_table_dependencies()` 打印出来方便 debug：

```cpp
  if (unlikely(trace->is_started()))
    trace_table_dependencies(trace, join_tab, primary_tables);
```

#### step 4：ref_optimizer_key_uses

在 `make_join_plan()` 中，打印完各个表的 table map id 和它们依赖的 table map 后，就通过调用 `update_ref_and_keys()`：

```cpp
// Build the key access information, which is the basis for ref access.
if (where_cond || query_block->outer_join) {
  if (update_ref_and_keys(thd, &keyuse_array, join_tab, tables, where_cond,
                          ~query_block->outer_join, query_block, &sargables))
    return true;
}
```

在 update_ref_and_keys() 的末尾通过 `print_keyuse_array()` 打印出来构造好的 key use 信息：

```cpp
print_keyuse_array(thd, &thd->opt_trace, keyuse);
```

#### step 5：pulled_out_semijoin_tables

还是在 `make_join_plan()` 中，更新完 key 和 ref 后，如果当前 Query_block 中存在 nested semi join 则进行这个优化：

```cpp
/*
  Pull out semi-join tables based on dependencies. Dependencies are valid
  throughout the lifetime of a query, so this operation can be performed
  on the first optimization only.
*/
if (!query_block->sj_pullout_done && !query_block->sj_nests.empty() &&
    pull_out_semijoin_tables(this))
  return true;
```

#### step 6：rows_estimation

还是在 make_join_plan() 中，对 Q4 来说，进行完上面的优化后就开始为当前 Query_block 中的各个 table 估算 row count：

```cpp
// Make a first estimate of the fanout for each table in the query block.
if (estimate_rowcount()) return true;
```

#### step 7：execution_plan_for_potential_materialization

还是在 make_join_plan 中：

```cpp
if (sj_nests && optimize_semijoin_nests_for_materialization(this))
  return true;
```

#### step 8：considered_execution_plans

在上面 optimize_semijoin_nests_for_materialization() 中，对所有的 nested join 都会调用 choose_table_order() 进行优化：

```cpp
for (TABLE_LIST *sj_nest : join->query_block->sj_nests) {
	// ...
    if (Optimize_table_order(join->thd, join, sj_nest).choose_table_order())
      return true;
	// ...
}
```

#### step 9：attaching_conditions_to_tables

在 JOIN::optimize 中，当 make_join_plan() 结束后进行 make_join_query_block() 的优化：

```cpp
if (make_join_query_block(this, where_cond)) {
  if (thd->is_error()) return true;

  zero_result_cause = "Impossible WHERE noticed after reading const tables";
  create_access_paths_for_zero_rows();
  goto setup_subq_exit;
}
```

在 make_join_query_block 中进行该优化：

```cpp
/*
  Extract remaining conditions from WHERE clause and join conditions,
  and attach them to the most appropriate table condition. This means that
  a condition will be evaluated as soon as all fields it depends on are
  available. For outer join conditions, the additional criterion is that
  we must have determined whether outer-joined rows are available, or
  have been NULL-extended, see JOIN::attach_join_conditions() for details.
*/
```

#### step 10：optimizing_distinct_group_by_order_by

还是在 JOIN::optimize 中，完成 make_join_query_block 后，会进行该优化：

```cpp
if (optimize_distinct_group_order()) return true;
```

#### step 11：finalizing_table_conditions

还是在 JOIN::optimize 中，完成上面的优化后接下来开始这个优化：

```cpp
if (!plan_is_const()) {
  // Test if we can use an index instead of sorting
  test_skip_sort();

  if (finalize_table_conditions()) return true;
}
```

#### step 12：refine_plan

还是在 JOIN::optimize 里面，完成上面的优化后开始 refine_plan：

```cpp
if (make_join_readinfo(this, no_jbuf_after))
  return true; /* purecov: inspected */
```

#### step 13：considering_tmp_tables

还是在  JOIN::optimize 里面：

```cpp
if (make_tmp_tables_info()) return true;
```

这个基本上就是 JOIN::optimize 的末尾了。

### 阶段 3：join_execution

- TODO：debug 一下看看这个是谁在调用

join_execution 发生在 `Query_expression::ExecuteIteratorQuery()` 中

这个阶段仅有 2 个 step：
- temp_table_aggregate
- sorting_table

问题：
1. optimizer trace 为啥会有 execution 相关的逻辑？
2. 另外这里面看起来并没有罗列全所有的 execution 逻辑？