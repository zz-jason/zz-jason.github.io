---
title: "重构 EXPLAIN"
date: 2017-08-13T11:06:51+08:00
categories: ["TiDB", "Query Optimization"]
---

explain 的用途非常广泛，比如用来查看某个表的信息，查看执行计划等等。explain 的语法可以参考 mysql 文档：[EXPLAIN Syntax](https://dev.mysql.com/doc/refman/5.7/en/explain.html)，另外 EXPLAIN, DESCRIBE, DESC 这几个关键字是等效的，所以很多时候我会直接使用 desc 而不是 explain

explain 这个东西，虽然各家都支持，但因为各自内部 planner 的实现不一样，导致 explain 结果的展现形式和内容各有区别。TiDB 在做 explain 的时候就不去考虑兼容 MySQL 了

还是先讲讲为什么重构吧。主要原因还是 planner 经过了一轮重构后，新的 planner 不支持 explain。但是新老 planner 会共享一些 operator，比如 Projection，Selection，Limit 等。对于这些复用的 operator，可以称之为重构，对于那些新增的 operator，可以说是支持。anyway，我们的目的是给新 planner 支持 explain，以方便 TiDB 进行性能调优或者 debug

之前实现 explain 的时候，采用的方法是给每个 operator 实现 json.Marsharl() 接口，但是我想把这个接口留着，用来打印详细的 operator 的 log 信息。为什么要这样做呢，其实现有的 explain 结果并不能很精确的定位问题，比如我们 explain 并没有展现每个 operator 的 schema 信息，也没有展现每个表达式返回结果的类型信息等。如果 plan 做错了，我们还是很难有最直观的证据说这个 plan 确实是做错了，得去 debug 打印日志来观察和推断

在这里 [#3809](https://github.com/pingcap/tidb/pull/3809) 我重新构建了新 planner 的 explain 的框架，在接下来的几个 pr 中：[#3883](https://github.com/pingcap/tidb/pull/3883) [#3915](https://github.com/pingcap/tidb/pull/3915) [#3953](https://github.com/pingcap/tidb/pull/3953)，分别为各个 operator 实现了 ExplainInfo 的生成。Explain 重构完成后，主要有如下几个变化：

不再使用 json.Marsharl()，而是使用新增的 ExplainInfo() 接口
构造 explain 结果的地方挪到了 planner，而不是之前的 executor 中
结果显示上，新的 explain 结果在格式上更加方便查看，不容易存在跨行情况了
重构之前的 explain 结果也是一个表格，但是会有三列，分别是 id，json，parent，其中 json 列是一个格式化好的 json 字符串，包括各种换行符制表符等等，显示上看起来会比较费劲。重构完成后，一条 SQL 的 explain 结果长这样：

```sql
TiDB > EXPLAIN SELECT a.col2, a.col3, a.col4, a.col5 FROM tblA a, tblB b WHERE a.col1=b.col2 AND b.col1=4;
+----------------+--------------+-------------------------------+------+----------------------------------------------------------+-------+
| id             | parents      | children                      | task | operator info                                            | count |
+----------------+--------------+-------------------------------+------+----------------------------------------------------------+-------+
| TableScan_10   | Selection_11 |                               | cop  | table:b, range:(-inf,+inf), keep order:false             |   0.8 |
| Selection_11   |              | TableScan_10                  | cop  | eq(b.col1, 4)                                            |   0.8 |
| TableReader_12 | IndexJoin_6  |                               | root | data:Selection_11                                        |   0.8 |
| TableScan_9    |              |                               | cop  | table:a, range:(-inf,+inf), keep order:true              |     1 |
| TableReader_13 | IndexJoin_6  |                               | root | data:TableScan_9                                         |     1 |
| IndexJoin_6    | Projection_5 | TableReader_12,TableReader_13 | root | outer:TableReader_13, outer key:b.col2, inner key:a.col1 |   0.8 |
| Projection_5   |              | IndexJoin_6                   | root | a.col2, a.col3, a.col4, a.col5                           |   0.8 |
+----------------+--------------+-------------------------------+------+----------------------------------------------------------+-------+
```

上面这条 SQL 来自 issue [#3914](https://github.com/pingcap/tidb/pull/3914)，里面有完整的建表语句，感兴趣的读者可以试试

大家可能在上面那个 issue 的 comment 中发现我贴的 dot 图了。这也是 explain 未来可以展示的结果之一，比如可以丰富 explain 的语法，以后直接 `explain format=dot select ...` 得到 plan 对应的 dot 文件的内容，然后上 [webgraphviz](http://www.webgraphviz.com/) 生成 dot 图；或者把结果存文件，用本机安装的 dot 命令生成 dot 图（`dot -T png -O xxx.dot`）


