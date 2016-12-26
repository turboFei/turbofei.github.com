---
layout: post
category: spark
tagline: "spark Shuffle"
summary: spark shuff部分是spark源码的重要组成部分，shuffle发生在stage的交界处，对于spark的性能有重要影响，源码更新后，spark的shuffle机制也不一样，本文分析spark2.0的shuffle实现。
tags: [spark,shuffle]
---
{% include JB/setup %}
目录

* toc
{:toc}

### Background ###
{{ page.summary }}



## Shuffle##

shuffle是Mapreduce框架中一个特定的phase，介于Map和Reduce之间。shuffle的英文意思是混洗，包含两个部分，shuffle write 和shuffle read。这里有一篇文章:[详细探究Spark的shuffle实现](http://jerryshao.me/architecture/2014/01/04/spark-shuffle-detail-investigation/)，这篇文章写于2014年，讲的是早期版本的shuffle实现。随着源码的更新，shuffle机制也做出了相应的优化，下面分析spark-2.0的shuffle机制。

`shuffleWriter`是一个抽象类，具体实现有三种，`BypassMergeSortShuffleWriter`,`sortShuffleWriter`,`UnsafeShuffleWriter`.之前看过spark1.5的代码，当时是由hashShuffleWriter的，现在spark2.0多了一个`BypassMergeSortShuffleWriter`，我们就从这个类型的shuffleWriter说起。



### BypassMergeSortShuffleWriter###

