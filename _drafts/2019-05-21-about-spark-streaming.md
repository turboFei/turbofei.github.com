---
layout: post
category: spark
tagline: ""
summary: 简单讲解下spark streaming, structed streaming
tags: [spark,streaming]
---
{% include JB/setup %}
目录
* toc
{:toc}
### Background ###
{{ page.summary }}

我们都知道spark中有两种streaming，一种是spark streaming，另一种是structed streaming。spark streaming是微批处理，隔一段时间提交一批job，底层走的还是rdd。

而structed streaming是spark为了满足低时延的需求，重新设计的一套流式处理机制。相关的PR是[SPARK-29028 SPIP: Continuous Processing Mode for Structured Streaming](https://issues.apache.org/jira/browse/SPARK-20928).

