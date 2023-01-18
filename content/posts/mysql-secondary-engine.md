---
title: "[MySQL] Secondary Engine 源码阅读"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Summary


## DDL & Meta data

- 如何创建 secondary engine table，元数据如何存储
- 数据如何从 primary engine（innodb）同步到 secondary engine
- 如何修改表结构
- 如何删除 secondary engine table

对于新表：

创建一个有 secondary_engine 的 table，secondary_engine 为 rapid（heatwave）：

```sql
CREATE TABLE orders (id INT) SECONDARY_ENGINE = RAPID;
```

实测的时候，如果没有 load secondary engine 的 plugin，建表语句也能成功，show create table 也能显示出该表拥有 secondary_engine 属性。

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
- Sql_cmd_secondary_load_unload::execute：做基本的权限检查
- Sql_cmd_secondary_load_unload::mysql_secondary_load_or_unload：设置 MDL、降级事务隔离级别为 RC 等，为数据 unload 到 secondary 做准备
- secondary_engine_load_table：根据 secondary_engine 的名字（rapid）寻找对应的 plugin，plugin 需要也实现 “ha_resolve_by_name” 接口。上面报错的原因是根据 secondary engine 的名字找不到对应的 plugin，于是报错了 “ERROR 1286 (42000): Unknown storage engine 'RAPID'”，如果改成 “secondary engine plugin is not found” 之类的可能会更友好些。

后续的数据在发生 DML 后会自动从 innodb 同步到 rapid 这个 secondary_engine 中

参考：
- [2.2.2 Loading Data Manually](https://dev.mysql.com/doc/heatwave/en/heatwave-loading-data-manually.html)

## DML & Transacton

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
