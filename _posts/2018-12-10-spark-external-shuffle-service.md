---
layout: post
category: spark
tagline: ""
summary: External shuffle service(ESS)是独立运行一个外部shuffle服务，用于管理spark的shuffle数据，本文讲解为什么要使用ESS，以及需要注意的地方.此处特指yarnShuffleService.
tags: [spark,shuffle]
---
{% include JB/setup %}
目录
* toc
{:toc}
### Background ###
{{ page.summary }}

## What is external shuffle service?

首先，什么是外部shuffle服务。 

在工作之前，我没有使用过spark on yarn，都是在standalone模式下跑实验。所以之前没有注意到External shuffle service。

那首先聊一下shuffle service。 shuffle分为两部分，shuffle write和shuffle read，在write端，对每个task的数据，按照key值进行hash，得到新的partitionId，然后将这些数据写到一个partitionFile里面，在paritionFile里面的数据是partitionId有序的，外加会生成一个索引，索引每个partitionFile对应偏移量和长度。

而shuffle read 端就是从这些partitionFile里面拉取相应partitionId的数据，注意是拉取所有partitionFile的相应部分。

External shuffle Service就是管理这些shuffle write端生成的shuffle数据，ESS是和yarn一起使用的， 在yarn集群上的每一个nodemanager上面都运行一个ESS，是一个常驻进程。一个ESS管理每个nodemanager上的executor生成的shuffle数据。

```scala
  /** Registers a new Executor with all the configuration we need to find its shuffle files. */
  public void registerExecutor(
      String appId,
      String execId,
      ExecutorShuffleInfo executorInfo)
```

在注册executor时，使用appId, execId和ExecutorShuffleInfo(localDirs, shuffleManager类型).所以说ESS维护的是一个索引，这些shuffle数据会在application运行结束之后，清除这些localDirs来删除。

针对每个App， 都会有一个LoadingCache来保存Shuffle 的IndexFile，默认是100m, 由`spark.shuffle.service.index.cache.size`控制。因此这个参数不能设置太大， 如果太大，在nodemanager上有多个应用运行，势必造成ESS的压力。

## Why need external shuffle service?

Spark系统在运行含shuffle过程的应用时，Executor进程除了运行task，还要负责写shuffle 数据，给其他Executor提供shuffle数据。当Executor进程任务过重，导致GC而不能为其他Executor提供shuffle数据时，会影响任务运行。同时，ESS的存在也使得，即使executor挂掉或者回收，都不影响其shuffle数据，因此只有在ESS开启情况下才能开启动态调整executor数目。

因此，spark提供了external shuffle service这个接口，常见的就是spark on yarn中的，YarnShuffleService。这样，在yarn的nodemanager中会常驻一个externalShuffleService服务进程来为所有的executor服务，默认为7337端口。

其实在spark中shuffleClient有两种，一种是blockTransferService，另一种是externalShuffleClient。如果在ESS开启，那么externalShuffleClient用来fetch  shuffle数据，而blockTransferService用于获取broadCast等其他BlockManager保存的数据。

如果ESS没有开启，那么spark就只能使用自己的blockTransferService来拉取所有数据，包括shuffle数据以及broadcast数据。

## How it works？

与外部shuffle service对应的参数有以下几个。

| `spark.shuffle.service.enabled`          | false | Enables the external shuffle service. This service preserves the shuffle files written by executors so the executors can be safely removed. This must be enabled if `spark.dynamicAllocation.enabled` is "true". The external shuffle service must be set up in order to enable it. See[dynamic allocation configuration and setup documentation](http://spark.apache.org/docs/latest/job-scheduling.html#configuration-and-setup) for more information. |
| ---------------------------------------- | ----- | ------------------------------------------------------------ |
| `spark.shuffle.service.port`             | 7337  | Port on which the external shuffle service will run.         |
| `spark.shuffle.registration.timeout`     | 5000  | Timeout in milliseconds for registration to the external shuffle service. |
| `spark.shuffle.registration.maxAttempts` | 3     | When we fail to register to the external shuffle service, we will retry for maxAttempts times. |

第一个参数是打开外部服务，这里看到描述里面写当打开动态分配时，必须设置为true，是为了让外部shuffle service管理shuffle output files，方便释放闲置的executor。

第二个参数是设置shuffle 服务的端口。

后面两个参数，就是注册超时时长与重试次数，在 shuffle需要传输大量数据时，shuffle service比较繁忙，回复这些注册信息的时延较高，因此可能会发生注册失败错误，此时要将这两个参数调大。

在spark on yarn中，会设置以下参数。

```shell
<property>

<name>yarn.nodemanager.aux-services</name>

<value>spark_shuffle</value>

</property>

<property>

<name>yarn.nodemanager.aux-services.spark_shuffle.class</name>

<value>org.apache.spark.network.yarn.YarnShuffleService</value>

</property>

<property>

<name>spark.shuffle.service.port</name>

<value>7337</value>

</property>

```



## Reference 

[Spark Configuration](http://spark.apache.org/docs/latest/configuration.html)

[External Shuffle Service](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-ExternalShuffleService.html)

