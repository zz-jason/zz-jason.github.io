---
title: "[MySQL] Secondary Engine 源码阅读 3：Secondary Engine 故障诊断"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Secondary Engine 故障诊断和行为控制

### Session/System Variables

MySQL 为每个插件提供了添加 Session/System Variable 的接口，Secondary Engine 也是一个插件，可以根据需要实现定义各种类型的变量，方便用户控制 Secondary Engine 内部的各种行为。mock Secondary Engine 中没有这样的例子，不过我们在其他插件中可以轻松找到它们，比如在 auth_null 这个插件中，我们看到它是这样自定义 Session/System 变量的：

```cpp
mysql_declare_plugin(audit_null){
    MYSQL_AUDIT_PLUGIN,     /* type                            */
    &audit_null_descriptor, /* descriptor                      */
    "NULL_AUDIT",           /* name                            */
    PLUGIN_AUTHOR_ORACLE,   /* author                          */
    "Simple NULL Audit",    /* description                     */
    PLUGIN_LICENSE_GPL,
    audit_null_plugin_init,   /* init function (when loaded)     */
    nullptr,                  /* check uninstall function        */
    audit_null_plugin_deinit, /* deinit function (when unloaded) */
    0x0003,                   /* version                         */
    simple_status,            /* status variables                */
    system_variables,         /* system variables                */
    nullptr,
    0,
} mysql_declare_plugin_end;
```

`mysql_declare_plugin` 这个宏里面倒数第 4 和 3 个参数就是插件自定义的变量。插件定义一个变量的方法也比较简单，在不少插件实现中可以找到参考代码，在 MySQL 开发文档中也有比较详细的说明和例子，这里我们不再展开，感兴趣的朋友可以参考这篇文档：[4.4.2.2 Server Plugin Status and System Variables](https://dev.mysql.com/doc/extending-mysql/8.0/en/plugin-status-system-variables.html)

### PERFORMANCE_SCHEMA tables

MySQL 提供了实现 PERFORMANCE_SCHEMA plugin 的接口，使得开发者可以根据需要自定义需要的内存表放到 PERFORMANCE_SCHEMA 中，比如 Secondary Engine 如果有需要也可以实现这个接口自定义一些表。MySQL 提供了 **pfs_table_plugin** 这个例子，感兴趣的朋友可以详细阅读参考，这里不再展开。

