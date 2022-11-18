---
layout: post
category: spark
tagline: ""
summary: ebay对于Kyuubi Spark 引擎big result sets场景做的一些优化
tags: [kyuubi,spark]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 前言

[eBay基于kyuubi构建spark服务的gateway](https://www.bilibili.com/video/BV1oa411R7BC/)，而最常见的场景就是Spark SQL, 主要分为以下两种:

1. 用户在BI 工具上面执行sql语句，执行结果会被截断(kyuubi.operation.result.max.rows)，只取几千条结果返回。
2. 用户使用JDBC拉取全部的执行结果，可能会有几千万条。

对于第二种场景，如果sql的执行结果集特别大，如果Spark driver将所有计算结果保存在内存中，那么Spark driver会成为瓶颈，非常容易产生OOM。

因此Kyuubi社区提供了一个[针对大计算结果集的一个方案](https://kyuubi.apache.org/docs/latest/deployment/spark/incremental_collection.html?highlight=big%20result), 可以将`kyuubi.operation.incremental.collect` 设为true来将原本调用dataFrame的 `collect`方法改为调用`toLocalIterator` 方法。

我们都知道dataFrame是分partition的，每个partition对应一个task去执行。`collect`方法是将所有task并行执行，然后收集结果。而`toLocalIterator`是一个lazy 操作，它是一次计算一个partition的结果，只有在client端需要读取下一部分结果时候，才会计算下一个partition的结果，也就是将所有task串行执行。

虽然它保护了Spark driver, 不容易OOM，但是性能大部分时间很差，特别是用户的sql比较复杂时候，大大的拉长计算时间。

本文讲下，我们针对big result sets场景下做的一些优化。



### 问题分析

Kyuubi Spark 大计算结果集场景下，主要有以下问题

- 不能将所有结果都保存在内存中，防止OOM，而且不能入侵Spark内核，去做一些计算结果spill的优化
- 如果使用incremental collect, 延迟计算导致运行时间大大拉长



对于第一个问题，不能全部放在内存，我们可以将计算结果落地。

对于第二个问题，延迟计算导致性能问题，我们可以进行预计算，将结果整齐的落入文件，然后在incremental 拉取结果时，整齐的从文件中读取。

所以，我们做了一些优化，将sql的计算结果，进行预计算，整齐的存入hdfs文件，然后在后续的拉取中，依然采用incremental 拉取，但是尽量的减少拉取时候split的partition(task)数量，进行快速拉取。



### SQL 分类

用户的sql分为以下两种：

1. 没有排序操作
2. 有排序操作，在后续从文件中读取结果返回时候，依然要保持顺序



对于不需要排序的sql，我们可以直接将计算结果落盘，但是对于需要排序的sql，我们不能直接落盘，因为直接落盘之后，再重新读取，顺序是不能保证的。



### Spark读取文件过程

下面是 `org.apache.spark.sql.execution.DataSourceScanExec::createNonBucketedReadRDD`的代码，用于在非bucket读取时候创建RDD.

这里面有两个参数，一个是 maxSplitBytes, 代表一个partition(task)最大处理的bytes 大小，另一个是openCostInBytes,代表打开一个文件所需要的开销。

可以看到这个方法会把selectedPartitions里面的每个文件，按照maxSplitBytes进行split(如果支持split), 然后flatMap展开，之后按照split之后的length，从大到小排序。

然后将排序好的splitFiles，构建partitions，构建过程就是把排好序的splitFiles进行合并，如果`currentSize + file.length > maxSplitBytes`，那么就把current选择的splitFile(s)作为一个partition，然后两个splitFile 合并之间有一个openCostInBytes的开销。

所以，即使我们将计算结果有序的写入多个数据文件中，再次读取的时候，这些结果的顺序也会被打乱。

```scala
  /**
   * Create an RDD for non-bucketed reads.
   * The bucketed variant of this function is [[createBucketedReadRDD]].
   *
   * @param readFile a function to read each (part of a) file.
   * @param selectedPartitions Hive-style partition that are part of the read.
   * @param fsRelation [[HadoopFsRelation]] associated with the read.
   */
  private def createNonBucketedReadRDD(
      readFile: (PartitionedFile) => Iterator[InternalRow],
      selectedPartitions: Array[PartitionDirectory],
      fsRelation: HadoopFsRelation): RDD[InternalRow] = {
    val openCostInBytes = fsRelation.sparkSession.sessionState.conf.filesOpenCostInBytes
    val maxSplitBytes =
      FilePartition.maxSplitBytes(fsRelation.sparkSession, selectedPartitions)
    logInfo(s"Planning scan with bin packing, max size: $maxSplitBytes bytes, " +
      s"open cost is considered as scanning $openCostInBytes bytes.")

    val splitFiles = selectedPartitions.flatMap { partition =>
      partition.files.flatMap { file =>
        // getPath() is very expensive so we only want to call it once in this block:
        val filePath = file.getPath
        val isSplitable = relation.fileFormat.isSplitable(
          relation.sparkSession, relation.options, filePath)
        PartitionedFileUtil.splitFiles(
          sparkSession = relation.sparkSession,
          file = file,
          filePath = filePath,
          isSplitable = isSplitable,
          maxSplitBytes = maxSplitBytes,
          partitionValues = partition.values
        )
      }
    }.sortBy(_.length)(implicitly[Ordering[Long]].reverse)

    val partitions =
      FilePartition.getFilePartitions(relation.sparkSession, splitFiles, maxSplitBytes)

    new FileScanRDD(fsRelation.sparkSession, readFile, partitions)
  }
```



### 针对不需要排序的SQL结果落地

对于不需要排序的sql，我们可以直接将计算结果进行落地，但是为了整齐的存入，整齐的读取，我们加入了一些参数。

1. minFileSize, 这个参数代表我们可以接受的写出文件平均大小
2. fileCoalesceNumThreshold， 这个参数代表我们对文件进行合并的文件数量阈值，如果写出的文件平均size小于minFileSize, 且文件个数大于这个阈值，我们将会对写出文件进行合并，期待合并的文件大小是下面第三个参数`partitonBytes`的值，算出合并之后文件个数之后，将前面写出文件读出再写入到`Coalesce`路径。
3. partitonBytes， 这个参数代表，我们在读取落地的计算结果时候，每个partition(task)处理的文件bytes



写入过程大概如下:

1. 根据sql query的schema, 创建一张external parquet分区表(分区键是一个unique的string, 表路径在sessionScrathPath之下)
2. 将sql的结果写入到这张表中
3. 拿到表路径的content summary, 得到写出文件数量和size，如果复合上面提到的`Coalesce`条件，则对这些文件进行合并，做到整齐的写入

这里我们之所以创建分区表，是为了在job 完成commit files时候，可以只用rename 一个partition path而不是去rename所有文件，来减少对hdfs namenode的rpc。



关于读取过程，先看下Spark代码 `org.apache.spark.sql.execution.datasources.FilePartition::maxSplitBytes`.

第一个参数 `defaultMaxSplitBytes` 是从`spark.sql.files.maxPartitionBytes`中获得，默认128M。

第二个参数`openCostInBytes` 是从`spark.sql.files.openCostInBytes`中获得，默认4M。

第三个参数 `minPartitionNum`, 是首先从`spark.sql.files.minPartitionNum`中获得，如果未设置，则取`spark.default.parallelism`的值，默认是当前`cores* 2`。

然后`totalBytes`是要读取文件的size之和， `bytesPerCore` 是 `totalBytes/minPartitionNum`.

最后`maxSplitBytes`的结果，会取 `defaultMaxSplitBytes` 和 `Math.max(openCostInBytes, bytesPerCore)` 之中的最小值。

也就是说当我们计算资源很丰富，`cores`很大时候，`bytesPerCore` 会很小，导致得到的 `maxSplitBytes` 会很小。

比如说，当我们要incremental读取写出去的400M计算结果，而当前cores数量是80， 那么 `bytesPerCore`是5M，默认参数情况下，我们得到的`maxSplitBytes` split size也就是5M，那么Spark会至少分配80个task 去串行的读取这个计算结果。

Spark这样做的目的是为了最大化利用`cores`来快速并行执行，而我们在incremental collect时候，task都是串行。默认情况下(`spark.sql.files.minPartitionNum`未设置), 这会造成，读取计算的结果的时间大大拉长。

```scala
  def maxSplitBytes(
      sparkSession: SparkSession,
      selectedPartitions: Seq[PartitionDirectory]): Long = {
    val defaultMaxSplitBytes = sparkSession.sessionState.conf.filesMaxPartitionBytes
    val openCostInBytes = sparkSession.sessionState.conf.filesOpenCostInBytes
    val minPartitionNum = sparkSession.sessionState.conf.filesMinPartitionNum
      .getOrElse(sparkSession.sparkContext.defaultParallelism)
    val totalBytes = selectedPartitions.flatMap(_.files.map(_.getLen + openCostInBytes)).sum
    val bytesPerCore = totalBytes / minPartitionNum

    Math.min(defaultMaxSplitBytes, Math.max(openCostInBytes, bytesPerCore))
  }
```



所以针对计算结果读取过程，我们做的优化如下:

1. 根据计算结果的totalSize和上面说的`partitonBytes` ，将 `totalSize/partitonBytes` 的结果用来临时去set `spark.sql.files.minPartitionNum`（会在operation 结束之后还原), 这样我们按照`partitionBytes`去控制incremental读取计算结果的partition(task)数量，整齐的读取



### 针对需要排序的SQL计算结果落地

前面说过，将计算结果落地之后再读取，不能保证有序。

除非我们将排序的SQL结果写入一个文件，并且写入的结果要有序。



所以，针对需要排序的SQL计算结果落地，我们需要把计算结果写入一个文件，而这必然要求我们设置一个阈值，我将之命名为 `sortLimitThreshold`, 默认为100万条。只有当需要排序SQL的计算结果小于这个阈值时，才会将计算结果落地。同时我们也加了一个参数为`sortLimitEnabled`.

此处我们面临两个问题

1. 如何将结果只写入一个文件
2. 且保证文件内容有序



对于问题1， 我们只需在原有sql基础上，加上一个 `limit $count` 即可让其结果只输出到一个文件。

对于问题2，我们需要借助Spark 里面的`TakeOrderedAndProjectExec`来保证输出文件内容有序。



关于`TakeOrderedAndProjectExec`,  从代码中可以看出只有当 limit < `spark.sql.execution.topKSortFallbackThreshold`的值时候，才会用`TakeOrderedAndProjectExec`.

```scala
  /**
   * Plans special cases of limit operators.
   */
  object SpecialLimits extends Strategy {
    override def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
      case ReturnAnswer(rootPlan) => rootPlan match {
        case Limit(IntegerLiteral(limit), Sort(order, true, child))
            if limit < conf.topKSortFallbackThreshold =>
          TakeOrderedAndProjectExec(limit, order, child.output, planLater(child)) :: Nil
        case Limit(IntegerLiteral(limit), Project(projectList, Sort(order, true, child)))
            if limit < conf.topKSortFallbackThreshold =>
          TakeOrderedAndProjectExec(limit, order, projectList, planLater(child)) :: Nil
        case Limit(IntegerLiteral(limit), child) =>
          CollectLimitExec(limit, planLater(child)) :: Nil
        case Tail(IntegerLiteral(limit), child) =>
          CollectTailExec(limit, planLater(child)) :: Nil
        case other => planLater(other) :: Nil
      }
      case Limit(IntegerLiteral(limit), Sort(order, true, child))
          if limit < conf.topKSortFallbackThreshold =>
        TakeOrderedAndProjectExec(limit, order, child.output, planLater(child)) :: Nil
      case Limit(IntegerLiteral(limit), Project(projectList, Sort(order, true, child)))
          if limit < conf.topKSortFallbackThreshold =>
        TakeOrderedAndProjectExec(limit, order, projectList, planLater(child)) :: Nil
      case _ => Nil
    }
  }

  val TOP_K_SORT_FALLBACK_THRESHOLD =
    buildConf("spark.sql.execution.topKSortFallbackThreshold")
      .internal()
      .doc("In SQL queries with a SORT followed by a LIMIT like " +
          "'SELECT x FROM t ORDER BY y LIMIT m', if m is under this threshold, do a top-K sort" +
          " in memory, otherwise do a global sort which spills to disk if necessary.")
      .version("2.4.0")
      .intConf
      .createWithDefault(ByteArrayMethods.MAX_ROUNDED_ARRAY_LENGTH)
```





写入过程如下:

1. 如果`sortLimitEnabled`是true，那么去拿到`spark.sql(statement).count`的结果`rowCount`, 如果其小于`sortLimitThreshold`, 那么在原有`statement` 基础之上加上`limit $rowCount`, 以确保其结果输出到单个文件
2. 临时 set `spark.sql.execution.topKSortFallbackThreshold` 为rowCount, 以确保其使用`TakeOrderedAndProjectExec`，确保其输出文件内容有序



### 总结

对于big result sets场景中，为了让服务更加稳定，我们对其结果进行预计算，将计算结果整齐的落入文件，然后在读取时候，整齐的读出，减少split的partition 数量，可以极大的提高用户query的稳定性和性能。
