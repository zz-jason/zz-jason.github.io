---
title: "[MySQL 8.0] 为 MySQL 添加新的存储引擎插件"
date: 2023-07-02T00:00:00Z
categories: ["MySQL", "Storage"]
draft: true
---
![](posts/mysql-new-storage/featured.jpg)

## 简介

虽然 MySQL 默认采用 InnoDB 存储引擎，但它提供的插件机制使我们可以根据需要开发新的存储引擎，这篇文章我们利用 MySQL 插件机制实现一个存储引擎 demo，实现 insert、update、delete 以及 select 这 4 个功能。

## 编译和安装

```sh
cmake -S . -B build/debug \
      -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=1 \
      -DWITH_LEAN_STORAGE_ENGINE=1 \
      -DWITH_EXAMPLE_STORAGE_ENGINE=1 \
      -DDOWNLOAD_BOOST=1 \
      -DWITH_BOOST=build \
      -DCMAKE_INSTALL_PREFIX=build/install

cmake --build build/debug -j `nproc`
cmake --install build/debug
```

为即将启动的 mysql 实力创建一个 my.cnf 配置文件，内容如下。为了方便管理，将这个 my.cnf 文件也一并放入 `$MYSQL_HOME/deploy/instance-1` 目录中：

```txt
[mysqld]
datadir=deploy/instance-1/data
```

初始化实例：

```sh
$MYSQL_HOME/build/install/bin/mysqld \
    --defaults-file=$MYSQL_HOME/deploy/instance-1/my.cnf \
    --user=root \
    --initialize-insecure
```

之后只需要执行下面的命令即可启动 mysqld 进程：

```sh
$MYSQL_HOME/build/install/bin/mysqld \
    --defaults-file=$MYSQL_HOME/deploy/instance-1/my.cnf \
    --user=root
```

使用编译的 mysql 客户端连接该实例：

```sh
$MYSQL_HOME/build/install/bin/mysql --prompt "MySQL (\U) > " -h 127.0.0.1 -P 3306 -u root
```

## 从 EXAMPLE 出发

MySQL codebase 的 storage/example 中包含了一个添加存储引擎的样例，我们可以先通过这个样例简单了解 MySQL 存储引擎插件。先执行几个简单的测试 SQL：

```sql
create table t(a bigint, b bigint) engine=example;
insert into t values(1, 1);
```

很不幸，在执行 insert 语句时我们遇到了这样的报错：

```txt
ERROR 1662 (HY000): Cannot execute statement: impossible to write to binary log since BINLOG_FORMAT = ROW and at least one table uses a storage engine limited to statement-based logging.
```

没关系，我们接下来拷贝一份 example 目录到 storage/leanstore 中，然后修改存储引擎的名字为 `leanstore`，接着为 leanstore 支持 insert 语句。

## 支持 insert

这个章节的目标是跑通下面的 insert 语句：

```sql
drop table if exists t;
create table t(a bigint, b bigint) engine=leanstore;
insert into t values(1, 1);
```

从报错信息来看是因为 “at least one table uses a storage engine limited to statement-based logging”，这个需要修改 `table_flags()` 的范围值，为其加上 `HA_BINLOG_ROW_CAPABLE`：

```cpp
  /// @brief This is a list of flags that indicate what functionality the
  /// storage engine implements. The current table flags are documented in
  /// handler.h
  ulonglong table_flags() const override {
    return HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE;
  }
```

接下来再执行  insert 语句就能成功了，但要完整的支持 insert 把数据存下来，还需要实现 `write_row()` 接口，example 中并没有对 mysql 传入的数据做任何处理。