---
layout: post
category: spark
tagline: ""
summary: 
tags: [spark,sql]
---
{% include JB/setup %}
目录
* toc
{:toc}
{{ page.summary }}

### 前言

eBay的Hadoop集群上面每天运行着大量的Spark计算任务，对于数据计算任务，数据质量非常重要，特别是对于金融数据，数据发生corruption将会产生很严重的后果。本文分享一次数据质量相关的issue以及我们排查的过程和解决方案。

#### 场景：

一天，金融分析团队的同事报告了一个issue，他们发现在Zeta(数据开发IDE，此case中也是使用spark引擎)和spark-2.3上运行同样的SQL，有一列数据结果不一致，发生了部分数据丢失。

此处对数据进行脱敏:

| zeta      | spark-2.3 |
| --------- | --------- |
| 0.4493    | 0.449286  |
| 157.3459  | NULL      |
| -0.2091   | -0.209138 |
| 139.1228  | NULL      |
| -0.485562 | -0.485562 |

可以看出来这列的数据，在zeta中查询是有的，但是在部署的spark-2.3的CLI中去查询，出现了部分缺失。

#### 排查

其实上述两个查询中用的spark大版本是一致的，team的同事通过对比两个环境中的配置，发现有一个参数在最近进行了变更。

该参数为，`spark.sql.decimalOperations.allowPrecisionLoss`,默认为true。

在zeta环境中未设置此参数，所以为true，而在spark-2.3 client的defaults conf中，配置了该参数为false。

该参数为PR SPARK-22036 引入，是为了控制在两个Decimal类型做计算的时候，是否允许丢失精度。



在详细介绍该参数之前，介绍一下Decimal。

Decimal是数据库中的一种数据类型，不属于浮点数类型，可以在定义时划定整数部分以及小数部分的位数。对于一个Decimal类型，scale表示其小数部分的位数，precision表示整数部分位数和小数部分位数之和。一个Decimal 类型表示为Decimal(precision, scale)，在Spark中，precision和scale的上限都是38。

下面介绍`spark.sql.decimalOperations.allowPrecisionLoss`参数。

当该参数为true(默认)，表示允许丢失精度，会根据Hive行为和SQL ANSI 2011规范来决定result的类型，即如果无法精确的表示，则舍入结果的小数部分。

当该参数为false时，代表不允许丢失精度，这样会将数据表示的更加准确。eBay的ETL部门在进行数据validation的时候，对数据精度有高要求，因此我们引入了这个参数，并将其设置为false以满足ETL业务的生产需求。

设置这个参数的初衷是好的，但是为什么会引发这个data corruption问题呢?

用户的SQL数据非常的长，通过查看相关SQL的执行计划，然后进行简化，得到一个可以复现的SQL语句，如下:

```sql
set spark.sql.decimalOperations.allowPrecisionLoss=false;
select case when 1=2 then 1 else 1.123456789012345678901234 end * 1;
```

上面的select语句将会返回一个NULL。

我们将上面语句的执行计划打印出来。

```sql
"== Physical Plan ==
*(1) Project [null AS (CASE WHEN (1 = 2) THEN CAST(1 AS DECIMAL(34,24)) ELSE CAST(1.123456789012345678901234 AS DECIMAL(34,24)) END * CAST(1 AS DECIMAL(34,24)))#170]
+- Scan OneRowRelation[]"
```

执行计划很简单，里面有一个二元操作(乘法)，左边的case when 是一个Decimal(34, 24)类型，右边是一个Literal(1).

程序员都知道，在编程中，如果两个不同类型的操作数做计算，会将低级别的类型向高级别的类型进行类型转换。Spark中也是如此。

一条SQL语句进入Spark-sql引擎之后，要经历Analysis->optimization->生成可执行物理计划的过程，而这个过程就是不同的Rule作用在Plan上面不断进行转换的过程。

在Spark sql中有一系列关于类型转换的Rule。

我们来看一下Rule `ImplicitTypeCasts` 中和BinaryOperator相关的代码。

```scala
// sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/TypeCoercion.scala
case b @ BinaryOperator(left, right) if left.dataType != right.dataType => 
  findTightestCommonType(left.dataType, right.dataType).map { commonType => 
    if (b.inputType.acceptsType(commonType)) { 
      // If the expression accepts the tightest common type, cast to that. 
      val newLeft = if (left.dataType == commonType) left else Cast(left, commonType) 
      val newRight = if (right.dataType == commonType) right else Cast(right, commonType) 
      b.withNewChildren(Seq(newLeft, newRight)) 
    } else { 
      // Otherwise, don't do anything with the expression. 
      b 
    } 
 }.getOrElse(b)  // If there is no applicable conversion, leave expression unchanged. 
```

针对一个BinaryOperator(例如 + - * /), 如果左边的数据类型和右边不一致，那么会寻找一个左右操作数的common type, 然后将左右操作数都转换为common type。针对我们此处case中的 Decimal(34, 24) 和Literal(1), 它们的common type就是Decimal(34, 24),所以这里的Literal(1)将被转换为Decimal(34, 24)。

这样该二元操作的两边就都是Decimal类型。接下来这个二元操作会被Rule `DecimalPrecision`中的decimalAndDecimal方法处理。由于该二元操作是乘法操作，我们看乘法操作部分的代码。

```scala
// sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/DecimalPrecision.scala
case Multiply(e1 @ DecimalType.Expression(p1, s1), e2 @ DecimalType.Expression(p2, s2)) =>
  val resultType = if (SQLConf.get.decimalOperationsAllowPrecisionLoss) {
    DecimalType.adjustPrecisionScale(p1 + p2 + 1, s1 + s2)
  } else {
    DecimalType.bounded(p1 + p2 + 1, s1 + s2)
  }
  val widerType = widerDecimalType(p1, s1, p2, s2)
  CheckOverflow(Multiply(promotePrecision(e1, widerType), promotePrecision(e2, widerType)), resultType)
```

此处我们的操作数已经都是Decimal(34, 24)类型，所以p1=p2=34, s1=s2=24.

如果不允许精度丢失，那么resultType就是 DecimalType.bounded(p1+p2+1, s1+s2), bounded方法代表precision 和 scale都不能超过38，所以这里的ResultType就是Decimal(38, 38), 也就是小数部分为38位，那么整数部分就只剩下0位来表示，也就是说如果整数部分非0，那么这个结果就会overflow。在当前版本中，如果发生Decimal Operation 计算发生了overflow，那么就会返回一个Null的结果。

这也解释了前面的场景中，为什么使用spark-2.3客户端跑的结果，非Null的结果中整数部分都是0，而且小数部分精度更高(因为不允许精度丢失)。

好了，问题定位到这里结束，下面讲解决方案。

#### 解决方案

通过观察spark sql中Decimal 相关的Rule，发现了Rule `DecimalPrecision`中的nondecimalAndDecimal方法，这个方法是用来处理非Decimal类型和Decimal的二元操作。

此方法代码不多，如下。

```scala
// sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/DecimalPrecision.scala 
private val nondecimalAndDecimal: PartialFunction[Expression, Expression] = {
  case b @ BinaryOperator(left, right) if left.dataType != right.dataType =>
    (left, right) match {
       case (l: Literal, r) if r.dataType.isInstanceOf[DecimalType]
         && l.dataType.isInstanceOf[IntegralType] =>
         b.makeCopy(Array(Cast(l, DecimalType.fromLiteral(l)), r))
       case (l, r: Literal) if l.dataType.isInstanceOf[DecimalType]
         && r.dataType.isInstanceOf[IntegralType] =>
         b.makeCopy(Array(l, Cast(r, DecimalType.fromLiteral(r))))
       case (l @ IntegralType(), r @ DecimalType.Expression(_, _)) =>
         b.makeCopy(Array(Cast(l, DecimalType.forType(l.dataType)), r))
       case (l @ DecimalType.Expression(_, _), r @ IntegralType()) =>
         b.makeCopy(Array(l, Cast(r, DecimalType.forType(r.dataType))))
       case (l, r @ DecimalType.Expression(_, _)) if isFloat(l.dataType) =>
         b.makeCopy(Array(l, Cast(r, DoubleType)))
       case (l @ DecimalType.Expression(_, _), r) if isFloat(r.dataType) =>
         b.makeCopy(Array(Cast(l, DoubleType), r))
       case _ => b
     }
}
```

用文字描述一下此处代码的意思，此代码的目的也是为了将BinaryOperator的两个操作数转换为同一类型。

- 如果其中非Decimal类型的操作数是Literal, 那么使用DecimalType.fromLiteral方法将该Literal转换为Decimal，例如，如果是Literal(1)，则转化为Decimal(1, 0)，如果是Literal(100),则转化为Decimal(3, 0)。
- 如果其中非Decimal类型操作数是Integer类型，那么使用DecimalType.forType将Integer转换为Decimal类型，由于Integer.MAX_VALUE 为2147483647，小于3*10^9，所以将Integer转换为Decimal(10, 0).
- 如果其中非Decimal类型的操作是float/double类型，则将DecimalType转换为double类型(此为DB通用做法)。

因此，这里的DecimalPrecision Rule的nonDecimalAndDecimal方法处理一个Decimal类型和另一个非Decimal类型操作数的BinaryOperator的做法要比前面提到的ImplicitTypeCasts规则处理更加合适，不容易发生decimal operation 的overflow.

经过 nonDecimalAndDecimal处理之后的两个Decimal类型会被`DecimalPrecision`中的decimalAndDecimal方法(上文提及过)继续处理。

针对上述提到的case，是一个MuiltiPly 操作，p1=34, s1=24, p2 =1, s2=0。

其ResultType为Decimal(36,24)，也就是说可以有12位表示整数部分，不容易溢出。

Spark sql中关于类型转换的Rule是作用在Analysis阶段的Resolution子阶段。 而Resolution子阶段有一批的Rule，这批Rule会一直作用在一个Plan上，直到这个Plan到达一个Fixpoint(即不动点，继续作用Rule也不再改变Plan)。

因此，我们可以在ImplicitTypeCasts规则中对操作数类型进行判断，如果在一个BinaryOperator中有Decimal类型的操作数，则此处跳过处理，这个BinaryOperator后续会被DecimalPrecision规则中的nonDecimalAndDecimal方法和decimalAndDecimal方法继续处理，最终到达FixPoint.

我们向Spark社区提了一个PR [SPARK-29000](https://github.com/apache/spark/pull/25701), 目前已经合入master分支。

##### 用户可感知的overflow

除此之外，默认的DecimalOperation如果发生了overflow，那么其结果将返回为空，这样的错误是不容易被用户感知的, 是很危险的。

SQL ANSI 2011提出了当算术操作发生overflow时候，应该抛出一个异常。这也是大多数数据库的做法(例如SQLService, DB2). 

PR [SPARK-23179](https://github.com/apache/spark/pull/20350) 引入了一个参数`spark.sql.decimalOperations.nullOnOverflow` 用来控制在Decimal Operation 发生overflow时候的处理方式。

默认是true，代表在DecimalOperation发生overflow时返回NULL的结果。

如果设置为false，则会在发生overflow时候抛出一个异常。 

因此，我们在上面的基础上合入该PR，以保证我们的计算任务的数据质量。