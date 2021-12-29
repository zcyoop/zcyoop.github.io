---
layout: post
title:  'Hive与Hbase之间的区别与关系'
date:   2018-11-17
tags: hbase hive
categories: [大数据框架]
---

#### 1. 区别

1. Hbase：Hadoop database，也就是基于Hadoop的数据库，是一种NoSQL的数据库，主要用于海量数据的实时随机查询，例如：日志明细，交易清单等。
2. Hive： Hive是hadoop的数据仓库，跟数据库有点差，主要是通过SQL语句对HDFS上**结构化**的数据进行计算和处理，适用于离线批量数据处理
   - 通过元数据对HDFS上的数据文件进行描述，也就是通过定义一张表来描述HDFS上的结构化文本，包括各列的数据名称、数据类型，方便数据的处理
   - 基于上面一点，通过SQL来处理和计算HDFS的数据，Hive会将SQL翻译为Mapreduce来处理数据

#### 2. 关系

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/hbase_hive_relation.jpg)

在大数据架构中，通常HBase和Hive是协作关系：

1. 通过ETL（Extract-Transform-Load，提取、转换、加载）工具将数据源抽取到HDFS上存储
2. 通过Hive清洗、处理和计算源数据
3. 如果清洗过后的数据是用于海量数据的随机查询，则可将数据放入Hbase
4. 数据应用从Hbase中查询数据



*参考*

- [Hive和Hbase之间的差异？](https://www.zhihu.com/question/21677041)



