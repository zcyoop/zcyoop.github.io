---
layout: post
title:  'Spark源码分析 - start-all'
date:   2018-12-31 16:54:31
tags: spark
categories: [大数据框架,spark]
---

#### 版本

spark-1.6

#### 过程分析

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2018-12-27/spark-start-all.png)

start-all.sh

```bash
# 加载环境
. "${SPARK_HOME}/sbin/spark-config.sh"
# 启动Master
"${SPARK_HOME}/sbin"/start-master.sh $TACHYON_STR

# 启动Worker
"${SPARK_HOME}/sbin"/start-slaves.sh $TACHYON_STR
```

start-master.sh

```bash
......
# 类名
CLASS="org.apache.spark.deploy.master.Master"

# 加载该类
if [[ "$@" = *--help ]] || [[ "$@" = *-h ]]; then
  echo "Usage: ./sbin/start-master.sh [options]"
  pattern="Usage:"
  pattern+="\|Using Spark's default log4j profile:"
  pattern+="\|Registered signal handlers for"

  "${SPARK_HOME}"/bin/spark-class $CLASS --help 2>&1 | grep -v "$pattern" 1>&2
  exit 1
fi
......
```

`org.apache.spark.deploy.master.Master` 让我们先来看看`main()`方法

```scala
  def main(argStrings: Array[String]) {
    // 日志注册
    SignalLogger.register(log)
    val conf = new SparkConf
    val args = new MasterArguments(argStrings, conf)
    // 注册消息处理的环境以及Master的通讯工具，具体通讯方式会另外补充一篇博文
    val (rpcEnv, _, _) = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, conf)
    // 注册完成后等待接受消息
    rpcEnv.awaitTermination()
  }
```

> PS : Spark-1.6以后RPC默认使用Netty替代Akka，在Netty上加了一层封装，为实现对Spark的定制开发。
>
> 具体是为什么，见[Enable Spark user applications to use different versions of Akka](https://issues.apache.org/jira/browse/SPARK-5293)

`startRpcEnvAndEndpoint()`

```scala
  def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
    val securityMgr = new SecurityManager(conf)
    //注册通讯环境
    val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
   // 注册Master通讯节点
    val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
      new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
    val portsResponse = masterEndpoint.askWithRetry[BoundPortsResponse](BoundPortsRequest)
    (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
  }
```

然后到Worker

`start-slave.sh` 

```bash
# 流程与Master一样
CLASS="org.apache.spark.deploy.worker.Worker"

if [[ $# -lt 1 ]] || [[ "$@" = *--help ]] || [[ "$@" = *-h ]]; then
  echo "Usage: ./sbin/start-slave.sh [options] <master>"
  pattern="Usage:"
  pattern+="\|Using Spark's default log4j profile:"
  pattern+="\|Registered signal handlers for"

  "${SPARK_HOME}"/bin/spark-class $CLASS --help 2>&1 | grep -v "$pattern" 1>&2
  exit 1
fi

```

依然是先来看看main()方法

```scala
  def main(argStrings: Array[String]) {
 	//与Master里面如出一辙
    SignalLogger.register(log)
    val conf = new SparkConf
    val args = new WorkerArguments(argStrings, conf)
    val rpcEnv = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, args.cores,
      args.memory, args.masters, args.workDir, conf = conf)
    rpcEnv.awaitTermination()
  }
```

但是如果大家都是在等待消息的话可定是无法完成交互的，也就是说在启动这些的时候肯定还有其他东西也有运行，于是查看了下Worker继承的类`ThreadSafeRpcEndpoint` 里面包含一个`onStart()`方法，

```scala
  /**
   * Invoked before [[RpcEndpoint]] starts to handle any message.
   * 会在RpcEndingpoint创建前被调用，用于处理信息
   */
  def onStart(): Unit = {
    // By default, do nothing.
  }
```

于是找到Worker.onStart()

```scala
  override def onStart() {
    assert(!registered)
    //日志记录
    logInfo("Starting Spark worker %s:%d with %d cores, %s RAM".format(
      host, port, cores, Utils.megabytesToString(memory)))
    logInfo(s"Running Spark version ${org.apache.spark.SPARK_VERSION}")
    logInfo("Spark home: " + sparkHome)
    //创建Worker目录
    createWorkDir()
    shuffleService.startIfEnabled()
    webUi = new WorkerWebUI(this, workDir, webUiPort)
    webUi.bind()
    //向Master进行注册
    registerWithMaster()

    metricsSystem.registerSource(workerSource)
    metricsSystem.start()
    // Attach the worker metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(webUi.attachHandler)
  }
```

我们继续往后看Worker.registerWithMaster()

```scala
private def registerWithMaster() {
    // onDisconnected may be triggered multiple times, so don't attempt registration
    // if there are outstanding registration attempts scheduled.
    registrationRetryTimer match {
      case None =>
        registered = false
        // 会向所有的Master进行注册，
        // 需要补充的是在Spark中也存在Master的单点故障，所以也可以进行HA配置
        registerMasterFutures = tryRegisterAllMasters()
        connectionAttemptCount = 0
	......
    }
  }
```

继续查看 `tryRegisterAllMasters()`方法

```scala
 private def tryRegisterAllMasters(): Array[JFuture[_]] = {
    .....
            // 继续向Master注册
            registerWithMaster(masterEndpoint)
	......
  }
```

`registerWithMaster(masterEndpoint: RpcEndpointRef)`

```scala
  private def registerWithMaster(masterEndpoint: RpcEndpointRef): Unit = {
  	// 这里就是向Maser发送消息了，发送了一个RegisterWorker对象
    masterEndpoint.ask[RegisterWorkerResponse](RegisterWorker(
      workerId, host, port, self, cores, memory, webUi.boundPort, publicAddress))
      .onComplete {
        // This is a very fast action so we can use "ThreadUtils.sameThread"
        case Success(msg) =>
          Utils.tryLogNonFatalError {
            handleRegisterResponse(msg)
          }
        case Failure(e) =>
          logError(s"Cannot register with master: ${masterEndpoint.address}", e)
          System.exit(1)
      }(ThreadUtils.sameThread)
  }	
```

发送完消息多半就是去找Master接受消息的方法了

果然在Master中找到了一个`receiveAndReply()`方法

```scala
  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    //这里就是接受Work注册消息的地方了
    case RegisterWorker(
        id, workerHost, workerPort, workerRef, cores, memory, workerUiPort, publicAddress) => {	 
      // 首先日志输出一些Work的资源信息
      logInfo("Registering worker %s:%d with %d cores, %s RAM".format(
        workerHost, workerPort, cores, Utils.megabytesToString(memory)))
      // 由于是对所有的Master发送的消息，所以存在StandBy接受了消息
      // 这里是Master对自己的状态进行判断，如果自己是StandBy，则什么也不做
      if (state == RecoveryState.STANDBY) {
        context.reply(MasterInStandby)
      } else if (idToWorker.contains(id)) {
        //如果该Worker已经在注册表里面，同样是什么也没做
        context.reply(RegisterWorkerFailed("Duplicate worker ID"))
      } else {
        //最后经过重重判断，对传过来的Work信息进行注册
        val worker = new WorkerInfo(id, workerHost, workerPort, cores, memory,
          workerRef, workerUiPort, publicAddress)
        if (registerWorker(worker)) {
          persistenceEngine.addWorker(worker)
          context.reply(RegisteredWorker(self, masterWebUiUrl))
          schedule()
        } else {
          val workerAddress = worker.endpoint.address
          logWarning("Worker registration failed. Attempted to re-register worker at same " +
            "address: " + workerAddress)
          context.reply(RegisterWorkerFailed("Attempted to re-register worker at same address: "
            + workerAddress))
        }
      }
    }
```

其实到这里Start-All基本上就就结束了，后续的就是一些资源的调度了，会在后面继续进行分析

