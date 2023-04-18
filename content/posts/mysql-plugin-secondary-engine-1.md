---
title: "[MySQL] Secondary Engine 源码阅读 1：Secondary Engine 插件简介"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

> 本系列文章以 MySQL Secondary Engine 源码为例，探讨了其插件机制、读写流程、故障诊断、DDL操作等方面的内容，或许能对读者有所参考。如果感兴趣，请随意阅读以下列表：
> 
> 1. [MySQL] Secondary Engine 源码阅读 1：Secondary Engine 插件简介
> 2. [MySQL] Secondary Engine 源码阅读 2：Secondary Engine 读流程
> 3. [MySQL] Secondary Engine 源码阅读 3：Secondary Engine 故障诊断
> 4. [MySQL] Secondary Engine 源码阅读 4：Secondary Engine DDL
> 5. [MySQL] Secondary Engine 源码阅读 5：Secondary Engine 总结

## Summary

* What is a secondary engine
* MySQL plugin fragmework
* The mock secondary engine plugin
* Build, install, and debug the mock secondary engine

## 1. What is a secondary engine

Demo：初步体验 MySQL Secondary Engine

## 2. MySQL plugin fragmework

MySQL 有着强大的 Plugin 机制，比如审计日志、查询改写，链接管理等，在 MySQL 源码中的 plugin 目录中我们能看到非常多的 plugin 实现样例，而 Secondary Engine 也是通过 MySQL 的 Plugin 机制实现的。

实现一个插件，只需要使用 `mysql_declare_plugin` 这个宏，定义好插件的名字，以及相关接口的函数指针即可.


## 3. The mock secondary engine plugin

后面的所有代码研究都会基于 mock secondary engine 进行，它的插件定义是这样的：

```cpp
mysql_declare_plugin(mock){
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &mock_storage_engine,
    "MOCK",
    PLUGIN_AUTHOR_ORACLE,
    "Mock storage engine",
    PLUGIN_LICENSE_GPL,
    Init,
    nullptr,
    Deinit,
    0x0001,
    nullptr,
    nullptr,
    nullptr,
    0,
} mysql_declare_plugin_end;
```

- [ ] TODO：简要说明插件接口中几个关键函数的作用

MySQL 源码中有一个 mock 的 Secondary Engine 实现，它位于 storage/secondary_engine_mock/ha_mock.cc 中，从代码注释来看这个 mock 的 Secondary Engine 主要是用来方便测试使用：

```cpp
/**
 * The MOCK storage engine is used for testing MySQL server functionality
 * related to secondary storage engines.
 *
 * There are currently no secondary storage engines mature enough to be merged
 * into mysql-trunk. Therefore, this bare-minimum storage engine, with no
 * actual functionality and implementing only the absolutely necessary handler
 * interfaces to allow setting it as a secondary engine of a table, was created
 * to facilitate pushing MySQL server code changes to mysql-trunk with test
 * coverage without depending on ongoing work of other storage engines.
 *
 * @note This mock storage engine does not support being set as a primary
 * storage engine.
 */
```


## 4. Build, install, and debug the mock secondary engine

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
                         --initialize-insecure
```

Start MySQL instance after initialization:

```sh
build/install/bin/mysqld --defaults-file=my.cnf --user=root
```

Connect to the server:

```sh
build/install/bin/mysql -u root
```
## References

- [MySQL HeatWave User Guide](https://dev.mysql.com/doc/heatwave/en/heatwave-introduction.html)
- [MySQL · 源码阅读 · Secondary Engine](http://mysql.taobao.org/monthly/2020/11/04/)