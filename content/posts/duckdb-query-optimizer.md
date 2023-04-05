---
title: "[Duckdb] Query Optimizer"
date: 2023-02-19T10:46:14Z
categories: ["DuckDB"]
draft: true
---

## ClientContext::CreatePreparedStatement

驱动 planner 和 optimizer 将 ast 转换成 logical plan 并优化成 physical plan

## Planner::CreatePlan

从 AST 转换成 logical plan 是在 `Planner::CreatePlan(unique_ptr<SQLStatement> statement)` 这个函数中完成的。当我们执行下面这条语句时，函数的调用堆栈如下：

```gdb
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x000000010a4d15ab duckdb`duckdb::Planner::CreatePlan(this=0x00007ff7b6cc3950, statement=duckdb::SQLStatement @ 0x000060b000003060) at planner.cpp:112:10
    frame #1: 0x00000001107ed9e6 duckdb`duckdb::ClientContext::CreatePreparedStatement(this=0x00006150000249a0, lock=0x000060200001ab70, query="select t.a, count(*) as cnt from t, s where t.a>s.a group by t.a having cnt > 1 order by cnt desc;", statement=nullptr, values=0x0000000000000000) at client_context.cpp:325:10
    frame #2: 0x0000000110b3dc2f duckdb`duckdb::ClientContext::PrepareInternal(this=0x000060400006c698)::$_1::operator()() const at client_context.cpp:526:36
    frame #3: 0x0000000110b3d8a1 duckdb`decltype(__f=0x000060400006c698)::$_1&>(fp)()) std::__1::__invoke<duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1&>(duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1&) at type_traits:3918:1
    frame #4: 0x0000000110b3d75d duckdb`void std::__1::__invoke_void_return_wrapper<void, true>::__call<duckdb::ClientContext::PrepareInternal(__args=0x000060400006c698)::$_1&>(duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1&) at invoke.h:61:9
    frame #5: 0x0000000110b3d6c5 duckdb`std::__1::__function::__alloc_func<duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1, std::__1::allocator<duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1>, void ()>::operator(this=0x000060400006c698)() at function.h:178:16
    frame #6: 0x0000000110b37d79 duckdb`std::__1::__function::__func<duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1, std::__1::allocator<duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock&, std::__1::unique_ptr<duckdb::SQLStatement, std::__1::default_delete<duckdb::SQLStatement> >)::$_1>, void ()>::operator(this=0x000060400006c690)() at function.h:352:12
    frame #7: 0x0000000110b56054 duckdb`std::__1::__function::__value_func<void ()>::operator(this=0x00007ff7b6cc47f0)() const at function.h:505:16
    frame #8: 0x00000001108162d9 duckdb`std::__1::function<void ()>::operator(this=0x00007ff7b6cc47f0)() const at function.h:1182:12
    frame #9: 0x00000001107fd3cf duckdb`duckdb::ClientContext::RunFunctionInTransactionInternal(this=0x00006150000249a0, lock=0x000060200001ab70, fun=0x00007ff7b6cc47f0, requires_valid_transaction=false)> const&, bool) at client_context.cpp:942:3
    frame #10: 0x00000001107feb77 duckdb`duckdb::ClientContext::PrepareInternal(this=0x00006150000249a0, lock=0x000060200001ab70, statement=nullptr) at client_context.cpp:525:2
    frame #11: 0x00000001107fffc7 duckdb`duckdb::ClientContext::Prepare(this=0x00006150000249a0, statement=nullptr) at client_context.cpp:537:10
    frame #12: 0x00000001108330d0 duckdb`duckdb::Connection::Prepare(this=0x000060300002de80, statement=nullptr) at connection.cpp:127:18
    frame #13: 0x000000010938a0f1 duckdb`::duckdb_shell_sqlite3_prepare_v2(db=0x0000608000000120, zSql="select t.a, count(*) as cnt from t, s where t.a>s.a group by t.a having cnt > 1 order by cnt desc;", nByte=-1, ppStmt=0x00007ff7b6cc56a0, pzTail=0x00007ff7b6cc56c0) at sqlite3_api_wrapper.cpp:191:28
    frame #14: 0x000000010928e703 duckdb`shell_exec(pArg=0x00007ff7b6cc61a0, zSql="select t.a, count(*) as cnt from t, s where t.a>s.a group by t.a having cnt > 1 order by cnt desc;", pzErrMsg=0x00007ff7b6cc5b40) at shell.c:13136:10
    frame #15: 0x0000000109335798 duckdb`runOneSqlLine(p=0x00007ff7b6cc61a0, zSql="select t.a, count(*) as cnt from t, s where t.a>s.a group by t.a having cnt > 1 order by cnt desc;", in=0x0000000000000000, startline=1) at shell.c:20050:8
    frame #16: 0x0000000109292b49 duckdb`process_input(p=0x00007ff7b6cc61a0) at shell.c:20168:17
    frame #17: 0x000000010925984d duckdb`main(argc=2, argv=0x00007ff7b6cc72b0) at shell.c:20989:12
    frame #18: 0x00007ff807390310 dyld`start + 2432
```

从函数的调用堆栈可以看到，statement 的 plan 和 optimize 是由 `ClientContext::CreatePreparedStatement`驱动的，在 325 行调用 Planner::CreatePlan 拿到初始的 logical plan 后，在第 346 行会接着调用 `Optimizer::Optimize` 进行查询优化。



## Optimizer::Optimize

optimize 中定义了一系列的优化规则，优化方式比较简单从前到后 apply 一遍这些优化规则：

```cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p) {
	Verify(*plan_p);
	this->plan = std::move(plan_p);
  // first we perform expression rewrites using the ExpressionRewriter
	// this does not change the logical plan structure, but only simplifies the expression trees
	RunOptimizer(OptimizerType::EXPRESSION_REWRITER, [&]() { rewriter.VisitOperator(*plan); });

	// perform filter pullup
	RunOptimizer(OptimizerType::FILTER_PULLUP, [&]() {
		FilterPullup filter_pullup;
		plan = filter_pullup.Rewrite(std::move(plan));
	});

	// perform filter pushdown
	RunOptimizer(OptimizerType::FILTER_PUSHDOWN, [&]() {
		FilterPushdown filter_pushdown(*this);
		plan = filter_pushdown.Rewrite(std::move(plan));
	});

  ...
```

篇幅所限这里不一一例举。这里的思路和大多数优化器对 logical plan 的 rewrite 过程很像，和 tidb 的 logical optimization 也是一样的思路：每个 rewrite rule 都一定是有收益的，rewrite rule 之间的先后顺序需要经过精细调优，固化在优化器代码中。duckdb 中这一阶段需要进行的 rewirte rule 和它们之间的先后关系是：

1. EXPRESSION_REWRITER
2. FILTER_PULLUP
3. FILTER_PUSHDOWN
4. REGEX_RANGE
5. IN_CLAUSE
6. JOIN_ORDER
7. DELIMINATOR
8. UNUSED_COLUMNS
9. STATISTICS_PROPAGATION
10. COMMON_SUBEXPRESSIONS
11. COMMON_AGGREGATE
12. COLUMN_LIFETIME
13. TOP_N
14. REORDER_FILTER



以上是目前 duckdb 内置的所有 rewrite rule。除了内置的 rewrite rule 以外，duckdb 还支持自定义 optimize rule，在 apply 完这些 rewrite rule 后会顺序的 apply 这些用户自定义的 extension rewrite rule：

```cpp
  ...
  // apply simple expression heuristics to get an initial reordering
	RunOptimizer(OptimizerType::REORDER_FILTER, [&]() {
		ExpressionHeuristics expression_heuristics(*this);
		plan = expression_heuristics.Rewrite(std::move(plan));
	});

	for (auto &optimizer_extension : DBConfig::GetConfig(context).optimizer_extensions) {
		RunOptimizer(OptimizerType::EXTENSION, [&]() {
			optimizer_extension.optimize_function(context, optimizer_extension.optimizer_info.get(), plan);
		});
	}

	Planner::VerifyPlan(context, plan);

	return std::move(plan);
}
```

## PhysicalPlanGenerator::CreatePlan

这是一个启发式的物理优化过程，没有记忆化搜索的动态规划。整个 physical plan 的构造过程是自下而上的。先得到 child 的 plysical plan 然后再构造自己对应的 physical plan。在构造 physical plan 过程中也会去计算 estimated cardinality，在构造当前 physical plan 时会根据 child 的 cardinality 进行启发式的优化。比如 `PhysicalPlanGenerator::CreatePlan(LogicalComparisonJoin &op)`  是这样判断是否采用 IndexJoin 的：

```cpp
	if (has_equality) {
		Index *left_index {}, *right_index {};
		TransformIndexJoin(context, op, &left_index, &right_index, left.get(), right.get());
		if (left_index &&
		    (ClientConfig::GetConfig(context).force_index_join || rhs_cardinality < 0.01 * lhs_cardinality)) {
			auto &tbl_scan = (PhysicalTableScan &)*left;
			swap(op.conditions[0].left, op.conditions[0].right);
			return make_unique<PhysicalIndexJoin>(op, std::move(right), std::move(left), std::move(op.conditions),
			                                      op.join_type, op.right_projection_map, op.left_projection_map,
			                                      tbl_scan.column_ids, left_index, false, op.estimated_cardinality);
		}
		if (right_index &&
		    (ClientConfig::GetConfig(context).force_index_join || lhs_cardinality < 0.01 * rhs_cardinality)) {
			auto &tbl_scan = (PhysicalTableScan &)*right;
			return make_unique<PhysicalIndexJoin>(op, std::move(left), std::move(right), std::move(op.conditions),
			                                      op.join_type, op.left_projection_map, op.right_projection_map,
			                                      tbl_scan.column_ids, right_index, true, op.estimated_cardinality);
		}
    ...
```

这里没有 physical property 概念，也没有复杂的 cost model，理解起来相对简单。经过 PhysicalPlanGenerator::CreatePlan() 后就得到了一个单机版的物理执行计划，后续就交给 DuckDB push-based execution engine 进行计算了。
