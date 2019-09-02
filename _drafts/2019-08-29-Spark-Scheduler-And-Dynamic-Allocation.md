---
layout: post
category: spark
tagline: ""
summary: 本文讲Spark中的调度，TaskScheduler 和 Dynamic Allocation
tags: [spark, scheduler]
---
{% include JB/setup %}
目录
* toc
{:toc}
{{ page.summary }}

### 前言

在Spark中的关于调度有一些概念，比如Job，Stage, Task.

Spark中遇到action算子会触发job的提交，然后根据宽窄依赖划分stage，然后提交stage。每个stage中都是一些相互独立的task，每个task读取一个partition的数据进行计算。

而Spark中有两个scheduler，一个High-level的DAGScheduler 和一个 Low-level的TaskScheduler。本文task的调度过程。另外dynamic allocation 是一个生产中常用的功能，是为了提高资源利用率，那么在dynamic allocation中，task又是如何调度的呢，本文将对这些内容进行讲解。

### DAGScheduler && TaskScheduler

DAGScheduler是一个High-level的调度器，是面向stage的调度器。

首先DAG是有向无环图，是将整个Job的拓扑结构画出来，然后将stages划分出来。

DAGScheduler是一个上层的接口，负责的是将stage提交，并且其拥有一个`DAGSchedulerEventProcessLoop`用来负责和executor通信，处理一些executor的执行结果。比如说task完成，executor lost这些信息，针对task的一些处理还是交由里面更加low-level的TaskScheduler进行处理。

### 关于task的启动



### Dynamic Allocation 什么时候申请资源