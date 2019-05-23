---
layout: post
category: spark
tagline: "Stay hungry, stay foolish~"
summary: 简单记录一下对spark的PR。Slowly, but always forward~
tags: [spark, PR, Apache]
---
{% include JB/setup %}

* toc
{:toc}


### Background ###

{{ page.summary }}

### SPARK-27637

#### Description

在spark中有两种shuffle client，自带的blockTransferService和externalShuffleClient。

如果spark.shuffle.service.enabled=true,那么spark使用external shuffle client进行拉取shuffle block data，使用 nettyBlockTransferService进行拉取broadcast block data。

如果spark.shuffle.service.enabled=false，那么shuffle block data 和broadcast block data 都由nettyBlockTransterService进行拉取。

ExternalShuffleService(ESS)是一个shuffle服务，它可以忽视executor是否存活，保管executor的shuffle数据，就算executor是dead，也可以去ESS去取数据，因此在开启ESS情况下，是可以回收executor的，因此，如果要使用动态分配executor，必须开启ESS。

而如果不使用ESS，那么就需要spark的executor自己去管理shuffle 数据，因此取shuffle数据就是通过直接连接BlockManager的ip:port进行。

Broadcast数据是首先executor会访问其他executor去取，如果无法取到，则去向driver进行请求，由于这些broadcast数据必定是由executor管理，而不能委托给ESS，因此必须需要使用nettyBlockTransferService,无论ESS是否开启。

在spark.shuffle.service.enabled=true且spark.executor.dynamicAllocation.enabled=true 时，由于executor可以被动态回收；如果在取broadcast数据的时候成功连接对应的executor，但是在开始取数据时候，executor被回收掉，那么必然造成取数据的失败。

Spark中有一个RetryingBlockFetcher，如果在连接失败之后，会抛出java.io.IOException，之后会进行校验是否shouldRetry。

```java
  private synchronized boolean shouldRetry(Throwable e) {
    boolean isIOException = e instanceof IOException
      || (e.getCause() != null && e.getCause() instanceof IOException);
    boolean hasRemainingRetries = retryCount < maxRetries;
    return isIOException && hasRemainingRetries;
  }
```

可以看出这里只是判断是否是IOException并且还有剩余的重试此处，如果满足，就进行重试，之后也是必然的失败。如果这里设置的最大重试为10次，超时为30s，那么这里将会浪费掉五分钟的时间。因此，这里的校验是不合理的。

#### Solution

解决方案很简单，简单描述一下：

在超时之后，如果是IOException，则向driverEndPoint发送rpc请求，判断这个executor是否存活。如果executor已经dead，则抛出ExecutorDeadException, 这就会造成 shouldRetry为false(因为isIOException == false). 核心代码如下: 具体参考PR.

```scala
         try {
            new OneForOneBlockFetcher(client, appId, execId, blockIds, listener,
              transportConf, tempFileManager).start()
          } catch {
            case e: IOException =>
              if (driverEndPointRef.askSync[Boolean](IsExecutorAlive(execId))) {
                throw e
              } else {
                throw new ExecutorDeadException(s"The relative remote executor(Id: $execId), " +
                  "which maintains the block data to fetch is dead.")
              }
          }
```
Ps: 感谢社区大佬对我代码的review，非常佩服他们的代码功力，受益匪浅.



#### Link
[ISSUE SPARK-27637](https://issues.apache.org/jira/browse/SPARK-27637)
[PR SPARK-27637](https://github.com/apache/spark/pull/24533)

### SPARK-27814

#### Description

在spark中有条件下推机制，条件下推可以大大加快sql的执行速度。

例如在spark源码里面 jdbc工具类获得一个query schema的方法代码如下。

```scala
@Since("2.1.0")
def getSchemaQuery(table: String): String = {
  s"SELECT * FROM $table WHERE 1=0"
}
```

这里加了一个`where 1=0`，就是为了将这个条件下推，加快查询速度。

在分区表情况下，如果一些过滤条件是加在分区key上面，这时进行条件下推可以直接跳过许多分区，但是对分区表的分区键进行谓词下推有一个限制，就是只能下推类型为`string`的分区键.

在spark2.4.* 某个PR之前，就拿spark2.3.2中的源码来说。

其对prediction转换为filter的方法如下:

`org.apache.spark.sql.hive.client.HiveShim ` 的`convertFilter`方法

```scala
      case op @ SpecialBinaryComparison(NonVarcharAttribute(name), ExtractableLiteral(value)) =>
        Some(s"$name ${op.symbol} $value")
```

解析下这段代码，如果这个prediction是一个 `attribute = 'value'`类型的表达式(也可以是> < != in， etc)，那么他就会判断一下这个列不是一个varchar类型的partition key， 因为hive不支持varchar类型的partition key下推，但是在spark catalyst里面，varchar也被表达为string。

而在master分支对之前的这部分逻辑进行了改动。

目前的方法为:

```scala
    case op @ SpecialBinaryComparison(
        ExtractAttribute(NonVarcharAttribute(name)), ExtractableLiteral(value)) =>
      Some(s"$name ${op.symbol} $value")

    object ExtractAttribute {
      def unapply(expr: Expression): Option[Attribute] = {
        expr match {
          case attr: Attribute => Some(attr)
          case Cast(child @ AtomicType(), dt: AtomicType, _)
              if Cast.canUpCast(child.dataType.asInstanceOf[AtomicType], dt) => unapply(child)
          case _ => None
        }
      }
    }
```

可以明显看出比之前多了一个`ExtractAttribute`方法，这个方法是用于操作cast操作。首先判断能不能cast，能cast就进行转换类型操作。之后再进行 NonVarcharAttribute判断。这个cast操作引入了一个风险。

以下是复现场景。

如果有一张表分区键为 c3 类型为`int`.

```sql
table test (c1 int, c2 string) partitioned by (c3 Int)
```

如果对c3进行cast操作查询，如下:

```sql
select * from test where (cast c3 as string)  = '0'
```

这个操作会首先进行int转string类型，这个转换是允许的，进行convertFilter之后变成filter `c3 ="0"`，注意这里的`"0"`加了双引号，是一个String。这会导致这个filter下推，而这时hive不允许的，因此会抛出以下异常.

```sql
Caused by: MetaException(message:Filtering is supported only on partition keys of type string)
```

#### Solution

解决方案:

```scala
    object ExtractAttribute {
      val partitionKeys = table.getPartitionKeys.asScala.map(_.getName).toSet

      def unapply(expr: Expression): Option[Attribute] = {
        var castToStr = false
        expr match {
          case attr: Attribute
              if (!castToStr || !partitionKeys.contains(attr.name) ||
                attr.dataType == StringType) =>
            Some(attr)
          case Cast(child @ AtomicType(), dt: AtomicType, _)
              if Cast.canUpCast(child.dataType.asInstanceOf[AtomicType], dt) =>
            castToStr = (castToStr || dt == StringType)
            unapply(child)
          case _ => None
        }
      }
    }
```

解决方案就是改变ExtractAttribute的unapply方法。首先记录，这个抽取过程中有没有发生转换为字符串的操作，因为这个cast都是向上转换，因为只要有一个转换为字符串的操作，都说明对最底层的attribute进行了转换为字符串的操作。因为在提取到最底层时候，进行三个判断。

- 1、有没有进行转换为字符串的操作，如果没有返回Some(attribute)，有则执行第2个判断
- 2、该attribute是不是partition key，如果不是则返回Some(attribute)，是则进行第三个判断
- 3、该attribute是不是原本就是StringType(catalyst类型)，是则返回Some(attribute)，否则，只能返回None。

如此，就解决了这个fatal的问题~

引申一下:下面的这条语句是不是造成下推出错的。

```sql
select * from test where (cast c3 as BigDecimal)  = '1.1'
```

可能要问不是说只有string才可以下推么，是的，所以这个prediction不会下推到分区过滤。我想是因为它会转化为filter `c3 = 1.1`后面的`1.1`还是一个数字类型，这导致其不会被下推到partitionFilter。而前面的cast as string的filter是`c3 ="0"`,后面的`"0"`是一个字符串，导致其会下推到partitionFilter，造成异常。

#### Link

[ISSUE SPARK-27814](https://issues.apache.org/jira/browse/SPARK-27814)

[PR SPARK-27814](https://github.com/apache/spark/pull/24685)

### SPARK-27562(In Progress)

**这个PR应该是改动太多了，不太好review，所以没有committer进行review，但是个人认为这个PR非常有意义。特别是针对之后的RemoteShuffleService(一个用于计算存储分离架构的ExternalShuffleService).**



#### Description

 参考[ISSUE SPARK-4105](https://issues.apache.org/jira/browse/SPARK-4105).

 shuffle是spark应用中一个重要的操作。 shuffle是map端的数据进行重新分区，然后reduce端去拉取每个map端对应的分区数据。因此在shuffle过程中，数据会进行网络传输。而网络传输面临着数据传输出错的风险，spark本身有一种校验shuffle传输数据的机制。

**在[SPARK-26089 对应PR](https://github.com/apache/spark/pull/23453)合入之前的校验机制**

这个机制是当shuffle数据有压缩编码，比如snappy，lz4时，判断shuffle数据的大小，如果数据大小小于一个阈值，比如16m，则对这个数据进行校验，校验方法为将数据输入流，拷贝到输出流再转为输入流，如果在流拷贝中没有出错，则表示数据没有损坏。

但是这个校验机制存在一定的限制：

1、 非压缩的数据无法校验（非压缩数据也存在传输出错风险）

2、shuffle数据超过阈值则无法校验

3、流拷贝消耗内存

因此，目前的校验机制存在shuffle数据传输出错，导致shuffle read 端task由于数据出错造成任务出错的风险。

**在[SPARK-26089 对应PR](https://github.com/apache/spark/pull/23453)合入之后的校验机制**

这个PR对shuffle校验机制进行了一些优化：

1. 针对大的shuffle block会校验开头的部分数据，如果没出错，则通过校验，进入执行阶段
2. 不再采用流拷贝操作，不会浪费内存。

但是也存在一些问题：

1. 如果大的shuffle block中间的数据出错，依然会造成task出错，而无法重新fetch
2. 依然不会对未使用codec的数据进行校验。

#### Solution

首先，我们选取crc作为我们的校验方式，crc同时也是hadoop使用的通信校验方式，它简单且快速。这也是我们对比了其他校验方式，例如md5， sha系列算法之后的结果。

在shuffle write阶段，我们在获得 mapTask对应的partitionedFile之后，根据索引，计算出每个分区的crc值，然后跟随各个分区的长度索引，一起写入到shuffle.index文件中。关于shuffle的机制可以参考[我之前的文章，shuffle源码分析](/spark/2016/12/26/spark源码分析Shuffle实现).

spark 的shuffle writer分为三种，bypassShuffleWriter， SortShuffleWriter以及UnsafeShuffleWriter。其中bypassShuffleWriter是在reducer的数量小于阈值(默认200)时候使用，他的特点是每个mapTask创建reducerNum个shuffle文件，所以需要在reducer个数小时使用，否则会造成很多小文件。

而SortShuffleWriter和UnsafeShuffleWriter都是只创建一个PartitionedFile。所以在根据每个partition长度进行计算crc值时是很快的。

之后我们将计算好的crc值与partition index一起写入shuffle 索引文件。

在之后fetchBlock时，我们将每个partition的crc值随数据一起发送， 然后在shuffle read端，对拉取到的数据进行重新计算crc值，与原来的crc进行比对，如果相同，则数据不存在问题。

在shuffle read端，数据一般都是在内存中，计算crc是很快的，在计算完之后，对这个内存中的inputStream进行reset操作，就可以重新进行后面的执行操作，如果数据是落在磁盘中，则代表数据较大，  crc的计算效率是经过实战考验的，我们也做了相应测试，由于文件创建的inputStream不支持reset操作，我们在计算完crc值之后，重新根据文件创建inputStream.

这一套方案可以校验所有的数据，不论他是否进行压缩，是否太大， 都可以很好的计算，并且经过我们的是，性能没有下降。

具体还有其他细节:

- 原来的indexFile是写partitionNum+1 个long值，长度为8的倍数，crc值也是long值，我们加入一个1位的标志位，来区别是否进行了crc计算。
- 在写crc时的一致性保证。
- 在shuffle read端在发现crc值与原来的crc不同时的处理等等。

#### Link

 [ISSUE SPARK-27562](https://issues.apache.org/jira/browse/SPARK-27562)

[PR SPARK-27562](https://github.com/apache/spark/pull/24447)

### SPARK-27716(In Progress)

#### Description

使用jdbc 的DataSource，我们可以将一个RDD保存到database中。比如mysql，greenplum。

这些操作的主要逻辑由`JdbcUtils`中的`saveTable`函数完成,该函数的注释如下.

```
  /**
   * Saves the RDD to the database in a single transaction.
   */
  def saveTable(
      df: DataFrame,
      tableSchema: Option[StructType],
      isCaseSensitive: Boolean,
      options: JdbcOptionsInWrite)
```

事实上这个注释的描述是错的，注释中说saveTable操作是在单个事务中完成。

其实saveTable是调用savePartition操作来将每个分区的数据保存在数据库中，而savePartition的操作是在单个事务完成，而针对整体的saveTable却并不是单个事务。

可能会存在某个分区出错，但是saveTable却处于一个中间状态的情况，这不符合事务的要求。

在使用jdbc的数据传输操作可以分为以下几种。

- case1: append数据到一个已经存在的表中。
- case2：overwrite一张表，但是这个表是级联表，如果将这个表drop会牵连到其他的表，因此只能将这个表清空，再append数据。
- case3：overwrite一张表，该表存在且不是级联表，因此可以先drop表。
- case4：要保存的表不存在。

在这个PR中，我对case3和case4进行了事务支持。

#### Solution

针对case3和case4，我们可以先将数据保存到一个临时表中。

我们使用一个累加器来记录成功savePartition的分区数。累加器是spark中的一个分布式的计数器。

来每个分区都执行savePartition之后，我们拿累加器的值和分区的数目做比较，如果所有分区都成功的savePartition，那么我们可以在driver上面drop destination table if exists,然后对临时表做rename操作，将其rename到最终的表。

由于drop table 和rename table 都是原子操作，所以我们可以保证case3和case4是在单事务进行。

如果有分区savePartition失败，那么我们将在driver端抛出一个分区失败的异常，提醒user。

在finally模块，我们将会进行删除临时表的操作，最大重试次数为三次，确保不会产生一些垃圾数据。

#### Link

[PR SPARK-27716](https://github.com/apache/spark/pull/24610)

 [ISSUE SPARK-27716](https://issues.apache.org/jira/browse/SPARK-27716)

### To Be Continued~
