---
title: 'MongoDB 在集群模式下Count也真实数据量不一致'
date: 2020-11-22
tags: mongodb
categories: [数据库,MongoDB]
---



## 1. 背景

在同步Clickhouse数据时，发现MongoDB数据量与Clickhouse数据量不一致，经同事提醒，可能是分片MongoDB集群Count不一致导致吗，于是Google查询相关资料

## 2.相关信息

通过查看[官网](https://docs.mongodb.com/manual/reference/method/db.collection.count/)发现中有解释这种现象的解释

*   On a sharded cluster, [`db.collection.count()`](https://docs.mongodb.com/manual/reference/method/db.collection.count/#db.collection.count "db.collection.count()") can result in an _inaccurate_ count if [orphaned documents](https://docs.mongodb.com/manual/reference/glossary/#term-orphaned-document)exist or if a [chunk migration](https://docs.mongodb.com/manual/core/sharding-balancer-administration/) is in progress.
    
*   To avoid these situations, on a sharded cluster, use the [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group "$group") stage of the [`db.collection.aggregate()`](https://docs.mongodb.com/manual/reference/method/db.collection.aggregate/#db.collection.aggregate "db.collection.aggregate()") method to [`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum "$sum") the documents. For example, the following operation counts the documents in a collection


官方文档解释了这种现象的原因以及解决方法：
**不准确的原因**：

*   **操作的是分片的集合（前提）**；
*   shard 分片正在做块迁移，导致有重复数据出现
*   存在[孤立文档](https://docs.mongodb.com/manual/reference/glossary/#term-orphaned-document)（因为不正常关机、块迁移失败等原因导致）

**解决方法**  
使用**聚合 aggregate** 的方式查询 count 数量，**shell 命令**如下：  

```
db.collection.aggregate(
   [
      { $group: { _id: null, count: { $sum: 1 } } }
   ]
)
```

**java 代码** 
所以在 Java 中也可以采用聚合的方式获取 count 结果，使用**聚合 aggregate** 的方式可以准确的获取 sharding 后的集合的 count 结果。  

```
DBObject groupFields = new BasicDBObject("\_id", null);
groupFields.put("count", new BasicDBObject("$sum", 1));
BasicDBObject group = new BasicDBObject("$group", groupFields);

List<BasicDBObject> aggreList = new ArrayList<BasicDBObject>();
aggreList.add(group);
AggregateIterable<Document> output = collection.aggregate(aggreList);

for (Document dbObject : output)
{
      System.out.println("Aggregates count = "+ dbObject);
}
```

## 参考

- *[MongoDB：count 结果不准确的原因与解决方法](http://he-zhao.cn/2018/03/08/Mongodb-Count/)*

- [官方文档](https://docs.mongodb.com/manual/reference/method/db.collection.count/)