---
layout: post
title:  'spark源码分析 - shell'
date:   2018-12-27 21:58:14
tags: spark
categories: [大数据框架,spark]
---



![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2018-12-27/spark-shell-processes%20.png)

spark-shell

```bash
function main() {
# 对当前系统进行判断，通过spark-submits.sh 启动 org.apache.spark.repl.Main
  if $cygwin; then
    stty -icanon min 1 -echo > /dev/null 2>&1
    export SPARK_SUBMIT_OPTS="$SPARK_SUBMIT_OPTS -Djline.terminal=unix"
    "${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"
    stty icanon echo > /dev/null 2>&1
  else
    export SPARK_SUBMIT_OPTS
    "${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"
  fi
}
```

org.apache.spark.repl.Main

```scala
def main(args: Array[String]) {
    //初始化SparkILoop，调用process方法
    _interp = new SparkILoop
    _interp.process(args)
}
```

SparkILoop.process

```scala
private def process(settings: Settings): Boolean = savingContextLoader {
	......
	//前面内容很多，大致就是判断一些参数、初始化解释器之类的，例如运行的模式，
    //然后就运行主要的两个方法
    addThunk(printWelcome())
    addThunk(initializeSpark())
    ......
    //后面的也不是很重要
}
```

printWelcome

```scala
//打印Spark 中的版本等信息，也就是每次启动Spark-shell显示的欢迎界面
def printWelcome() {
    echo("""Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version %s
      /_/
""".format(SPARK_VERSION))
    import Properties._
    val welcomeMsg = "Using Scala %s (%s, Java %s)".format(
      versionString, javaVmName, javaVersion)
    echo(welcomeMsg)
    echo("Type in expressions to have them evaluated.")
    echo("Type :help for more information.")
 }
```

initializeSpark

```scala
def initializeSpark() {
    intp.beQuietDuring {
        //创建SparkContex，也就是创建spark运行环境，以及下面的SparkSql运行环境
      command("""
         @transient val sc = {
           val _sc = org.apache.spark.repl.Main.interp.createSparkContext()
           println("Spark context available as sc.")
           _sc
         }
        """)
      command("""
         @transient val sqlContext = {
           val _sqlContext = org.apache.spark.repl.Main.interp.createSQLContext()
           println("SQL context available as sqlContext.")
           _sqlContext
         }
        """)
      command("import org.apache.spark.SparkContext._")
      command("import sqlContext.implicits._")
      command("import sqlContext.sql")
      command("import org.apache.spark.sql.functions._")
    }
  }
```

createSparkContext

```scala
//初始化SparkContex，初始化createSQLContext就不贴了
def createSparkContext(): SparkContext = {
    val execUri = System.getenv("SPARK_EXECUTOR_URI")
    val jars = SparkILoop.getAddedJars
    val conf = new SparkConf()
      .setMaster(getMaster())
      .setJars(jars)
      .set("spark.repl.class.uri", intp.classServerUri)
      .setIfMissing("spark.app.name", "Spark shell")
    if (execUri != null) {
      conf.set("spark.executor.uri", execUri)
    }
    sparkContext = new SparkContext(conf)
    logInfo("Created spark context..")
    sparkContext
  }
```

