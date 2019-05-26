---
layout: post
category: spark
tagline: ""
summary: 关于spark sql 的execution部分源码解析
tags: [spark, sql]
---
{% include JB/setup %}
目录
* toc
{:toc}
### Background ###

{{ page.summary }}

### 关于spark-sql模块

spark-sql模块下的代码作用有:

- sparkSession，SQLContext，DataSet，DataFrameWriter和reader等类
- api 包: python，r的api
- catalog包: catalog以及column，table，function，database的接口类
- expression包: 包括aggregator，UDF， UDAF以及窗口函数
- internal包：包括catalogImpl， HiveSerde，sessionState，sharedState等internal类
- jdbc包： 里面都是方言类 dialect
- sources包: 用于下推至DataSource的filter以及一些DataSource和sql relation相关接口
- **streaming**包: DataStreamReader， writer，StreamingQueryException等于streaming有关的类。
- util包: QueryExecutionListener类
- **execution**包：本文的重点

#### execution包

该包下源码分为:

- 直接在根目录下
  - 基本物理操作: coalesceExec, filterExec, projectExec, unionExec, RangeExec, and etc.
  - cacheManager：用于cache table.
  - 一些exec: sort, DataSourceScan
  - wholeStageCodegenExec
  - SparkPlanner用于生产物理计划的
  - limit以及其他操作
- aggregate包: 当然是与聚合相关的exec 以及UDAF
- arrow包: arrow也是一种列式存储格式，这个包有他的 writer以及工具类.
- columnar包: 与列状态，访问，类型相关的类
- command包: 一些命令,ddl, analyzeTableCommand, functions, create等命令
- DataSources包: 与DataSource有关，parquet, jdbc, orc等等
  - 关于DataSource options， writer， 工具类等等
- exchange： 里面有broadcastExchangeExec， exchangeCoordinator，shuffleExchangeExec等相关类，后面会重点分析这个,其实在物理计划中转换的重点就是这部分，所以ensureRequirements就在这个包中。
- joins: join相关的， hashJoin, broadcastHashJoin, broadcastNestedLoopJoin, sortMergeJoin,shuffleHashJoinExec.
- **streaming**包: 应该是continuous streaming相关的底层实现，应该不是走rdd，是一个真正流式的实现.
- window包: 窗口函数相关exec。
- python包，r包，metric包。



下文重点分析 **aggregate**, **exchange**, **joins**包。

### exchange包

#### Exchange & ExchangeCoordinator

首先讲一下Exchange，顾名思义就是交换，是为了在多个线程之间进行数据交换，完成并行。

Exchange分为两种，一种是BroadcastExchange另外一种是ShuffleExchange。Broadcast就是将数据发送至driver，然后由driver广播，这适合于数据量较小时候的shuffle。另一种ShuffleExchange就比较常见了，就是多对多的分发。

如果开启了`spark.sql.adaptive.enabled`，也就是自适应执行，

那么在使用ShuffleExchange的时候有对应的ExchangeCoordinator；如果没开启ae，那就不需要协调器。

ExchangeCoordinator顾名思义，是Exchange协调器，是一个用于决定怎么在stage之间进行shuffle数据的coordinator。这个协调器用于决定之后的shuffle有多少个partition用来需要fetch shuffle 数据。

一个coordinator有三个参数.

- numExchanges 
- targetPostShuffleInputSize
- minNumPostShufflePartitions

第一个参数是用于表示有多少个ShuffleExchangeExec需要注册到这个coordinator里面。因此，当我们要开始真正执行时，我们需要知道到底有多少个ShuffleExchangeExec。

第二个参数是表示后面shuffle阶段每个partition的输入数据大小，这是用于adaptive-execution的，用于推测后面的shuffle阶段需要多少个partition。可以通过`spark.sql.adaptive.shuffle.targetPostShuffleInputSize`来配置。

第三个参数是一个可选参数，表示之后shuffle阶段最小的partition数量。如果这个参数被定义，那么之后的shuffle阶段的partition数量不能小于这个值。

Coordinator的流程如下：

- 在一个`SparkPlan`执行之前，对于一个`ShuffleExchangeExec`操作，如果它被指定了一个`coorinator`，那么它将会注册到这个协调器，这发生在`doPrepare`方法里.
- 当开始执行`SparkPlan`，注册到这个协调器里面的ShuffleExchangeExec将会调用`postShuffleRDD`方法来相应的 post-shuffle `ShuffledRowRDD`.如果这个协调器已经决定了如何去shuffle data，那么这个Exec会马上获得它对应的`ShuffledRowRDD`.
- 如果这个coordinator已经决定了如何shuffle data，它会让注册到自己的`ShuffleExchangeExec`s来提交pos-shuffle stage。然后基于pre-shuffle阶段partition的统计信息，来决定post-shuffle的partition数量，如果post-shuffle需要，它也会将一些需要的连续的partitions放在一起发送给post-shuffle.
- 最后，这个coordinator会为所有注册的`ShuffleExchangeExec`创建post-shuffle `ShuffledRowRDD`。

目前的ae是比较老版本的ae，intel有一个ae项目，相信会在spark-3.0之后会合入，可以了解一下新的ae。

#### EnsureRequirements

这就是前面说的在SparkPlan 之前的do-prepare，需要为sparkplan的可执行做一些准备工作。

主要分为以下几部分:

- 确定distribution和Ordering
- 确定join 的条件和join keys出现顺序匹配
- 创建coordinator
  - ae开启
  - 支持Coordinator
    - 有ShuffleExchangeExec且是HashPartitionings
    - 支持Distribution并且child个数大于1
  - 针对ShuffleExchangeExec创建coordinator
  - 针对一个post-shuffle 对应几个pre-shuffle的创建coordinator，例如join 一个对应多个pre-shuffle，而几个pre-shuffle有不同的分区划分方式

#### Join

这里的join是执行阶段的join，不是解析阶段的join。

join包括BroadcastHashJoinExec, ShuffledHashJoinExec以及SortMergeSortMergeJoinExec。

join策略的选择`SparkStrategies`类里面，面临join，首先会尝试BroadcastJoin，然后是ShuffledHashJoin最后才是SortMergeHashJoin,只要能满足前面的条件就会优先使用前面的join。

而这些JoinExec是在前面构建物理之前进行构建，在之后真正执行物理计划的时候执行。



#### Aggregate

Agg和join有些类似，都是需要进行shuffle操作，但不同的是Aggregate可以是一个一元操作，而join是多元操作。

Aggregate操作例如 max, count, min, sum，groupBy， reduce, reduceBy 以及一些UDAF(User Defined Aggregate Function)。而groupBy这些操作可能面临order by.

### Conclusion

本文大概讲解了下execution包中各个部分的用途，重点是如何进行Exchange，以及什么是ExchangeCoordinator。对于join和Aggregate未涉及太多。