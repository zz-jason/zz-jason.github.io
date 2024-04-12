---
title: "[MongoDB] 存储引擎接口"
date: 2024-03-19T00:00:00Z
categories: ["MongoDB", "Storage"]
draft: true
---
![](posts/mongo-storage-api/featured.jpg)

## How to compile

编译 mongo 4.0，依赖  pytyhon 2.7，环境准备好后，执行下面命令：

```sh
python2 buildscripts/scons.py all compiledb --disable-warnings-as-errors -j32 --dbg=on
```

## How to run and debug mongo

Run a mongo server:

```sh
mkdir -p /tmp/mongodb
nohup ./mongod --dbpath /tmp/mongodb --port 27017 > /tmp/mongodb/nohup.out 2>&1
```

Connect to it and run a query, see more in https://www.mongodb.com/docs/v4.4/mongo/#working-with-the-mongo-shell:

```sh
./mongo --host 127.0.0.1 --port 27017
```

In the running mongo shell:
```
db.myCollection.insertOne({x:1});
db.myCollection.count()
db.myCollection.find({x:1})
```

## 一些概念

`KVEngine`，定义在 src/mongo/db/storage/kv/kv_engine.h 中，代表了一个存储引擎实现，比如 WiredTiger，主要作用是创建、查找 RecordStore 等。

`RecordStore`，定义在 src/mongo/db/storage/record_store.h 中，一个 RecordStore 代表了一个 Collection，或者 Collection 的索引（也是 KV 结构），主要用于 Collection 中 Document 的 CURD 操作。

参考：
- [MongoDB mongorocks 引擎原理解析](https://zhuanlan.zhihu.com/p/414821545)
- 