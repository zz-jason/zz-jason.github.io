---
title: "[MySQL] Secondary Engine 源码阅读 4：Secondary Engine DDL"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

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



