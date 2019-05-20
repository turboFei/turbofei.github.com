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



