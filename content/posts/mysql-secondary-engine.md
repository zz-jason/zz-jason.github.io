---
title: "[MySQL] Secondary Engine 源码阅读"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Summary

几个关注的问题：

DDL & Meta data

- 如何创建 secondary engine table，元数据如何存储
- 数据如何从 primary engine（innodb）同步到 secondary engine
- 如何修改表结构
- 如何删除 secondary engine table

DML & Transacton

- primary engine（innodb）的事务读写如何同步到 secondary engine
- secondary engine 表现出来的事务隔离级别是什么

DQL

- 如何 secondary engine 中优化和执行 select statement
- 如何根据 cost 判断是采用 primary engine 执行还是 secondary engine 执行
- secondary engine 的执行结果如何返回给 mysql client

Diagnose

- secondary engine 有哪些系统变量、系统表，分别什么含义，如何维护
- secondary engine 如何排查慢查询慢在哪
- secondary engine 是否支持 explain analyze，是否支持 trace

## The Secondary Engine Framework

### Register

Data

## Query Processing on Secondary Engine

### Optimize query on secondary engine

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

1. SQL 执行时如何处理 secondary engine，计算是否可以下推？

### Execute query on secondary engine

### Collect results from secondary engine

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
