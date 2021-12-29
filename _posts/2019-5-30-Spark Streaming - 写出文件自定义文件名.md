---
layout: post
title:  'Spark Streaming - 写出文件自定义文件名'
date:   2019-5-30
tags: spark
categories: [大数据框架,spark]
---

### 1.背景

​       在工作中碰到了个需求，需要将`Spark Streaming`中的文件写入到`Hive`表中，但是`Spark Streaming`中的`saveAsTextFiles`会自己定义很多文件夹，不符合`Hive`读取文件的规范且`saveAsTextFiles`中的参数只能定义文件夹的名字，第二个是采用`Spark Streaming`中的`foreachRDD`，这个方法会将`DStream`转成再进行操作，但是`Spark Streaming`中的是多批次处理的结构，也就是很多RDD，每个RDD的`saveAsTextFile`都会将前面的数据覆盖，所以最终采用的方法是重写`saveAsTextFile`输出时的文件名



### 2.分析

#### 2.1 分析代码

既然是重写`saveAsTextFile`输出逻辑，那先看看他是如何实现输出的

```scala
def saveAsTextFile(path: String): Unit = withScope {
    // https://issues.apache.org/jira/browse/SPARK-2075
    //
    // NullWritable is a `Comparable` in Hadoop 1.+, so the compiler cannot find an implicit
    // Ordering for it and will use the default `null`. However, it's a `Comparable[NullWritable]`
    // in Hadoop 2.+, so the compiler will call the implicit `Ordering.ordered` method to create an
    // Ordering for `NullWritable`. That's why the compiler will generate different anonymous
    // classes for `saveAsTextFile` in Hadoop 1.+ and Hadoop 2.+.
    //
    // Therefore, here we provide an explicit Ordering `null` to make sure the compiler generate
    // same bytecodes for `saveAsTextFile`.
    val nullWritableClassTag = implicitly[ClassTag[NullWritable]]
    val textClassTag = implicitly[ClassTag[Text]]
    val r = this.mapPartitions { iter =>
      val text = new Text()
      iter.map { x =>
        text.set(x.toString)
        (NullWritable.get(), text)
      }
    }
    RDD.rddToPairRDDFunctions(r)(nullWritableClassTag, textClassTag, null)
      .saveAsHadoopFile[TextOutputFormat[NullWritable, Text]](path)
  }
```

可以看出`saveAsTextFile`是依赖`saveAsHadoopFile`进行输出，因为`saveAsHadoopFile`接受`PairRDD`，所以在`saveAsTextFile`中通过`rddToPairRDDFunctions`转成(NullWritable,Text)类型的RDD，再通过`saveAsHadoopFile`进行输出

可以看出输出的逻辑还是Hadoop的那一套，所以我们可以通过重写`TextOutputFormat`来解决输出文件名的相同的问题

#### 2.2 代码编写

##### 2.2.1 saveAsHadoopFile算子

首先先看下官方提供的`saveAsHadoopFile`算子说明

```scala
/**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class
   * supporting the key and value types K and V in this RDD.
   */
  def saveAsHadoopFile[F <: OutputFormat[K, V]](
      path: String)(implicit fm: ClassTag[F]): Unit = self.withScope {
    saveAsHadoopFile(path, keyClass, valueClass, fm.runtimeClass.asInstanceOf[Class[F]])
  }
/**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class
   * supporting the key and value types K and V in this RDD. Compress the result with the
   * supplied codec.
   */
  def saveAsHadoopFile[F <: OutputFormat[K, V]](
      path: String,
      codec: Class[_ <: CompressionCodec])(implicit fm: ClassTag[F]): Unit = self.withScope {
    val runtimeClass = fm.runtimeClass
    saveAsHadoopFile(path, keyClass, valueClass, runtimeClass.asInstanceOf[Class[F]], codec)
  }
 /**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class
   * supporting the key and value types K and V in this RDD. Compress with the supplied codec.
   */
def saveAsHadoopFile(
      path: String,
      keyClass: Class[_],
      valueClass: Class[_],
      outputFormatClass: Class[_ <: OutputFormat[_, _]],
      codec: Class[_ <: CompressionCodec]): Unit = self.withScope {
    saveAsHadoopFile(path, keyClass, valueClass, outputFormatClass,
      new JobConf(self.context.hadoopConfiguration), Some(codec))
  }
/**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class
   * supporting the key and value types K and V in this RDD.
   *
   * Note that, we should make sure our tasks are idempotent when speculation is enabled, i.e. do
   * not use output committer that writes data directly.
   * There is an example in https://issues.apache.org/jira/browse/SPARK-10063 to show the bad
   * result of using direct output committer with speculation enabled.
   */
  def saveAsHadoopFile(
      path: String,
      keyClass: Class[_],
      valueClass: Class[_],
      outputFormatClass: Class[_ <: OutputFormat[_, _]],
      conf: JobConf = new JobConf(self.context.hadoopConfiguration),
      codec: Option[Class[_ <: CompressionCodec]] = None): Unit = self.withScope {...}
```

这里我们使用的是`def saveAsHadoopFile(path: String, keyClass: Class[_],valueClass: Class[_], outputFormatClass: Class[_ <: OutputFormat[_, _]], codec: Class[_ <: CompressionCodec]): Unit = self.withScope { }`

依次传入 path：路径、keyClass：key类型、valueClass：value类型、outputFormatClass：outformat方式，剩下两个参数为默认值

##### 2.2.2 MultipleTextOutputFormat分析

```scala
/**
 * This abstract class extends the FileOutputFormat, allowing to write the
 * output data to different output files. There are three basic use cases for
 * this class.
 * 
 * Case one: This class is used for a map reduce job with at least one reducer.
 * The reducer wants to write data to different files depending on the actual
 * keys. It is assumed that a key (or value) encodes the actual key (value)
 * and the desired location for the actual key (value).
 * 
 * Case two: This class is used for a map only job. The job wants to use an
 * output file name that is either a part of the input file name of the input
 * data, or some derivation of it.
 * 
 * Case three: This class is used for a map only job. The job wants to use an
 * output file name that depends on both the keys and the input file name,
 */
@InterfaceAudience.Public
@InterfaceStability.Stable
public abstract class MultipleOutputFormat<K, V>
extends FileOutputFormat<K, V> {

  /**
   * Create a composite record writer that can write key/value data to different
   * output files
   * 
   * @param fs
   *          the file system to use
   * @param job
   *          the job conf for the job
   * @param name
   *          the leaf file name for the output file (such as part-00000")
   * @param arg3
   *          a progressable for reporting progress.
   * @return a composite record writer
   * @throws IOException
   */
  public RecordWriter<K, V> getRecordWriter(FileSystem fs, JobConf job,
      String name, Progressable arg3) throws IOException {

    final FileSystem myFS = fs;
    final String myName = generateLeafFileName(name);
    final JobConf myJob = job;
    final Progressable myProgressable = arg3;

    return new RecordWriter<K, V>() {

      // a cache storing the record writers for different output files.
      TreeMap<String, RecordWriter<K, V>> recordWriters = new TreeMap<String, RecordWriter<K, V>>();

      public void write(K key, V value) throws IOException {

        // get the file name based on the key
        String keyBasedPath = generateFileNameForKeyValue(key, value, myName);

        // get the file name based on the input file name
        String finalPath = getInputFileBasedOutputFileName(myJob, keyBasedPath);

        // get the actual key
        K actualKey = generateActualKey(key, value);
        V actualValue = generateActualValue(key, value);

        RecordWriter<K, V> rw = this.recordWriters.get(finalPath);
        if (rw == null) {
          // if we don't have the record writer yet for the final path, create
          // one
          // and add it to the cache
          rw = getBaseRecordWriter(myFS, myJob, finalPath, myProgressable);
          this.recordWriters.put(finalPath, rw);
        }
        rw.write(actualKey, actualValue);
      };

      public void close(Reporter reporter) throws IOException {
        Iterator<String> keys = this.recordWriters.keySet().iterator();
        while (keys.hasNext()) {
          RecordWriter<K, V> rw = this.recordWriters.get(keys.next());
          rw.close(reporter);
        }
        this.recordWriters.clear();
      };
    };
  }

  /**
   * Generate the leaf name for the output file name. The default behavior does
   * not change the leaf file name (such as part-00000)
   * 
   * @param name
   *          the leaf file name for the output file
   * @return the given leaf file name
   */
  protected String generateLeafFileName(String name) {
    return name;
  }

  /**
   * Generate the file output file name based on the given key and the leaf file
   * name. The default behavior is that the file name does not depend on the
   * key.
   * 
   * @param key
   *          the key of the output data
   * @param name
   *          the leaf file name
   * @return generated file name
   */
  protected String generateFileNameForKeyValue(K key, V value, String name) {
    return name;
  }

  /**
   * Generate the actual key from the given key/value. The default behavior is that
   * the actual key is equal to the given key
   * 
   * @param key
   *          the key of the output data
   * @param value
   *          the value of the output data
   * @return the actual key derived from the given key/value
   */
  protected K generateActualKey(K key, V value) {
    return key;
  }
  
  /**
   * Generate the actual value from the given key and value. The default behavior is that
   * the actual value is equal to the given value
   * 
   * @param key
   *          the key of the output data
   * @param value
   *          the value of the output data
   * @return the actual value derived from the given key/value
   */
  protected V generateActualValue(K key, V value) {
    return value;
  }
  

  /**
   * Generate the outfile name based on a given anme and the input file name. If
   * the {@link JobContext#MAP_INPUT_FILE} does not exists (i.e. this is not for a map only job),
   * the given name is returned unchanged. If the config value for
   * "num.of.trailing.legs.to.use" is not set, or set 0 or negative, the given
   * name is returned unchanged. Otherwise, return a file name consisting of the
   * N trailing legs of the input file name where N is the config value for
   * "num.of.trailing.legs.to.use".
   * 
   * @param job
   *          the job config
   * @param name
   *          the output file name
   * @return the outfile name based on a given anme and the input file name.
   */
  protected String getInputFileBasedOutputFileName(JobConf job, String name) {
    String infilepath = job.get(MRJobConfig.MAP_INPUT_FILE);
    if (infilepath == null) {
      // if the {@link JobContext#MAP_INPUT_FILE} does not exists,
      // then return the given name
      return name;
    }
    int numOfTrailingLegsToUse = job.getInt("mapred.outputformat.numOfTrailingLegs", 0);
    if (numOfTrailingLegsToUse <= 0) {
      return name;
    }
    Path infile = new Path(infilepath);
    Path parent = infile.getParent();
    String midName = infile.getName();
    Path outPath = new Path(midName);
    for (int i = 1; i < numOfTrailingLegsToUse; i++) {
      if (parent == null) break;
      midName = parent.getName();
      if (midName.length() == 0) break;
      parent = parent.getParent();
      outPath = new Path(midName, outPath);
    }
    return outPath.toString();
  }

  /**
   * 
   * @param fs
   *          the file system to use
   * @param job
   *          a job conf object
   * @param name
   *          the name of the file over which a record writer object will be
   *          constructed
   * @param arg3
   *          a progressable object
   * @return A RecordWriter object over the given file
   * @throws IOException
   */
  abstract protected RecordWriter<K, V> getBaseRecordWriter(FileSystem fs,
      JobConf job, String name, Progressable arg3) throws IOException;
}

```

可以看出，在写每条记录之前，MultipleOutputFormat将调用generateFileNameForKeyValue方法来确定文件名，所以在只需要重写generateFileNameForKeyValue方法即可

##### 2.2.3 MultipleOutFormat重写

```scala
class RDDMultipleTextOutputFormat extends MultipleTextOutputFormat[Any, Any] {

  private val start_time = System.currentTimeMillis()

  override def generateFileNameForKeyValue(key: Any, value: Any, name: String): String = {
    
    val service_date =  start_time + "-" + name //.split("-")(0)
    service_date
  }
}
```

Spark Streaming 代码修改

```scala
...//业务代码
.map(x=>(x,""))//由于saveAsHadoopFile接受PariRDD，所以需要转成这样 
.saveAsHadoopFile(finallpath,classOf[String],classOf[String],classOf[RDDMultipleTextOutputFormat])
```

到此，已经可以解决覆盖问题

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-5-30/TIM%E6%88%AA%E5%9B%BE20190530222709.jpg)

#### 参考

[Spark(Streaming)写入数据到文件](http://ileaf.tech/bigdata/2018/07/02/Spark-Streaming-%E5%86%99%E5%85%A5%E6%95%B0%E6%8D%AE%E5%88%B0%E6%96%87%E4%BB%B6-%E5%85%B3%E9%94%AE%E4%B8%BASpark%E6%A0%B9%E6%8D%AE%E6%95%B0%E6%8D%AE%E5%86%85%E5%AE%B9%E8%BE%93%E5%87%BA%E5%88%B0%E4%B8%8D%E5%90%8C%E8%87%AA%E5%AE%9A%E4%B9%89%E5%90%8D%E7%A7%B0%E6%96%87%E4%BB%B6-saveAsHadoopFile%E4%BB%A5%E5%8F%8A%E8%87%AA%E5%AE%9A%E4%B9%89MultipleOutputFormat/)

