---
title:  'Flink - Flink编程结构'
date:  2020-1-15
tags: flink
categories: [大数据框架,flink]
---

## Flink程序结构

### 概述

> 任何程序都是需要有**输入、处理、输出**。
> 那么Flink同样也是，Flink专业术语对应**Source，map，Sink**。而在进行这些操作前，需要根据需求初始化运行环境

### 执行环境

 Flink 执行模式分为两种，一个是流处理、另一个是批处理。再选择好执行模式后，为了开始编写Flink程序，需要根据需求创建一个执行环境。Flink目前支持三种环境的创建方式：

- 获取一个已经有的environment

- 创建一个本地environment

- 创建一个远程environment

   通常，只需要使用`getExecutionEnvironment()`。 它会根据你的环境来选择。 如果你在IDE中的本地环境中执行，那么它将启动本地执行环境。 否则，如果正在执行JAR，则Flink集群管理器将以分布式方式执行该程序。

流处理程序部分代码：

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
val text = env.socketTextStream("localhost", 9999)
```

批处理程序部分代码：

```scala
val env = ExecutionEnvironment.getExecutionEnvironment
val text = env.fromElements(
      "Who's she?","Alice")
```

### 数据源（Source）

Flink的source到底是什么？为了更好地理解，我们这里给出下面一个简单典型的wordcount程序。

```scala
//初始化流处理环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
//指定数据源
val text = env.socketTextStream(host, port, '\n')
//对数据源传入的数据进行处理
val windowCounts = text.flatMap { w => w.split("\\s") }
  .map { w => WordWithCount(w, 1) }
  .keyBy("word")
  .timeWindow(Time.seconds(5))
  .sum("count")
//输出结果
windowCounts.print()
```

在上述代码中`val text = env.socketTextStream(host, port, '\n')`就是指定数据源。Flink的source多种多样，例如我们可以根据不同的需求来自定义source。

#### DataStream API

**基于Socket**

- socketTextStream(host,port)：从套接字读取数据,只需指定要从中读取数据的主机和端口
- socketTextStream(hostName,port,delimiter)：指定分隔符
- socketTextStream(hostName,port,delimiter,maxRetry)：API尝试最大次数

**基于文件**

- readTextFile(path) ： 读取文本类型文件
- readFile(fileInputFormat, path) ：读取非文本文件，需要指定输入格式
- readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo) ：该方法为前两个方法的内部调用方法，可以根据给定的输入格式读取指定路径的文件。watchType、interval分别指定监控类型和监控间隔，监控类型包括三种：
  1. 当系统应仅处理新文件时使用FileMonitoringFunction.WatchType.ONLY_NEW_FILES 
  2. 当系统仅追加文件内容时使用FileMonitoringFunction.WatchType.PROCESS_ONLY_APPENDED
  3. 当系统不仅要重新处理文件的追加内容而且还要重新处理文件中的先前内容时，将使用FileMonitoringFunction.WatchType.REPROCESS_WITH_APPENDED

**自定义**

- addSource：附加新的source函数。 例如，要从Apache Kafka读取，可以使用addSource（new FlinkKafkaConsumer08 <>（...））。 请参阅[connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/connectors/)以获取更多内容。

#### DataSet API

**基于文件**

- readTextFile（path）/ TextInputFormat  ： 按行读取文件并将它们作为字符串返回。

- readCsvFile（path）/ CsvInputFormat  ： 解析逗号（或其他字符）分隔字段的文件。 返回元组，案例类对象或POJO的DataSet。 支持基本java类型及其Value对应作为字段类型。
- readFileOfPrimitives（path，delimiter）/ PrimitiveInputFormat  ： 使用给定的分隔符解析新行（或其他char序列）分隔的原始数据类型（如String或Integer）的文件。
- readSequenceFile（Key，Value，path）/ SequenceFileInputFormat  ； 创建JobConf并从指定路径读取文件，类型为SequenceFileInputFormat，Key class和Value类，并将它们返回为Tuple2 <Key，Value>。  

**基于Collection**

- fromCollection（Seq） ：从Seq创建数据集。 集合中的所有元素必须属于同一类型。
- fromCollection（Iterator） ：从迭代器创建数据集。 该类指定迭代器返回的元素的数据类型。
- fromElements（elements：_ *） ： 根据给定的对象序列创建数据集。 所有对象必须属于同一类型。
- fromParallelCollection（SplittableIterator） ： 并行地从迭代器创建数据集。 该类指定迭代器返回的元素的数据类型。
- generateSequence（from，to） ： 并行生成给定间隔中的数字序列。  

**通用**

- readFile（inputFormat，path）/ FileInputFormat  ： 接受文件输入格式。

- createInput（inputFormat）/ InputFormat  ： 接受通用输入格式。  

### 处理

在读取数据源以后就开始了数据的处理

```scala
val windowCounts = text.flatMap { w => w.split("\\s") }
  .map { w => WordWithCount(w, 1) }
  .keyBy("word")
  .timeWindow(Time.seconds(5))
  .sum("count")
```

flatMap 、map 、keyBy、timeWindow、sum那么这些就是对数据的处理。更多算子信息：

- [**DataStream Transformations（流处理）**](https://ci.apache.org/projects/flink/flink-docs-release-1.9/zh/dev/stream/operat- ors/#datastream-transformations)
- [**DataSet Transformations（批处理）**](https://ci.apache.org/projects/flink/flink-docs-release-1.9/zh/dev/batch/dataset_transformations.html)

### 保存数据（Sink）

在上述代码中`windowCounts.print()`也就是改程序的保存数据

 这里输出可以说是非常简单的。而sink当然跟source一样也是可以自定义的。
因为Flink数据要保存到myslq，是不能直接保存的，所以需要自定义一个sink。不定义sink可以吗？可以的，那就是自己在写一遍，每次调用都复制一遍，这样造成大量的重复，所以我们需要自定义sink。
那么常见的sink有哪些?如下：
**flink在批处理中常见的sink**
1.基于本地集合的sink（Collection-based-sink）
2.基于文件的sink（File-based-sink）

[DataStream Data Sink](https://ci.apache.org/projects/flink/flink-docs-release-1.9/zh/dev/datastream_api.html#data-sinks)

[DataSet Data Sink](https://ci.apache.org/projects/flink/flink-docs-release-1.9/zh/dev/batch/#data-sinks)

*参考*

[Flink程序结构](https://www.aboutyun.com/forum.php?mod=viewthread&tid=26371&extra=page%3D2%26filter%3Dauthor%26orderby%3Ddateline%26typeid%3D1393)