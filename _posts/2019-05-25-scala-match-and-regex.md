---
layout: post
category: coding
tagline: "match and regex"
summary: 
tags: [scala,coding]
---
{% include JB/setup %}
目录

* toc
{:toc}
### Background ###
{{ page.summary }}

scala的编程非常灵活，里面有很多的语法糖，在scala中模式匹配非常常用，往往用于取代if else，使得代码看起来更加优雅，本文讲一下自己在日常编程中用到模式匹配和正则相关的使用经验总结。

### match

match是和case一起使用。其实有些时候有一些隐式的match。比如下面这两段代码的功能都是一样的，都是过滤掉字符串中的`'_'`字符,其实这段代码可以简单的使用`str.filterNot(_ != '_')`，本例子只是讲解match case。

```scala
    str.flatMap {
      case '_' => ""
      case c => s"$c"
    }
    
    str.flatMap { char =>
      char match {
        case '_' => ""
        case c => s"c"
      }
    }
```

其实这个flatMap函数的输入是一个函数，输入是String类型，输出是一个字符集合。因此，这里省去的match实际上是对输入参数的match。

对于case来说，匹配到的可以是一个具体的value, 比如`'_'`是一个字符，而`_`代表任何value。也可以是匹配到一个类型的实例。例如:

```scala
  def testMatch(obj: Any): Unit = {
    obj match {
      case i: Int => println("Int")
      case b: Boolean => println("Boolean")
      case str: String => println("String")
      case _ => println("Other type")
    }
  }
```



#### 提取器

提取器是也是一个很常见的场景，对于一个object， 如果在 match的时候进行case操作，则会调用这个对象的`unapply`方法进行提取操作，因为是提取，所以这个unapply方法也对应的是一个match操作，而且可能会有提出不出来的场景，所以返回参数一定是Option类型的，因为有时提取可能为None。

下面是一个例子，这个例子中使用了一些样例类。

我们看`testClassMatch`方法，前两个case都是匹配到一个样例类的value比较好理解，重点看第三和第四个case。

我们看到有 `case dl @ SpecialTypeThree(ExtractLiteral(l1), ExtractLiteral(l2))`的操作，而SpecialTypeThree是一个object，作为抽取器，输入参数是`AbstractType`实例，它的unapply方法用于抽取出两个String. 而此处的`dl`代表的是匹配到的值，也就是一个`DoubleLiterals`实例。

而SpecialTypeThree抽取器返回值是两个string，而后面跟着的`ExtractLiteral(l1), ExtractLiteral(l2)`代表，两个抽取器对返回的两个String再做操作，如果依然能提取出非None的值，那么就代表匹配成功。否则，如果后面的两个ExtractLiteral 有一个返回None，都会代表此次匹配失败。

```scala
  abstract class AbstractType
  case class TypeOne(int: Int) extends AbstractType
  case class TypeTwo(bool: Boolean) extends AbstractType

  case class DoubleLiterals(lit1: String, lit2: String) extends AbstractType
  case class DoubleString(str1: String, str2: String) extends AbstractType

	def testClassMatch(instance: AbstractType): Option[String] = instance match {
   	case TypeOne(i) => Some("int")
    case TypeTwo(bool) => Some("bool")
    case dl @ SpecialTypeThree(ExtractLiteral(l1), ExtractLiteral(l2)) =>
      assert(ss.isInstanceOf[DoubleLiterals])
      Some(s"$l1\t$l2")

    case ds @ SpecialTypeThree(ExtractString(s1), ExtractString(s2)) =>
      assert(ds.isInstanceOf[DoubleString])
      Some(s"$s1\t$s2")
    case _ => None
  }

  private object SpecialTypeThree {
    def unapply(instance: AbstractType): Option[(String, String)] = instance match {
      case dl: DoubleLiterals =>
        Some(dl.lit1, dl.lit2)
      case ds: DoubleString =>
        Some(ds.str1, ds.str2)
      case _ => None
    }
  }

  private object ExtractLiteral {
    val reg = """([0-9]+)""".r
    def unapply(arg: String): Option[String] = {
      arg match {
        case reg(str) =>
          println("extract literal ing!")
          Some(str)
        case _ => None
      }
    }
  }
  private object ExtractString  {
    val reg = """([a-zA-Z]+)""".r
    def unapply(arg: String): Option[String] = {
      arg match {
        case reg(str) =>
          println("extract string ing!")
          Some(str)
        case _ => None
      }
    }
  }
```



### 正则匹配

scala中的正则表达式写起来非常的简单。

例如`"""([0-9]+)""".r`就代表一个正则表达式，用于匹配一个数字。

#### "" 与 ""”""“

此处讲一下`""`和`""""""`的区别，在scala中经常看到三引号。其实三引号最大的作用就是可以在里面使用一些需要转义的字符，并且支持换行。

例如

```scala
"""\t""" // 打印出来是\t而非制表符
"""turbo"fei""" //可在里面加引号
"""turbo
fei""" //可直接换行，不用使用\n表示换行符
```

当然常用的还是下面的方式:

```scala
    """turbo
      |fei
    """.stripMargin
```

这通常是我们打印多行信息的方式，它可以优雅的打印出信息，而这里的stripMargin方法是指定每行开始的字符，默认是`|`。

#### Regex

scala中的正则类是`scala.util.matching.Regex`.

创建Regex有两种方式:

```scala
val reg1 = new Regex("\\.")

// 后面是用于注释?
val reg2 = (\d\d\d\d)-(\d\d)-(\d\d)""".r("year", "month", "date")"""
```

正则表达式中的特殊字符在此就不详细列了，请参考官方标准。例如`\d`表示数字，`.`表示任意一个字符， 所以如果需要匹配`.`需要进行转义使用`\.`。

而String里面split方式使用的参数其实是一个正则表达式的字符串形式。

例如，如果我们需要按照`.`进行分割，那么就要需要使用`\\.` 当做正则传入。如下：

```scala
str.split("\\.")
```

所以在使用split方法时不要以为这是一个普通的字符串，其实这是一个toString的正则表达式。

在scala中，常常将要匹配的部分使用小括号包住，则，就可以配合match，匹配出对应的部分。

下面是一个例子:

```scala
    val reg = """(\d+) \+ (\d+)""".r("n1", "n2")
    "12 + 23" match {
      case reg(a, b) => println(a.toInt + b.toInt) // 35
      case _ =>
    }
```