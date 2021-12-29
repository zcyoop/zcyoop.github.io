---
layout: post
title:  'Zookeeper 简述'
date:   2018-11-13
tags: [hadoop,zookeeper]
categories: [大数据框架,zookeeper]
---



### 1. Zookeeper 概述

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

ZooKeeper 一个最常用的使用场景就是用于担任服务生产者和服务消费者的注册中心。



### 2. Zookeeper  的一些重要概念

#### 2. 1概念总结

- **Zookeeper是一个分布式程序**   只要半数节点存活，Zookeeper就可以正常运行。每个Server上都保存着一份相同的副本
- **Zookeeper数据保存在内存中**  为了保证高吞吐和低延迟
- **Zookeeper是高性能的**   在“读”多于“写”的情况下，高性能尤为明显。因为写操作会涉及到服务器间同步状态
- **Zookeeper底层只提供了两个功能**  1. 管理（存储、读取）用户提交的数据     2. 为用户监听提交的数据



#### 2.2 会话（session）

Session 指的是 ZooKeeper  服务器与客户端会话。在 ZooKeeper 中，一个客户端连接是指客户端和服务器之间的一个 TCP 长连接。

客户端启动的时候，首先会与服务器建立一个 TCP 连接，从第一次连接建立开始，客户端会话的生命周期也开始了。

通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向 Zookeeper 服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的 Watch 事件通知。

Session 的 sessionTimeout 值用来设置一个客户端会话的超时时间。

当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，只要在 sessionTimeout 规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。

在为客户端创建会话之前，服务端首先会为每个客户端都分配一个 sessionID。

由于 sessionID 是 Zookeeper 会话的一个重要标识，许多与会话相关的运行机制都是基于这个 sessionID 的。

因此，无论是哪台服务器为客户端分配的 sessionID，都务必保证全局唯一。



#### 2.3  Znode

节点通常分为两类：

- 第一类同样是指构成集群的机器，**我们称之为机器节点。**
- 第二类则是指数据模型中的数据单元，**我们称之为数据节点一ZNode。**

ZooKeeper 将所有数据存储在内存中，数据模型是一棵树（Znode Tree)，由斜杠（/）的进行分割的路径，就是一个 Znode，例如/foo/path1。每个上都会保存自己的数据内容，同时还会保存一系列属性信息。

在 Zookeeper 中，Node 可以分为持久节点和临时节点两类。

- 持久节点是指一旦这个 ZNode 被创建了，除非主动进行 ZNode 的移除操作，否则这个 ZNode 将一直保存在 ZooKeeper 上。
- 临时节点就不一样了，它的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除。

>另外，ZooKeeper 还允许用户为每个节点添加一个特殊的属性：SEQUENTIAL。
>
>一旦节点被标记上这个属性，那么在这个节点被创建的时候，ZooKeeper 会自动在其节点名后面追加上一个整型数字，这个整型数字是一个由父节点维护的自增数字。



#### 2.4 版本

在前面我们已经提到，Zookeeper 的每个 ZNode 上都会存储数据，对应于每个 ZNode，Zookeeper 都会为其维护一个叫作 Stat 的数据结构。

- cZxid：这是导致创建znode更改的事务ID。
- mZxid：这是最后修改znode更改的事务ID。
- pZxid：这是用于添加或删除子节点的znode更改的事务ID。
- ctime：表示从1970-01-01T00:00:00Z开始以毫秒为单位的znode创建时间。
- mtime：表示从1970-01-01T00:00:00Z开始以毫秒为单位的znode最近修改时间。
- dataVersion：表示对该znode的数据所做的更改次数。
- cversion：这表示对此znode的子节点进行的更改次数。
- aclVersion：表示对此znode的ACL进行更改的次数。
- ephemeralOwner：如果znode是ephemeral类型节点，则这是znode所有者的 session ID。 如果znode不是ephemeral节点，则该字段设置为零。
- dataLength：这是znode数据字段的长度。
- numChildren：这表示znode的子节点的数量。

#### 2.5  Watcher

Watcher（事件监听器），是 ZooKeeper 中的一个很重要的特性。

ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将

事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。



### 3. Zookeeper设计目标

#### 3.1 数据模型

![mark](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/img/7EdLJGGJlh.gif)

ZooKeeper 允许分布式进程通过共享的层次结构命名空间进行相互协调，这与标准文件系统类似。

名称空间由 ZooKeeper 中的数据寄存器组成，称为 Znode，这些类似于文件和目录。

与为存储设计的典型文件系统不同，ZooKeeper 数据保存在内存中，这意味着 ZooKeeper 可以实现高吞吐量和低延迟。

#### 3.2 可构建集群

为了保证高可用，最好是以集群形态来部署 ZooKeeper，这样只要集群中大部分机器是可用的（能够容忍一定的机器故障），那么 ZooKeeper 本身仍然是可用的。 

客户端在使用 ZooKeeper 时，需要知道集群机器列表，通过与集群中的某一台机器建立 TCP 连接来使用服务。

客户端使用这个 TCP 链接来发送请求、获取结果、获取监听事件以及发送心跳包。如果这个连接异常断开了，客户端可以连接到另外的机器上。

ZooKeeper 官方提供的架构图：

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/img/gE41kLbKBg.webp)

### 4. ZooKeeper 集群角色介绍

![mark](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/img/27jdbi7K9m.png?imageslim)

- ZooKeeper 集群中的所有机器通过一个 Leader 选举过程来选定一台称为 “Leader” 的机器。
- Leader 既可以为客户端提供写服务又能提供读服务。除了 Leader 外，Follower 和  Observer 都只能提供读服务。
- 当Client向Follow、Observer写入数据时，写入操作会同步到Leader，Leader会将同步的信息分发到其他节点上，当大多数节点写入完成后，会向Client返回响应
- Follower 和 Observer 唯一的区别在于 Observer 机器不参与 Leader 的选举过程，也不参与写操作的“过半写成功”策略，因此 Observer 机器可以在不影响写性能的情况下提升集群的读性能。

### 5.Shell命令

- zkCli.sh / zookeeper-client 进入

- help 查看帮助信息

- ls /  查看 “/” 节点下的信息

- ls2 / 查看 “/” 节点下的详细信息

- create /hive  "[数据信息]"     创建节点

- get  /bigdata  获取节点信息

- create -e /bigdata "aaa" 创建临时节点（退出之后删除

- 创建序号节点

  - create  -s  /bigdata/Impala  "node1"
  - create  -s  /bigdata/Impala  "node2"

- set  /bigdata  "Flink" 修改节点信息

- 监听

  - get /bigdata  [watch] 当该节点值改变时，会得到响应
  - ls  /bigdata  [watch]  当节信息改变时，会得到响应
  - 注册一次，监听一次

- delete  /bigdata/Hive    删除节点

- rmr  /bigdata   递归删除

- stat   /    查看状态


###  6.  监听器原理

![mark](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/img/aK1FCI8jeH.png?imageslim)

1. main()线程启动
2. 创建zookeeper-client，创建两个线程connect和listener
3. 通过connect线程将注册信息发送给zookeeper
4.  zookeeper监听器将注册监听的事件添加到注册的监听事件列表     例如：get  /bigdata  watch  client
5. zookeeper监听到有数据或路径的变化，就会发送消息到listener线程
6. listener线程内部调用process()方法，即自己编写的对应措施

*参看文章*

- [51CTO](https://mp.weixin.qq.com/s?src=11&timestamp=1542088492&ver=1241&signature=dpB23OYaRnNNdROWjRmTEQtCCLKz9L1lIfzvP6DaWV--hqr81oPdOTvl0rrHo4r5rtrroBcZF5UE9jSKQPWe7PG3hhmqqOjPKJZNN-rUPaDEVbraoKL08Jh4cS5YBmWK&new=1)
- [官方文档](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index*)

