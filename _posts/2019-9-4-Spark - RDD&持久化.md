---
layout: post
title:  'Spark - RDD&持久化'
date:  2019-9-4
tags: spark
categories: [大数据框架,spark]
---

### 1.RDD持久化简介

Spark 中一个很重要的能力是将数据持久化（或称为缓存），在多个操作间都可以访问这些持久化的数据。当持久化一个 RDD 时，每个节点的其它分区都可以使用 RDD 在内存中进行计算，在该数据上的其他 action 操作将直接使用内存中的数据。这样会让以后的 action 操作计算速度加快（通常运行速度会加速 10 倍）。缓存是迭代算法和快速的交互式使用的重要工具。

RDD 可以使用 persist() 方法或 cache() 方法进行持久化。数据将会在第一次 action 操作时进行计算，并缓存在节点的内存中。Spark 的缓存具有容错机制，如果一个缓存的 RDD 的某个分区丢失了，Spark 将按照原来的计算过程，自动重新计算并进行缓存。

**在 shuffle 操作中（例如 reduceByKey），即便是用户没有调用 persist 方法，Spark 也会自动缓存部分中间数据。这么做的目的是，在 shuffle 的过程中某个节点运行失败时，不需要重新计算所有的输入数据**。如果用户想多次使用某个 RDD，强烈推荐在该 RDD 上调用 persist 方法。



### 2. RDD的cache和persist的区别

先看看cache()源码

```java
  /**
   * Persist this RDD with the default storage level (`MEMORY_ONLY`).
   */
  def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)

  /**
   * Persist this RDD with the default storage level (`MEMORY_ONLY`).
   */
  def cache(): this.type = persist()
```

可以看出cache调用的任然是persist()方法，默认级别是`StorageLevel.MEMORY_ONLY`，也就是persis可以自定义缓存级别，而cache为默认级别，当然直接使用persist()也是默认级别



### 3. Persist的各个缓存级别

persist的各个级别都存在`StorageLevel`对象里面，直接看看`StorageLevel`里的方法

```java
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)
```

可以看出，一共有12个级别，在看看`StorageLevel`的构造方法

```java
class StorageLevel private(
    private var _useDisk: Boolean,
    private var _useMemory: Boolean,
    private var _useOffHeap: Boolean,
    private var _deserialized: Boolean,
    private var _replication: Int = 1)
  extends Externalizable {
	...
}
```

可以看到`StorageLevel`类的主构造器包含了5个参数：

- useDisk：使用硬盘（外存）
- useMemory：使用内存
- useOffHeap：使用堆外内存，这是Java虚拟机里面的概念，堆外内存意味着把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机）。这样做的结果就是能保持一个较小的堆，以减少垃圾收集对应用的影响。
- deserialized：反序列化，其逆过程序列化（Serialization）是java提供的一种机制，将对象表示成一连串的字节；而反序列化就表示将字节恢复为对象的过程。序列化是对象永久化的一种机制，可以将对象及其属性保存起来，并能在反序列化后直接恢复这个对象
- replication：备份数（在多个节点上备份）

理解了这5个参数，StorageLevel 的12种缓存级别就不难理解了。

- MEMORY_ONLY : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算。这是默认的级别。
- MEMORY_AND_DISK : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取。
- MEMORY_ONLY_SER : 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 fast serializer时会节省更多的空间，但是在读取时会增加 CPU 的计算负担。
- MEMORY_AND_DISK_SER : 类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算。
- DISK_ONLY : 只在磁盘上缓存 RDD。
- MEMORY_ONLY_2，MEMORY_AND_DISK_2，等等 : 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本。
- OFF_HEAP（实验中）: 类似于 MEMORY_ONLY_SER ，但是将数据存储在 off-heap memory，这需要启动 off-heap 内存。

### 4.如何选择存储级别

Spark 的存储级别的选择，核心问题是在内存使用率和 CPU 效率之间进行权衡。建议按下面的过程进行存储级别的选择 :

- 如果使用默认的存储级别（MEMORY_ONLY），存储在内存中的 RDD 没有发生溢出，那么就选择默认的存储级别。默认存储级别可以最大程度的提高 CPU 的效率,可以使在 RDD 上的操作以最快的速度运行。
- 如果内存不能全部存储 RDD，那么使用 MEMORY_ONLY_SER，并挑选一个快速序列化库将对象序列化，以节省内存空间。使用这种存储级别，计算速度仍然很快。
- 除了在计算该数据集的代价特别高，或者在需要过滤大量数据的情况下，尽量不要将溢出的数据存储到磁盘。因为，重新计算这个数据分区的耗时与从磁盘读取这些数据的耗时差不多。
- 如果想快速还原故障，建议使用多副本存储级别（例如，使用 Spark 作为 web 应用的后台服务，在服务出故障时需要快速恢复的场景下）。所有的存储级别都通过重新计算丢失的数据的方式，提供了完全容错机制。但是多副本级别在发生数据丢失时，不需要重新计算对应的数据库，可以让任务继续运行。



### 5.删除数据

Spark 自动监控各个节点上的缓存使用率，并以最近最少使用的方式（LRU）将旧数据块移除内存。如果想手动移除一个 RDD，而不是等待该 RDD 被 Spark 自动移除，可以使用 RDD.unpersist() 方法



*参考*

- [Spark持久化](https://dongkelun.com/2018/06/03/sparkCacheAndPersist/)
- [spark中cache和persist的区别](https://blog.csdn.net/houmou/article/details/52491419)