---
layout: post
category: spark
tagline: "spark-greenplum数据传输事务支持的实现"
summary: spark-greenplum是一个spark DataSource为greenplum的实现。通过使用postgresql copy命令的方式从dataframe分区向greenplum拷贝数据，相较于spark sql本身jbdc DataSource的速度提升了上百倍。本文讲解关于实现从spark sql向gp拷贝数据事务的实现。
tags: [spark, greenplum]
---
{% include JB/setup %}
### Background ###

{{ page.summary }}

相关PR为:[SPARK-GREENPLUM-4](https://github.com/yaooqinn/spark-greenplum/7)

### Spark-greenplum

Spark-greenplum的项目地址为:https://github.com/yaooqinn/spark-greenplum.

spark本身有jdbc的DataSource支持，可以进行spark sql 到greenplum的传输，但是速度慢。
查看JdbcUtils中的savePartition方法，其中的拷贝模块为:

```scala
        while (iterator.hasNext) {
          val row = iterator.next()
          var i = 0
          while (i < numFields) {
            if (row.isNullAt(i)) {
              stmt.setNull(i + 1, nullTypes(i))
            } else {
              setters(i).apply(stmt, row, i)
            }
            i = i + 1
          }
          stmt.addBatch()
          rowCount += 1
          if (rowCount % batchSize == 0) {
            stmt.executeBatch()
            rowCount = 0
          }
        }
```

这里看到，他是针对迭代器进行遍历，达到batchSize（默认为1000）之后进行一次insert操作，因此针对大批量的拷贝操作，速度较慢。

在postgresql中，有一个copy命令，可以参考文档：https://www.postgresql.org/docs/9.2/sql-copy.html.

下面的命令为将一个文件中的数据拷贝到一个表中.

```sql
COPY table_name [ ( column_name [, ...] ) ]
    FROM { 'filename' | STDIN }
    [ [ WITH ] ( option [, ...] ) ]
```

这是一个原子操作，这个copy的速度相较于jdbc DataSource中的按批插入，性能提升极大。

通过将每个dataFrame中partition的数据写入一个文件，然后使用copy from命令将这个文件中的数据拷贝到greenplum表中，针对每个分区中的copy操作分别是原子操作，但是如何针对所有分区实现事务呢？事务对于生产环境中是非常必要的。

在讲解事务实现之前，先讲下在针对文件中一些特殊字符的处理.

```
Backslash characters (\) can be used in the COPY data to quote data characters that might otherwise be taken as row or column delimiters. In particular, the following characters must be preceded by a backslash if they appear as part of a column value: backslash itself, newline, carriage return, and the current delimiter character.
```

从sparksql 写数据到文件的过程是将每个Row写到文件中的一行，而且各个column之间使用指定的delimiter间隔。因此，在写文件时需要对于一些特殊字符进行处理，比如换行符合回车符，这些肯定是需要特殊处理的，因此不处理，就会导致一个row写了多行，之后copy命令就无法正确识别，其次就是 row中如果有column的值包含和delimiter相同的字符也要进行转义，不然copy命令就无法通过delimiter识别出列的值，除此之外还有'\\'需要特殊处理，因为对delimiter的处理是在demiter前加'\\'因此，也要针对'\\'进行处理避免与delimiter的处理方式混淆。



### 事务实现

前面提到针对每个partition的copy命令都是原子操作，但是针对整体的partition如何实现原子操作呢？

从spark sql向greenplum插入数据分为以下几种情况:

- gp表存在，是overwrite操作，但是这个表是一个级联删除表，因此我们不能使用drop再create的操作，只能truncate再进行append。
- gp表存在，向表中append数据。
- gp表存在，是overwrite操作，是非级联表，因此可以对该表进行drop再create的操作。
- gp表不存在，可以直接进行create操作。



上面四种情况，可以分为两种:

1. 可以drop if exists，再导入数据
2. 必须append数据。

#### case1

针对第一种情况，实现事务很简单，方案如下:

首先创建一个临时表，然后针对每个分区，使用copy命令，将各个分区的数据拷贝到这个临时表中。最后，如果所有分区都成功拷贝。

那么在driver中进行以下两步操作:

1. drop $table if exists
2. alter table \$tempTable rename to $table

如果这两步都成功，那么则完成了事务。

如果有分区未成功拷贝，或者在以上两步中失败，则进行删除临时表的操作。并且抛出异常，提醒用户，事务未成功。

**如何判断分区成功数**

如何判断分区是否全部成功呢？我们使用 **LongAccmulator**来实现，在driver中注册一个累加器，然后每个分区成功时则累加器加一，如果最终累加器的值，等于dataFrame的分区数，那么代表全部成功，否则是部分失败。

关于LongAccmulator，想了解的可以去搜索了解，相当于一个分布式的atomicLong.

####  case2

针对第二种情况，我们添加一个transactionOn 的option。如果为true，那么我们将dataFrame进行coalesce(1)的操作，这样dataFrame就只有一个分区，针对这个分区中copy操作就是原子性的，这样就保证了事务。

 关于coalesce操作，它与reparation操作不同。

```
 def coalesce(numPartitions: Int, shuffle: Boolean = false,
               partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
```

对于coalesce操作，从m个分区编程n个分区，如果m<n是一定要进行shuffle的，如果m>n, 则如果非指定shuffle为true，则不需要进行shuffle。

因此coalesce(1)操作，不会造成shuffle压力，而且rdd操作是迭代读取，之后进行落盘(参考[rdd-basic](https://netease-bigdata.github.io/ne-spark-courseware/slides/spark_core/rdd_basics.html#1)）。只是每个partition分区的数据都发向一个节点，数据拷贝需要进行串行，然后就是可能造成磁盘压力，如果存储不够的话就很尴尬。

如果transactionOn为false，则不保障事务。
