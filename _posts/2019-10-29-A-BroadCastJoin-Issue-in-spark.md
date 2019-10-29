---
layout: post
category: spark
tagline: ""
summary: Spark中有三种Join, BroadcastJoin, ShuffleHashJoin, SortMergeJoin。而BroadcastJoin通常认为是一种较为轻量的Join，因为其不走shuffle，本文描述一个与BroadcastJoin相关比较诡异的Issue。
tags: [spark,sql]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 前言

在Spark中，根据Join的物理执行方式来划分种类，可以分为以下三种。

- BroadcastJoin: 适用于一个极小表和一个大表的Join，其会把极小表由driver给各个executor，而不会触发Shuffle，而shuffle往往是任务的瓶颈所在，因此通常Broadcast被认为是一种十分轻量的Join。
- ShuffleHashJoin:适用于一个小表和一个大表进行Join，会触发shuffle。
- SortMergeJoin：适用于两个大表进行Join，其首先会对两个表的数据进行划分partition排序，然后把相应的分区进行发送到task端进行merge执行，因此称之为SortMergeJoin。

本文讲一个和BroadcastJoin相关的比较诡异的issue。



### BroadcastJoin
前面提到BroadcastJoin是在一个极小表和一个大表进行Join时候选择的join方式，由于BroadcastJoin不需要进行shuffle，所以大家比较喜欢这种Join方式。但是由于Broadcast是需要将数据拉取到driver然后分发到各个executor，因此driver内存是一个瓶颈。前面也提到是极小表，那么是多小的表才会使用这种Join呢？

Spark中有一个参数称之为`spark.sql.autoBroadcastJoinThreshold`, 其代表当这个表在磁盘上size小于这个值时，会使用BroadcastJoin，而如果我们将其设为-1，代表disable BroadcastJoin（官方文档的解释)。

除了该threshold之外，Broadcast还有一个限制，就是广播的表的行数不能超过512 milions行，也就是5亿多行，这个值是hard code的。也就是说即使表的磁盘物理size小于threshold，条数超过这个行数也不能进行BroadcastJoin。



### 问题描述

有一天我们遇到的一个Issue。其异常信息如下:

```bash
Caused by: org.apache.spark.SparkException: Cannot broadcast the table with more than 512 millions rows: 620880056 rows
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1$$anonfun$apply$1.apply(BroadcastExchangeExec.scala:78)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1$$anonfun$apply$1.apply(BroadcastExchangeExec.scala:73)
	at org.apache.spark.sql.execution.SQLExecution$.withExecutionId(SQLExecution.scala:97)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1.apply(BroadcastExchangeExec.scala:72)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1.apply(BroadcastExchangeExec.scala:72)
	at scala.concurrent.impl.Future$PromiseCompletingRunnable.liftedTree1$1(Future.scala:24)
	at scala.concurrent.impl.Future$PromiseCompletingRunnable.run(Future.scala:24)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```



看到这个异常的第一反应就是去查询`spark.sql.autoBroadcastJoinThreshold`的值，然后查到的结果100m。当时的想法是，为什么这个表在磁盘上不到100m大小，而其有6亿行数据，难道是列数十分少，并且压缩的特别的严重，不像是生产环境中的表。

当时没有产生其他怀疑，就建议用户将`spark.sql.autoBroadcastJoinThreshold`设置为-1，禁用掉BroadcastJoin。

过了段时间，用户回复说设置了之后仍然报上面的异常。当时确认了线上的参数设置的确是-1.

用户的sql语句格式为:

```sql
select a.*, b.*, c.* 
from
a left join b left join c 
on
a.a1=b.b1 and b.b2=c.c1;
```



### 问题排查

首先，就是自己把用户的sql语句拿过来，使用 `setspark.sql.autoBroadcastJoinThreshold=-1` 命令设置参数，使用`explain`命令进行查询执行计划。

发现执行计划中有一个`BroadcastNestedLoopJoin`.

然后去查看源码。发现其只有在`JoinSelection`这个将执行计划转换为物理执行计划的规则`apply`的时候才进行调用。

这个规则就是选择使用Join的方式，优先BroadcastJoin，其次ShuffleHashJoin，最次SortMergeJoin。

通过查看其apply方法.

```scala
def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
  case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
	//省略若干关于决策BroadCastJoin, ShuffleHashJoin以及SortMergeJoin的代码
  
  case j @ logical.Join(left, right, joinType, condition)
  // 此处省略若干决策最优buildSide或根据BroadcastJoin Hint以及不得不选择一个buildSide的代码
  ...  
  joins.BroadcastNestedLoopJoinExec(
    planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

   // 省略CrossJoin,即笛卡尔积
	 ...
  case _ => Nil
}
```

发现这个`BroadcastNestedLoopJoinExec`只有在不能提取出equal的join key 的left/right Join 时才会调用，而且是一定会调用。

什么是equal Join key的Join。举个例子.

```sql
select ... from a left join b on a.id=b.id;
```

那么非equal 的left/right Join， 举个例子。

```sql
select ... from a left join b;
select ... from a left join b on a.id!=b.id;
```

### 解决方案

因此，问题就是用户在使用进行join时，表a 和表b的join key是空的，所以一定会调用`BroadcastNestedLoopJoinExec`,即使我们将BroadcastJoinThreshold设为-1.

所以解决方案就是更改用户的sql语句，更改为(此处不考虑列名冲突，如冲突，请用alias).

```sql
select * 
from (
select a.*, b.*
  from 
  a left join b
  on a.a1=b.b1
) d
left join c 
on
d.b2=c.c1;
```



### 附录

在附录中提供一个Unit test 以及对应的explain.

```scala
    test("test brodacast join") {
      withSQLConf("spark.sql.crossJoin.enabled" -> "true",
        "spark.sql.autoBroadcastJoinThreshold" -> "-1") {
        withTable("ta", "tb", "tc") {
          sql("create table ta(a1 int, a2 int) using parquet")
          sql("create table tb(b1 int, b2 int) using parquet")
          sql("create table tc(c1 int, c2 int) using parquet")

          sql("select * from (" +
            "select a1 from ta where a1 <>'123')as a " +
            "left join tb as b " +
            "left join tc as c " +
            "on a.a1=b.b1 and a.a1=c.c1").explain(false)

          sql("Select * from (select * from (" +
            "select a1 from ta where a1 <>'123')as a " +
            "left join tb as b on a.a1 = b.b1) as d " +
            "left join tc as c " +
            "on d.a1=c.c1").explain(false)

          sql("Select * from (select * from (" +
            "select a1 from ta where a1 <>'123')as a " +
            "left join tb as b on a.a1 != b.b1) as d " +
            "left join tc as c " +
            "on d.a1=c.c1").explain(false)
        }
      }
    }
```

第一条select语句的执行计划如下，由于其a 与b的 joinKey为空，所以其包含`BroadcastNestedLoopJoinExec`。

```bash
== Physical Plan ==
SortMergeJoin [a1#175], [c1#179], LeftOuter, (a1#175 = b1#177)
:- *(3) Sort [a1#175 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(a1#175, 5)
:     +- BroadcastNestedLoopJoin BuildRight, LeftOuter
:        :- *(1) Project [a1#175]
:        :  +- *(1) Filter (isnotnull(a1#175) && NOT (a1#175 = 123))
:        :     +- *(1) FileScan parquet default.ta[a1#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [IsNotNull(a1), Not(EqualTo(a1,123))], ReadSchema: struct<a1:int>
:        +- BroadcastExchange IdentityBroadcastMode
:           +- *(2) FileScan parquet default.tb[b1#177,b2#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<b1:int,b2:int>
+- *(5) Sort [c1#179 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(c1#179, 5)
      +- *(4) FileScan parquet default.tc[c1#179,c2#180] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<c1:int,c2:int>

```

第二条select语句执行计划如下，由于a 和b 和 b 和c之间都有equal 的join key，所以其不会触发Broadcast.

```bash
== Physical Plan ==
SortMergeJoin [a1#175], [c1#179], LeftOuter
:- SortMergeJoin [a1#175], [b1#177], LeftOuter
:  :- *(2) Sort [a1#175 ASC NULLS FIRST], false, 0
:  :  +- Exchange hashpartitioning(a1#175, 5)
:  :     +- *(1) Project [a1#175]
:  :        +- *(1) Filter (isnotnull(a1#175) && NOT (a1#175 = 123))
:  :           +- *(1) FileScan parquet default.ta[a1#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [IsNotNull(a1), Not(EqualTo(a1,123))], ReadSchema: struct<a1:int>
:  +- *(4) Sort [b1#177 ASC NULLS FIRST], false, 0
:     +- Exchange hashpartitioning(b1#177, 5)
:        +- *(3) FileScan parquet default.tb[b1#177,b2#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<b1:int,b2:int>
+- *(6) Sort [c1#179 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(c1#179, 5)
      +- *(5) FileScan parquet default.tc[c1#179,c2#180] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<c1:int,c2:int>

```

第三条语句由于a和b之间是一个不等的join key，所以其会触发`BroadcastNestedLoopJoinExec`。

```bash
== Physical Plan ==
SortMergeJoin [a1#175], [c1#179], LeftOuter
:- *(3) Sort [a1#175 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(a1#175, 5)
:     +- BroadcastNestedLoopJoin BuildRight, LeftOuter, NOT (a1#175 = b1#177)
:        :- *(1) Project [a1#175]
:        :  +- *(1) Filter (isnotnull(a1#175) && NOT (a1#175 = 123))
:        :     +- *(1) FileScan parquet default.ta[a1#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [IsNotNull(a1), Not(EqualTo(a1,123))], ReadSchema: struct<a1:int>
:        +- BroadcastExchange IdentityBroadcastMode
:           +- *(2) FileScan parquet default.tb[b1#177,b2#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<b1:int,b2:int>
+- *(5) Sort [c1#179 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(c1#179, 5)
      +- *(4) FileScan parquet default.tc[c1#179,c2#180] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/mllib-local/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<c1:int,c2:int>

```

