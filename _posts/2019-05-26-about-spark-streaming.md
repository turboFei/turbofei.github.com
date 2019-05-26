---
layout: post
category: spark
tagline: ""
summary: 简单讲解下spark streaming, structed streaming
tags: [spark,streaming]
---
{% include JB/setup %}
目录
* toc
{:toc}
### Background ###

{{ page.summary }}

我们都知道spark中有两种streaming，一种是spark streaming，另一种是structed streaming。spark streaming是微批处理，隔一段时间提交一批job，底层走的还是rdd。

而structed streaming是spark为了满足低时延的需求，重新设计的一套流式处理机制。相关的PR是[SPARK-29028 SPIP: Continuous Processing Mode for Structured Streaming](https://issues.apache.org/jira/browse/SPARK-20928).

### Spark Streaming

首先讲一下微批的streaming。这种streaming 使用`DStream`进行操作，其API与RDD编程类似。其对应的Context为StreamingContext.

#### StreamingContext

构造如下:

```scala
    val ssc = new StreamingContext(sparkConf, Seconds(2))
```

其底层也是会创建一个SparkContext，只不过StreamingContext提供了一些streaming编程的Api。可以看到后面的2s是微批的频率，每2秒钟触发一次批处理。

下面是一个HDFSWordCount的例子.

```scala
    StreamingExamples.setStreamingLogLevels()
    val sparkConf = new SparkConf().setAppName("HdfsWordCount")
    // Create the context
    val ssc = new StreamingContext(sparkConf, Seconds(2))
    // Create the FileInputDStream on the directory and use the
    // stream to count words in new files created
    val lines = ssc.textFileStream(args(0))
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()
```

可以看到是先指定好调度频率为2s，然后指定每个批次要执行的动作，然后调用start方法开始处理。

下面是start方法的核心代码：

```
ThreadUtils.runInNewThread("streaming-start") {
 sparkContext.setCallSite(startSite.get)
 sparkContext.clearJobGroup()
 sparkContext.setLocalProperty(SparkContext.SPARK_JOB_INTERRUPT_ON_CANCEL, "false")
 savedProperties.set(SerializationUtils.clone(sparkContext.localProperties.get()))
 scheduler.start()
}
```

可以看到是启动一个新的线程，是为了在设置callsite 以及job group这些 thread local时候不影响当前线程。

然后这里有一个scheduler.start, 这是 streaming任务的核心， JobScheduler.

#### JobScheduler

下面是JobScheduler的start方法:

```scala
  def start(): Unit = synchronized {
    if (eventLoop != null) return // scheduler has already been started

    logDebug("Starting JobScheduler")
    eventLoop = new EventLoop[JobSchedulerEvent]("JobScheduler") {
      override protected def onReceive(event: JobSchedulerEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = reportError("Error in job scheduler", e)
    }
    eventLoop.start()

    // attach rate controllers of input streams to receive batch completion updates
    for {
      inputDStream <- ssc.graph.getInputStreams
      rateController <- inputDStream.rateController
    } ssc.addStreamingListener(rateController)

    listenerBus.start()
    receiverTracker = new ReceiverTracker(ssc)
    inputInfoTracker = new InputInfoTracker(ssc)

    val executorAllocClient: ExecutorAllocationClient = ssc.sparkContext.schedulerBackend match {
      case b: ExecutorAllocationClient => b.asInstanceOf[ExecutorAllocationClient]
      case _ => null
    }

    executorAllocationManager = ExecutorAllocationManager.createIfEnabled(
      executorAllocClient,
      receiverTracker,
      ssc.conf,
      ssc.graph.batchDuration.milliseconds,
      clock)
    executorAllocationManager.foreach(ssc.addStreamingListener)
    receiverTracker.start()
    jobGenerator.start()
    executorAllocationManager.foreach(_.start())
    logInfo("Started JobScheduler")
  }
```

主要启动了以下组件:

- receiverTracker 用于接受数据，例如接收从kafka发送的数据
- inputInfoTracker 统计输入信息，用于监控
- jobGenerator 用于job生成，每个时间间隔生成一批job
- executorAllocationManager executor动态分配管理器

##### ExecutorAllocationManager

是否打开由`spark.streaming.dynamicAllocation.enabled`控制。可以看出和spark core中的参数很像。也新加了几个参数.


| 参数                                               | 说明             | 默认值 |
| -------------------------------------------------- | ---------------- | ------ |
| spark.streaming.dynamicAllocation.scalingInterval  | 动态分配调整间隔 | 60s    |
| spark.streaming.dynamicAllocation.scalingUpRatio   | ratio上限        | 0.9    |
| spark.streaming.dynamicAllocation.scalingDownRatio | ratio下限        | 0.3    |

这个动态分配管理器和Core中的有何不同呢？

在core中的管理器是基于空闲时间来控制回收这些executor，而在流处理这些微批中，一个executor空闲是不太可能的，因为每隔很少的时间都会有一批作业被调度，那么在streaming里面如何控制executor的分配和回收呢？

基本策略是基于每批作业处理的时间来确定是否是idle-ness.

- 使用streamingListener来获得每批jobs 处理的时间。
- 周期性的(spark.streaming.dynamicAllocation.scalingInterval)拿jobs处理时间和调度周期做对比。
-  如果 平均处理时间/调度间隔 >=  ratio上限，则调大executor数量。
-  如果 平均处理时间/调度间隔 =<  ratio下限，则调小executor数量。

默认上限是0.9，下限为0.3。即如果间隔为2s，如果平均处理时间大于等于1.8s，那么就要调大executor；如果平均处理时间小于等于0.6s，那么就要调小executor数量。

##### 调度

之后的过程就不详细讲了。最近也在做一个跟streamingListener有关的项目，其实了解调度可以从StreamingListener入手，可以看下listener都记录哪些事件。

```scala
trait StreamingListener {

  /** Called when the streaming has been started */
  def onStreamingStarted(streamingStarted: StreamingListenerStreamingStarted) { }

  /** Called when a receiver has been started */
  def onReceiverStarted(receiverStarted: StreamingListenerReceiverStarted) { }

  /** Called when a receiver has reported an error */
  def onReceiverError(receiverError: StreamingListenerReceiverError) { }

  /** Called when a receiver has been stopped */
  def onReceiverStopped(receiverStopped: StreamingListenerReceiverStopped) { }

  /** Called when a batch of jobs has been submitted for processing. */
  def onBatchSubmitted(batchSubmitted: StreamingListenerBatchSubmitted) { }

  /** Called when processing of a batch of jobs has started.  */
  def onBatchStarted(batchStarted: StreamingListenerBatchStarted) { }

  /** Called when processing of a batch of jobs has completed. */
  def onBatchCompleted(batchCompleted: StreamingListenerBatchCompleted) { }

  /** Called when processing of a job of a batch has started. */
  def onOutputOperationStarted(
      outputOperationStarted: StreamingListenerOutputOperationStarted) { }

  /** Called when processing of a job of a batch has completed. */
  def onOutputOperationCompleted(
      outputOperationCompleted: StreamingListenerOutputOperationCompleted) { }
}
```

前面与receiver相关的我们不管，只看跟处理有关的。

- BatchSubmitted 是代表提交了一批jobs,对应的是一个JobSet
- BatchStartted，当对应的jobSet里面的第一个job开始执行时候触发
- BatchCompleted，当对应的jobSet里面的所有job都完成时触发
- OutputOperationStarted 这是对应一个job的开始
- OutputOperationCompleted 对应一个job的完成

下面是一批jobs 信息的数据结构。

```scala
case class BatchInfo(
    batchTime: Time,
    streamIdToInputInfo: Map[Int, StreamInputInfo],
    submissionTime: Long,
    processingStartTime: Option[Long],
    processingEndTime: Option[Long],
    outputOperationInfos: Map[Int, OutputOperationInfo]
  ) 
```

可以看到每个batchInfo对应的key就是一个batchTime，这是独一无二的，最后面有一个outputOperationInfos，这是对应里面每个job的信息，里面包含每个job的failureReason，如果那个job出错的话。

之后就没啥说的了，最终那些streaming的job还是走的底层RDD，这就和普通的批任务没区别了。

### Structed Streaming

在spark Streaming中，最小的可能延迟受限于每批的调度间隔以及任务启动时间。因此，这不能满足更低延迟的需求。

如果能够连续的处理，尤其是简单的处理而没有任何的阻塞操作。这种连续处理的架构可以使得端到端延迟最低降低到1ms级别，而不是目前的10-100ms级别.

#### 基本概念

介绍下Epoch, waterMark

EpochTracker是使用一个AtomicLong来计算EpochID，而其`incrementCurrentEpoch`方法只有在`ContinuousCoalesceRDD`和`ContinuousWriteRDD`中被调用。也就是说只有在进行类似于shuffle 和action的时候才被调用，所以**Epoch**类似于RDD执行中的StageId。

而**waterMark**是一个标记，代表在这个时间点之前的数据全部都已经完成。

#### Example

Structed Streaming的Api 和sql比较类似。下面是一个StructuredNetworkWordCount 的例子.

```scala
object StructuredNetworkWordCount {
  def main(args: Array[String]) {
    if (args.length < 2) {
      System.err.println("Usage: StructuredNetworkWordCount <hostname> <port>")
      System.exit(1)
    }

    val host = args(0)
    val port = args(1).toInt

    val spark = SparkSession
      .builder
      .appName("StructuredNetworkWordCount")
      .getOrCreate()

    import spark.implicits._

    // Create DataFrame representing the stream of input lines from connection to host:port
    val lines = spark.readStream
      .format("socket")
      .option("host", host)
      .option("port", port)
      .load()

    // Split the lines into words
    val words = lines.as[String].flatMap(_.split(" "))

    // Generate running word count
    val wordCounts = words.groupBy("value").count()

    // Start running the query that prints the running counts to the console
    val query = wordCounts.writeStream
      .outputMode("complete")
      .format("console")
      .start()

    query.awaitTermination()
  }
}
```

可以看到这个写法就像是DataSource一样。

readStream 和 writeStream 对应的是`DataStreamReader`和`DataStreamWriter`.

#### To Be Continued