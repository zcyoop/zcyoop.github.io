---
layout: post
title:  'Flink - Flink DataStream端到端的Exactly-Once保障'
date:  2021-5-8
tags: [flink]
categories: [大数据框架]
---

## 1. Exactly-Once概述

​		一个一直运行的Flink Stream程序不出错那肯定时很好的，但是在现实世界中，系统难免会出现各种意外，一旦故障发生，Flink作业就会重启，读取最近Checkpoint的数据，恢复状态，并继续接着执行任务。

​		Checkpoint时可以保证程序内部状态的一致性的，但是任会有数据重新消费的问题，举个简单的例子：

​		一个简单的计算总和的程序，从Kafka获取数字，并相加到一起，如图所示

 ![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picFlink Exactly Once1.svg)

1. 程序正常Checkpoint，输出`1，4，9`
2. 往后消费`7，9`后程序异常退出，此时程序输出`1，4，9，16，25`
3. 程序从上次`5`的位置进行恢复往后消费
4. 一直消费到`11`，此时程序由于是从`5`往后消费，所以会重新输出`16,25`

在上述情况中，程序重启后部分数据被重新发送了一次，也就是一些数据在某些情况下不止被处理了一次，而是多次，即`At-Least-Once`。有时候我们期望一条数据只影响一次最终结果，也就是`Exactly-Once`

## 2. 实现方式

### 2.1 幂等写

​		幂等写（Idempotent Write）是指，任意多次笑一个系统写入相同数据，只对目标系统产生一次结果影响，例如重复向一个HashMap里面插入三组相同的二元组，只有第一次插入时，数组结果会发送变化，后面两次插入不会影响HashMap结果

### 2.2 事务写

​		事务（Transaction）时数据库系统所要解决的核心问题。Flink借鉴了数据库中的事务技术，同时结合自身的Checkpoint机制来保证Sink只对外部输出产生一次影响。

​		简单来说，Flink事务写是指，Flink先**将待输出的数据保存下来，暂时不提交到外部系统，等到CheckPoint结束，Flink上下游所有算子的数据一致时，再将之前保存的数据全部提交到外部系统**，如图所示。

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picFlink Exactly Once2.svg)

在事务写的具体实现上，Flink目前提供了两种方式：预写日志（Write-Ahead-Log，WAL）和两段式提交（Two-Phase-Commit，2PC）。这两种方式也是很多数据库和分布式系统实现事务经常采用的方式，Flink根据自身的条件对两种方式做了适应性调整。

### 2.2.1 Write-Ahead-Log协议原理

​		Write-Ahead-Log核心思想是，再写入下游系统之前，先把数据以日志的形式缓存下来，等收到明确的确认提交信息后，再将Log中的数据提交到下游系统。由于数据都写到了Log里，即使出现故障恢复，也可以根据Log中的数据决定是否需要恢复、如何恢复。而在Fliink中，数据会被保存在State中。

​		但是Write-Ahead-Log仍然无法提供百分之百的Exactly-Once，例如，写入下游系统时可能中途崩溃，导致部分数据提交，部分数据未提交。

​		Write-Ahead-Log的方式相对比较通用，目前Flink的Cassandra Sink使用这种方式提供Exactly-Once保障

### 2.2.2 Two-Phase-Commit 协议的原理和实现

​		Two-Phase-Commit 与Write-Ahead-Log相比，Flink中的Two-Phase-Commit协议不再将数据缓存在State中，而是直接将数据写入到外部系统，比如支持事务的Kafka。

​		在Flink写出数据到Kafka中时，Flink会先beginTransaction()开启事务，事务开启后再preCommit()预提交数据，待Flink Checkpoint完成后，Flink会commit()提交这些数据，此时一组数据就被写入到了Kafka。

​		值得注意的是，Kafka Consumer在默认情况下，是可以读取到preCommit()的数据，只有当`isolation.level`被设置为`read_committed`时，Kafka Consumer才会只读取commit()后的数据（ `read_uncommitted` - 是默认值）



*参考*

- Flink原理与实践

- [Apache Kafka 连接器](https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/kafka/#apache-kafka-连接器)