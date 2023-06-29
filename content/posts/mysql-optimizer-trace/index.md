---
title: "[MySQL 8.0] 通过 Optimizer Trace 概览查询优化"
date: 2023-06-07T00:00:00Z
categories: ["MySQL", "Query Optimization"]
---

![](featured.jpeg)

## 简介

MySQL 查询优化是一个非常复杂的过程，幸运的是，它支持了 optimizer trace 功能，使我们能够快速概览 MySQL 的查询优化过程。本文将以 TPC-H Q4 为例，通过 optimizer trace 功能，对 MySQL 8.0.31 查询优化的先后步骤和代码路径进行总结，以便更深入地了解 MySQL 查询优化的细节。

## TPC-H Q4 及其执行计划

TPC-H Q4 如下，它包含了分组聚合和子查询，是个稍微有点复杂的 SQL，在过滤 orders 表时使用了一个 exists 子查询：

```sql
SELECT O_ORDERPRIORITY,
       COUNT(*) AS ORDER_COUNT
FROM ORDERS
WHERE O_ORDERDATE >= DATE '1993-12-01'
  AND O_ORDERDATE < DATE '1993-12-01' + INTERVAL '3' MONTH
  AND EXISTS (
	      SELECT *
	      FROM LINEITEM
          WHERE L_ORDERKEY = O_ORDERKEY
            AND L_COMMITDATE < L_RECEIPTDATE
      )
GROUP BY O_ORDERPRIORITY
ORDER BY O_ORDERPRIORITY;
```

通过 `explain format=tree` 可以看到如下的执行计划。总结来说是先让 orders 表和 lineitem 表进行 nested loop semi join，然后对 join 结果进行分组聚合，最后对聚合结果进行排序，输出结果给客户端：

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

关于上面的执行计划我们重点关注这些问题：
1. MySQL 是在查询优化的哪个步骤将 exists 子查询改写成 semi join 的？
2. MySQL 代价估算是在什么时候完成的？
3. MySQL 的表中有哪些统计信息，如何获取？

## Optimizer Trace 用法和原理

MySQL Optimizer Trace 的详细用户文档见《[The Optimizer Trace](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_OPT_TRACE.html)》。为了 trace Q4 的优化过程，我们需要在执行 SQL 前把当前 session 的 `optimizer_trace` 打开，执行 SQL 后在 `information_schema.optimizer_trace` 表中获取 MySQL 为其生成的 trace 信息：

```sql
set @@optimizer_trace="enabled=on";
source q4.sql
select * from information_schema.optimizer_trace\G
```

MySQL Trace 的原理非常简单，就是在代码关键路径中埋点，将需要 trace 的信息通过诸如 `Opt_trace_array` 的形式添加到 optimizer trace 结果的 json 对象中，比如：

```cpp
  Opt_trace_context *const trace = &thd->opt_trace;
  Opt_trace_object trace_wrapper(trace);
  Opt_trace_object trace_optimize(trace, "join_optimization");
  trace_optimize.add_select_number(query_block->select_number);
  Opt_trace_array trace_steps(trace, "steps");
```

## Optimizer Trace 结果简介

Q4 的 Optimizer trace 结果 [trace.json](trace.json) 是一个很大的 json 文本，可以使用能够折叠 json 对象的编辑器或在线网站来辅助分析这个 json 文本。从 trace 结果来看，这条 SQL 在 MySQL 中先后经历了下面 3 个阶段。

阶段 1：join_preparation。位于 `sql_resolver.cc` 的 `Query_block::prepare()` 函数中，主要功能是解析 AST 上的各个 SQL 子句，同时也完成子查询相关的转换和优化，比如转换成 semi join，推导 table information，常量消除，冗余表达式消除等，干的事情比较杂。

阶段 2：join_optimization。位于 `sql_optimizer.cc` 的 `JOIN::optimize()` 函数中，该函数包含了查询优化的主要逻辑，通过一系列逻辑等价的 query rewrite，cost based join optimization，rule-based access path selection 等优化步骤将 Query_block 优化成 query execution plan（QEP）。

阶段 3：join_execution。位于 `sql_union.cc` 的  `Query_expression::ExecuteIteratorQuery()` 函数中，主要负责查询执行。

接下来我们简单看看各个阶段的主要优化步骤，尝试回答一开始提出的几个问题。

## 阶段 1：join_preparation

join_preparation 的入口是 `Query_block::prepare()`，它驱动了所有子过程。

### setup_conds

第 1 个 step 就是 exists 子查询所在的 Query_block 对应的 "join_preparation"，从 gdb 可以看到完整的调用路径：

```txt
Query_block::prepare sql_resolver.cc:300
  -> Query_block::setup_conds sql_resolver.cc:1690
    -> Item_cond::fix_fields item_cmpfunc.cc:5505
      -> Item_subselect::fix_fields item_subselect.cc:547
        -> SubqueryWithResult::prepare item_subselect.cc:2971
          -> Query_expression::prepare sql_union.cc:753
            -> Query_block::prepare sql_resolver.cc:438
              -> opt_trace_print_expanded_query opt_trace2server.cc:271
```

Q4 的 exists 子查询位于 `WHERE` 子句中，它的 Query_block 是在外层 Query_block 的 `setup_conds` 阶段被 prepare 的，调用入口是：

```cpp
// Set up join conditions and WHERE clause
if (setup_conds(thd)) return true;
```

### opt_trace_print_expanded_query

```txt
Query_block::prepare sql_resolver.cc:438
  -> opt_trace_print_expanded_query opt_trace2server.cc:271
```

处理完 WHERE 字句后，会依次处理其他 Query_block 的各个部分。当处理完当前 Query_block 的所有子查询后，通过 `opt_trace_print_expanded_query()` 将 expanded query 打印到 optimizer trace 中，即第 2 步的 "expanded_query" 中所看到的 SQL。从结果来看，exists 子查询仍然没有被改写，里面的关联表达式 ``LINEITEM`.`L_ORDERKEY` = `ORDERS`.`O_ORDERKEY`` 也仍然存在：

```sql
/* select#1 */ select `ORDERS`.`O_ORDERPRIORITY` AS `O_ORDERPRIORITY`,count(0) AS `ORDER_COUNT` from `ORDERS` where ((`ORDERS`.`O_ORDERDATE` >= DATE'1993-12-01') and (`ORDERS`.`O_ORDERDATE` < (DATE'1993-12-01' + interval '3' month)) and exists(/* select#2 */ select 1 from `LINEITEM` where ((`LINEITEM`.`L_ORDERKEY` = `ORDERS`.`O_ORDERKEY`) and (`LINEITEM`.`L_COMMITDATE` < `LINEITEM`.`L_RECEIPTDATE`)))) group by `ORDERS`.`O_ORDERPRIORITY` order by `ORDERS`.`O_ORDERPRIORITY`
```

### flatten_subqueries

```txt
Query_block::prepare sql_resolver.cc:548
  -> Query_block::flatten_subqueries sql_resolver.cc:3951
    -> Query_block::convert_subquery_to_semijoin sql_resolver.cc:2968
```

`Query_block::prepare()` 中进行 semi join 改写的接口调用如上所示，exists 子查询在之后的 `flatten_subqueries()` 中被改写成了 semi join，其对应的 optimizer trace 记录在了 `"transformation_to_semi_join"` 这个 json object 中。本文不详细分析子查询的改写过程，知道这个入口就行。

```cpp
if (has_sj_candidates() && flatten_subqueries(thd)) return true;
```

### apply_local_transforms

```txt
Query_block::prepare sql_resolver.cc:592
  -> Query_block::apply_local_transforms sql_resolver.cc:767
    -> Query_block::simplify_joins sql_resolver.cc:2168
```

`apply_local_transforms()` 由最外层的 Query_block 发起，然后递归的对内层 Query_block 调用 apply_local_transforms() 完成所有 Query_block 的本地优化，是一个 top-down 的优化过程，包含了一些常见的 top-down 的优化，比如 column pruning、partition pruning、outer join 转 inner join、ONLY_FULL_GROUPBY validation、predicate pushdown 等。

`simplify_joins()` 主要的功能是 outer join 转成 inner join，为后续 join reorder 提供更多可能性，它总共包含了 4 种可能的 transformation：OUTER_JOIN_TO_INNER、JOIN_COND_TO_WHERE、PAREN_REMOVAL、SEMIJOIN，optimizer trace 中看到的第 4 个 step 就对应了 SEMIJOIN，如果还有其他类型的转化发生，也会一并记录在 `"transformations"` 这个数组里。

## 阶段 2：join_optimize

join_optimize 的入口是 `JOIN::optimize()`，它驱动了所有子过程。接下来我们将看到 Q4 经历的各个 join_optimize 子过程。

### optimize_cond

```txt
JOIN::optimize sql_optimizer.cc:472
  -> optimize_cond sql_optimizer.cc:10250
```

optimize_cond() 主要优化查询的 where 和 having 条件，对应 optimizer trace 中的 "condition_processing" 部分。在 optimize_cond() 中，主要完成以下优化：
1. equality propagation，利用 x=y, y=z 推导出 x=y=z，将普通的二元等式转换为多元等式 (x, y, z, ...)
2. constant propagation，在 equality propagation 完成后，进行常量传播，例如根据 x=42 推导出所有多元等式 (x, y, z, ...) 中的字段都等于 42
3. trivial condition removal，在常量传播后，消除所有始终为真或始终为假的 predicate

### substitute_gc

```txt
JOIN::optimize sql_optimizer.cc:599
  -> substitute_gc sql_optimizer.cc:1192
```

substitute_gc 主要检查 query 的 where 条件、order by 等，将其中的表达式替换为匹配的 generated column。尽管 optimizer trace 中提到了这一信息，但由于 Q4 中涉及的 LINEITEM 和 ORDERS 表都没有 generated column，这个优化并未生效。

### make_join_plan

```txt
JOIN::optimize sql_optimizer.cc:694
  -> JOIN::make_join_plan sql_optimizer.cc:5284
    -> trace_table_dependencies sql_optimizer.cc:6246
```

MySQL 8.0 新增的 hypergraph join order 算法默认关闭。下面这些信息在 optimizer trace 中属于老的 join order 算法：
1. table_dependencies
2. ref_optimizer_key_uses
3. pulled_out_semijoin_tables
4. rows_estimation
5. execution_plan_for_potential_materialization
6. considered_execution_plans

MySQL 首先会在 "rows_estimation" 中估算每个基表和 JOIN 的基数，并在后续的优化过程中持续计算每个候选执行计划的 cost。在 "considered_execution_plans" 中，我们可以看到所有候选执行计划及其 cost。

optimizer trace 中有许多由 make_join_plan 暴露的详细信息，对于我们进一步了解 join order、access path 以及 cost estimation 非常有帮助。由于篇幅所限，我们在此不再详细展开，将来有机会的话再单独介绍这个优化过程。

### make_join_query_block

```txt
JOIN::optimize sql_optimizer.cc:779
  -> make_join_query_block sql_optimizer.cc:9577
```

make_join_query_block 是 MySQL 在完成 join order 后的一个优化过程，其主要目的是尽早过滤掉不需要的中间结果，将包括 join 的 on condition 在内的所有 predicate 尽可能地下推。optimizer trace 中的 "attaching_conditions_to_tables" 反映了这一过程中下推到各个表上的 predicate。

### optimize_distinct_group_order

```txt
JOIN::optimize sql_optimizer.cc:794
  -> JOIN::optimize_distinct_group_order sql_optimizer.cc:1464
```

完成 join 相关的优化后，MySQL 在 optimize_distinct_group_order() 中继续对 distinct、group by 和 order by 进行优化，例如将 distinct 转换为 group by，消除不必要的 trivial order by 等，这里面的细节也非常多，对应了 optimizer trace 中的 "optimizing_distinct_group_by_order_by" 部分。

### finalize_table_conditions

```txt
JOIN::optimize sql_optimizer.cc:1015
  -> JOIN::finalize_table_conditions sql_optimizer.cc:9108
```

进行最后一轮的 condition 优化，这一步主要是去除冗余的 filter，将缓存表达式中的常量，避免每一行数据都重新计算等，对应了 optimizer trace 中的 "finalizing_table_conditions" 部分。

### make_join_readinfo

```txt
JOIN::optimize sql_optimizer.cc:1018
  -> make_join_readinfo sql_select.cc:3112
```

做执行前的 plan 调整比如分配 join buffer，里面的内容比较杂，对应了 optimizer trace 中的 "refine_plan" 部分。

### make_tmp_tables_info

```txt
JOIN::optimize sql_optimizer.cc:1021
  -> JOIN::make_tmp_tables_info sql_select.cc:4219
```

这是 MySQL 查询优化的最后一步，为执行计划中各个 SQL 算子按需分配 tmp table，对应 optimizer trace 的 "considering_tmp_tables" 部分。

## 阶段 3：join_execution

比较有意思的是 MySQL optimizer trace 中还记录了查询执行过程的一些信息，例如 "temp_table_aggregate"，它的调用链路如下，从这可以看到 MySQL 查询执行的驱动过程，这部分内容我们不再展开：

```txt
Sql_cmd_dml::execute_inner sql_select.cc:799
  -> Query_expression::execute sql_union.cc:1823
    -> Query_expression::ExecuteIteratorQuery sql_union.cc:1763
      -> SortingIterator::Init sorting_iterator.cc:444
        -> SortingIterator::DoSort sorting_iterator.cc:531
          -> filesort filesort.cc:408
            -> TemptableAggregateIterator<DummyIteratorProfiler>::Init composite_iterators.cc:1680
```

## 总结

MySQL 8.0.31 在 optimizer trace 中新增了查询优化和执行的信息，特别是 make_join_plan 的细节。结合 MySQL 代码，optimizer trace 对我们理解执行计划生成过程非常有帮助。基于 MySQL 二次开发也可以扩展 optimizer trace 提升系统的可观测性。