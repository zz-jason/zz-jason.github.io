---
title: "[MySQL] Secondary Engine 源码阅读"
date: 2022-11-15T03:06:19Z
categories: ["MySQL"]
draft: true
---

## Summary

几个关注的问题：

1. 如何注册 Secondary Engine

1. 如何同步数据到 Secondary Engine

1. 如何读取 Secondary Engine 的数据

从 query processing 的视角来看，几个需要关注的细节

1. 如何实现自己的 optimizer

1. 如何判断读 secondary 还是 primary 的数据

1. 计算如何下推 secondary engine 中，结果如何收回来

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
