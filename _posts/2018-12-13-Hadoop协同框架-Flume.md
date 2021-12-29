---
layout: post
title:  'Hadoop协同框架-Flume'
date:   2018-12-13
tags: flume
categories: [大数据框架,flume]
---

### Flume结构

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2018-12-13/flume_structure.png)

- **Source** : 用户配置采集数据的方式（Http、LocalFileSystem、Tcp）
- **Channel** ——中间件
  - Memory Channel：临时存放到内存
  - FIle Channel ：临时存放到本地磁盘
- **Sink** ：将数据存放目的地（HDFS、本地文件系统、Logger、Http）



### 常用配置

```
# 每个组件的名称
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# netcat监控方式、监控的ip：localhost、端口：44444
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# sink 的方式 logger
a1.sinks.k1.type = logger

# 写入到内存、
a1.channels.c1.type = memory

# 绑定source和sink到channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

### Source

#### Exec source 

> 用于监控Linux命令

```
a1.sources = r1
a1.channels = c1
# 指定类型、命令
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /var/log/secure

a1.sources.r1.channels = c1
```

[Exec Source详细参数](https://flume.apache.org/FlumeUserGuide.html#exec-source)

#### Spooling Directory Source

> 用于监控文件，比Exec监控更加可靠

```
a1.channels = ch-1
a1.sources = src-1

fs.sources.r3.type=spooldir
fs.sources.r3.spoolDir=/opt/modules/apache-flume-1.6.0-bin/flume_template
fs.sources.r3.fileHeader=true
fs.sources.r3.ignorePattern=^(.)*\\.out$ # 过滤out结尾的文件
```

[Spooling Directory Source 详细参数](https://flume.apache.org/FlumeUserGuide.html#spooling-directory-source)

### Channel

#### Memory Channel

> 中间文件存放在内存中

```
a1.channels = c1
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 10000
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000
```

[Memory Channel 详细参数](https://flume.apache.org/FlumeUserGuide.html#memory-channel)

#### File Channel

> 中间文件存放在文件中

```
a1.channels = c1
a1.channels.c1.type = file
a1.channels.c1.checkpointDir = /mnt/flume/checkpoint
a1.channels.c1.dataDirs = /mnt/flume/data
```

[File Channel 详细参数](https://flume.apache.org/FlumeUserGuide.html#file-channel)

### Sink

#### Logger Sink

> 在INFO级别记录文件，通常用于调试

```
a1.channels = c1
a1.sinks = k1
a1.sinks.k1.type = logger
a1.sinks.k1.channel = c1
```

[Logger Sink 详细参数](https://flume.apache.org/FlumeUserGuide.html#logger-sink)

#### HDFS Sink

> 记录文件写入到HDFS中

```
a1.channels = c1
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1
a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H%M/%S
a1.sinks.k1.hdfs.filePrefix = events-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 10
a1.sinks.k1.hdfs.roundUnit = minute
```

[详细参数](https://flume.apache.org/FlumeUserGuide.html#hdfs-sink)