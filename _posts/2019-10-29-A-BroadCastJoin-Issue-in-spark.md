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

除了该threshold之外，Broadcast还有一个限制，就是广播的表的行数不能超过512 milions行，也就是5亿多行，这个值是hard code的, 因为BroadcastJoin是要基于小表构建hashMap, 行数就对应其构建hashMap的元素数量，因此必须对小表的行数有限制。也就是说即使表的磁盘物理size小于threshold，条数超过这个行数也不能进行BroadcastJoin。



### 问题描述

有一天我们遇到的一个Issue。其异常信息如下:

```bash
Caused by: org.apache.spark.SparkException: Cannot broadcast the table with more than 512 millions rows: 620880056 rows
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1$$anonfun$apply$1.apply(BroadcastExchangeExec.scala:78)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1$$anonfun$apply$1.apply(BroadcastExchangeExec.scala:73)
	at org.apache.spark.sql.execution.SQLExecution$.withExecutionId(SQLExecution.scala:97)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1.apply(BroadcastExchangeExec.scala:72)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec$$anonfun$relationFuture$1.apply(BroadcastExchangeExec.scala:72)
```

看到这个异常的第一反应就是去查询`spark.sql.autoBroadcastJoinThreshold`的值，然后查到的结果100m。当时的想法是，为什么这个表在磁盘上不到100m大小，而其有6亿行数据，难道是列数十分少，并且压缩的特别的严重，不像是生产环境中的表。

然后查询了一下表的信息，果然这个表只有一列，类型为Decimal类型，然后使用的压缩方式是snappy，而表的大小只有3.5m，想必是由于列是数值类型，所以压缩十分恐怖。

当时没有产生其他怀疑，就建议用户将`spark.sql.autoBroadcastJoinThreshold`设置为-1，禁用掉BroadcastJoin。

过了段时间，用户回复说设置了之后仍然报上面的异常。

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
  //此处省略若干决策最优buildSide或根据BroadcastJoin Hint以及不得不选择一个buildSide的代码
  ...  
  joins.BroadcastNestedLoopJoinExec(
    planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

   // 省略CrossJoin,即笛卡尔积的相关代码
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

因此，问题就是用户在使用进行 left/right join时，表a 和表b的join key是空的，所以一定会调用`BroadcastNestedLoopJoinExec`,即使我们将BroadcastJoinThreshold设为-1.

所以解决方案就是更改用户的sql语句，其实用户之前的sql在`a left join b`的时候没有添加join 条件，所以就相当于一个cross join，所以如果我们将原来语句中a b之间的left  join 改成cross join 就可以绕过BroadcastJoin，而去使用Cross join。但是，Cross join 是一个很重的join，其会产生M*R个task(M为 mapTask数量,R为reduceTask数量)。

PS: 此处看起来，如果你要进行一个大表和小表的cross join，而且小表的条数又不会超过Broadcast的条数上限，那么将cross join 替换为无join条件的left join，走Broadcast是一个不错的选择。

```sql
select a.*, b.*, c.* 
from
a cross join b left join c 
on
a.a1=b.b1 and b.b2=c.c1;
```

所以在找到解决方案之后，我还是跟用户去确认了下，到底是不是想要cross join的结果，可以拿一个小数据集进行测试，跟用户沟通了之后，才发现之前的sql产生的结果并不是他想要的，他想要的是下面的SQL。

```sql
select a.*, b.*, c.* 
from
a left join b
on 
a.a1=b.b1 
left join c 
on
and b.b2=c.c1;
```

### 总结

首先，明确需求很重要，可以先拿小数据集测试下自己想要的结果是否和测试结果一致。

Spark在进行一个 non-equal key  left/right join条件(可能join 条件为空，也可能非空但是不是key equal)，一定会有BroadcastJoin，即使是两个超大的表也会，这样可能会导致三种结果。

- 大表被Broadcast十分缓慢。
- 由于BroadcastJoin要将数据拉取到driver，可能造成driver的OOM。
- 即使不会造成OOM，大表也可能造成hard code的Broadcast 条数限制，导致无法执行。

所以，我们要在明确需求的前提下，正确的使用left/right join以及设置合适的join条件。

### 附录

在附录中提供一个Unit test 以及对应的explain.

```scala
  test("test brodacast join") {
    withSQLConf("spark.sql.crossJoin.enabled" -> "true",
      "spark.sql.autoBroadcastJoinThreshold" -> "-1") {
      withTable("ta", "tb", "tc") {
        sql("create table ta(aid int) using parquet")
        sql("create table tb(bid int, bid2 int) using parquet")
        sql("create table tc(cid int) using parquet")

        sql("select * from " +
          "ta left join tb left join tc " +
          "on ta.aid=tb.bid and tb.bid2 = tc.cid").explain(false)

        sql("select * from " +
          "ta cross join tb left join tc " +
          "on ta.aid=tb.bid and tb.bid2 = tc.cid ").explain(false)

        sql("select * from " +
          "ta left join tb " +
          "on ta.aid = tb.bid " +
          "left join tc " +
          "on tb.bid2=tc.cid").explain(false)

        sql("select * from " +
          "ta left join tb " +
          "on ta.aid != tb.bid " +
          "left join tc " +
          "on tb.bid2=tc.cid").explain(false)
      }
    }
  }
```

第一条select语句的执行计划如下，由于其a 与b的 joinKey为空，所以其包含`BroadcastNestedLoopJoinExec`。

```sql
== Physical Plan ==
SortMergeJoin [bid2#177], [cid#178], LeftOuter, (aid#175 = bid#176)
:- *(3) Sort [bid2#177 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(bid2#177, 5)
:     +- BroadcastNestedLoopJoin BuildRight, LeftOuter
:        :- *(1) FileScan parquet default.ta[aid#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<aid:int>
:        +- BroadcastExchange IdentityBroadcastMode
:           +- *(2) FileScan parquet default.tb[bid#176,bid2#177] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<bid:int,bid2:int>
+- *(5) Sort [cid#178 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(cid#178, 5)
      +- *(4) FileScan parquet default.tc[cid#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<cid:int>
```

第二条select语句执行计划如下，由于a 和b是cross join，而且b c之间有equi join key，所以其不会有`BroadcastNestedLoopJoinExec`.

```sql
== Physical Plan ==
SortMergeJoin [bid2#177], [cid#178], LeftOuter, (aid#175 = bid#176)
:- *(3) Sort [bid2#177 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(bid2#177, 5)
:     +- CartesianProduct
:        :- *(1) FileScan parquet default.ta[aid#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<aid:int>
:        +- *(2) FileScan parquet default.tb[bid#176,bid2#177] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<bid:int,bid2:int>
+- *(5) Sort [cid#178 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(cid#178, 5)
      +- *(4) FileScan parquet default.tc[cid#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<cid:int>

```

第三条语句由于a b 和 b c之间都有equi join key，所以其不会触发`BroadcastNestedLoopJoinExec`。

```sql
== Physical Plan ==
SortMergeJoin [bid2#177], [cid#178], LeftOuter
:- *(5) Sort [bid2#177 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(bid2#177, 5)
:     +- SortMergeJoin [aid#175], [bid#176], LeftOuter
:        :- *(2) Sort [aid#175 ASC NULLS FIRST], false, 0
:        :  +- Exchange hashpartitioning(aid#175, 5)
:        :     +- *(1) FileScan parquet default.ta[aid#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<aid:int>
:        +- *(4) Sort [bid#176 ASC NULLS FIRST], false, 0
:           +- Exchange hashpartitioning(bid#176, 5)
:              +- *(3) FileScan parquet default.tb[bid#176,bid2#177] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<bid:int,bid2:int>
+- *(7) Sort [cid#178 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(cid#178, 5)
      +- *(6) FileScan parquet default.tc[cid#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<cid:int>

```

第四条语句由于a b之间虽然有 join key， 但是是非 equi的join key `ta.aid != tb.bid`，所以其会触发`BroadcastNestedLoopJoinExec`。

```sql
== Physical Plan ==
SortMergeJoin [bid2#177], [cid#178], LeftOuter
:- *(3) Sort [bid2#177 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(bid2#177, 5)
:     +- BroadcastNestedLoopJoin BuildRight, LeftOuter, NOT (aid#175 = bid#176)
:        :- *(1) FileScan parquet default.ta[aid#175] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/ta], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<aid:int>
:        +- BroadcastExchange IdentityBroadcastMode
:           +- *(2) FileScan parquet default.tb[bid#176,bid2#177] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tb], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<bid:int,bid2:int>
+- *(5) Sort [cid#178 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(cid#178, 5)
      +- *(4) FileScan parquet default.tc[cid#178] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/Users/fwang12/ebay/spark-longwing/launcher/spark-warehouse/tc], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<cid:int>
```

