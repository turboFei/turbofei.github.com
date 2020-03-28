---
layout: post
category: spark
tagline: ""
summary: 记录Spark中的timeout参数
tags: [spark]
---
{% include JB/setup %}
目录
* toc
{:toc}
{{ page.summary }}

### 前言

Spark中许多timeout参数，本文对core和sql模块的相关参数进行梳理，基于spark-2.3.

### 已废弃参数

spark从1.4开始用rpc取代了akka，所以akka相关的参数被rpc参数取代。详情请看[RPC]部分.

- Spark.akka.askTimeout -> spark.rpc.spark.rpc.askTimeout
- spark.akka.lookupTimeout -> spark.rpc.lookupTimeout

### Misc

spark.starvation.timeout

- spark.blacklist.timeout

  > 默认 1h
  >
  > 一个节点或者executor对整个应用而言被blacklist的时间

- spark.files.fetchTimeout

  > 默认60s
  >
  > 通过调用SparkContex.addFile方法添加文件时的timeout.

- spark.launcher.childConectionTimeout

  > 默认10s
  >
  > 当调用SparkLauncher start()方法时，等待子线程和launcher server通信的timeout.

- spark.starvation.timeout

  > 默认15s
  >
  > 用于在应用初始阶段，以其为时间间隔检查是否task已经启动， 如果没有启动则表示task处于饥饿状态，打出warning通知用户。如果已经task启动， 则退出检查。



### Dynamic Allocation

- spark.dynamicAllocation.cachedExecutorIdleTimeout

  > 

  spark.dynamicAllocation.executorIdleTimeout
  spark.dynamicAllocation.schedulerBacklogTimeout
  spark.dynamicAllocation.shuffleTimeout
  spark.dynamicAllocation.sustainedSchedulerBacklogTimeout



### Network

spark.network.auth.rpcTimeout
spark.network.timeout

### RPC 

spark.rpc.RpcTimeout
spark.rpc.askTimeout
spark.rpc.long.timeout
spark.rpc.lookupTimeout
spark.rpc.short.timeout

### Shuffle

spark.shuffle.io.connectionTimeout
spark.shuffle.registration.timeout
spark.shuffle.sasl.timeout

### SQL

spark.sql.broadcastTimeout
spark.sql.catalyst.plans.logical.EventTimeTimeout
spark.sql.catalyst.plans.logical.NoTimeout
spark.sql.catalyst.plans.logical.ProcessingTimeTimeout
spark.sql.streaming.GroupStateTimeout
spark.sql.streaming.stopTimeout

### Storage

spark.storage.blockManagerSlaveTimeoutMs
spark.storage.blockManagerTimeout

### Task

spark.task.killTimeout
spark.task.reaper.killTimeout

### Worker

spark.worker.driverTerminateTimeout
spark.worker.timeout



