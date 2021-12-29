---
layout: post
title:  'Hive Update、Delete操作配置'
date:   2018-11-13
tags: [hadoop,hive]
categories: [大数据框架,hive]
---



### Hive Update、Delete操作配置

####  条件

- 只支持ORC存储格式
- 表必须分桶
- 更新指定配置文件

#### 创建存储为ORC的分桶表

```sql
CREATE TABLE table_name (
  id                int,
  name              string
)
CLUSTERED BY (id) INTO 2 BUCKETS STORED AS ORC
TBLPROPERTIES ("transactional"="true");
```

#### 修改配置文件

修改添加hive中的hive-site.xml配置文件

```xml
<property>
    <name>hive.support.concurrency</name>
    <value>true</value>
</property>
<property>
    <name>hive.enforce.bucketing</name>
    <value>true</value>
</property>
<property>
    <name>hive.exec.dynamic.partition.mode</name>
    <value>nonstrict</value>
</property>
<property>
    <name>hive.txn.manager</name>
    <value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
</property>
<property>
    <name>hive.compactor.initiator.on</name>
    <value>true</value>
</property>
```

如果是CDH，如图所示

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2018-12-13/hive-conf.png)

如果上述配置文件不生效，则在`hive-site.xml 的 Hive Metastore Server 高级配置代码段（安全阀）`与上述相同的配置