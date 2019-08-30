---
layout: post
category: spark
tagline: ""
summary: 本文讲Spark的Blacklist机制
tags: [spark, scheduler]
---
{% include JB/setup %}
目录
* toc
{:toc}
{{ page.summary }}

### 前言

Spark是一个分布式计算框架，task会分发到各个节点去执行，难免会有一些bad node 会导致task的失败，task失败的次数多了会导致stage失败，如果stage连续失败达到阈值又会导致application级别的失败。而blacklist,即黑名单机制，就是为了减少这些由于bad node从而导致应用最终失败的情况。

BlackList机制是由PR[SPARK-8425](https://issues.apache.org/jira/browse/SPARK-8425)提出，里面有设计文档以及相关sub-task.

### BlackList机制

在spark的调度中，首先是触发job，然后划分stage进行提交，而在每个stage中都是一些互相独立的task。因此针对spark的调度机制，blacklist分为多种级别。

#### BlackList级别

- (Executor, task)级别
  - 一个task可以在一个executor上的最大尝试次数，如果超出次数，该executor加入该task的blacklist
- （node, task)级别
  - 一个task可以在一个node上的最大尝试次数，如果超出这个次数，则该node加入该task的blacklist
- (executor, stage)级别
  - 在一个stage中，如果其中的tasks在一个executor里失败次数超出阈值，则加入该stage的blacklist
- (node, stage)级别
  - 在一个stage中，如果其中tasks在一个node上失败次数超出阈值，则加入该stage的blacklist
- (Executor, application)级别
  - 在一个应用中，如果其中tasks在一个executor上失败次数超出阈值，则加入application 对应的blacklist
- (Node, application)级别
  - 在一个应用中，如果其中tasks在一个node上失败次数超出阈值，则加入application对应的blacklist



需要提出的是，针对**application**的blacklist不是无限期的，有一个参数控制，`spark.blacklist.timeout`,默认是1h，超出这个时间之后会从blacklist中移出。

可以配置是否kill掉**Application BlackList**的executor，如果一个node被加入到application级别的BlackList，那么node上的所有executor都要kill掉。



#### FetchFailure & BlackList

还有一个就是在FetchFailed错误的时候进行的blacklist了。

在进行shuffle fetch的时候，如果没有开启ExternalShuffleService，那么我们是向remote 的 executor索要数据，所以这时候会将这个executor 加入blacklist。

而如果是开启了ESS，那么就是说我们向那个节点的nodemanager里面的ESS服务索要数据失败，那么就会将这个ESS所在的整个Node加入到blackList。

#### Yarn Node launch Failure

前面提到的都是在任务执行过程中失败，那么如果是在yarn 启动container的时候就失败呢。

[SPARK-16630](https://issues.apache.org/jira/browse/SPARK-16630) 提出了将这个不能成功启动container的Node加入blackList.



### BlackList 参数

下面是BlackList的参数，都与前面的介绍对应，不再写中文介绍。

| 参数名称 | 默认值| 说明|
| ------------------------------------------------------- | ----- | ------------------------------------------------------------ |
| `spark.blacklist.enabled`                               | false | If set to "true", prevent Spark from scheduling tasks on executors that have been blacklisted due to too many task failures. The blacklisting algorithm can be further controlled by the other "spark.blacklist" configuration options. |
| `spark.blacklist.timeout`                               | 1h    | (Experimental) How long a node or executor is blacklisted for the entire application, before it is unconditionally removed from the blacklist to attempt running new tasks. |
| `spark.blacklist.task.maxTaskAttemptsPerExecutor`       | 1     | (Experimental) For a given task, how many times it can be retried on one executor before the executor is blacklisted for that task. |
| `spark.blacklist.task.maxTaskAttemptsPerNode`           | 2     | (Experimental) For a given task, how many times it can be retried on one node, before the entire node is blacklisted for that task. |
| `spark.blacklist.stage.maxFailedTasksPerExecutor`       | 2     | (Experimental) How many different tasks must fail on one executor, within one stage, before the executor is blacklisted for that stage. |
| `spark.blacklist.stage.maxFailedExecutorsPerNode`       | 2     | (Experimental) How many different executors are marked as blacklisted for a given stage, before the entire node is marked as failed for the stage. |
| `spark.blacklist.application.maxFailedTasksPerExecutor` | 2     | (Experimental) How many different tasks must fail on one executor, in successful task sets, before the executor is blacklisted for the entire application. Blacklisted executors will be automatically added back to the pool of available resources after the timeout specified by`spark.blacklist.timeout`. Note that with dynamic allocation, though, the executors may get marked as idle and be reclaimed by the cluster manager. |
| `spark.blacklist.application.maxFailedExecutorsPerNode` | 2     | (Experimental) How many different executors must be blacklisted for the entire application, before the node is blacklisted for the entire application. Blacklisted nodes will be automatically added back to the pool of available resources after the timeout specified by`spark.blacklist.timeout`. Note that with dynamic allocation, though, the executors on the node may get marked as idle and be reclaimed by the cluster manager. |
| `spark.blacklist.killBlacklistedExecutors`              | false | (Experimental) If set to "true", allow Spark to automatically kill the executors when they are blacklisted on fetch failure or blacklisted for the entire application, as controlled by spark.blacklist.application.*. Note that, when an entire node is added to the blacklist, all of the executors on that node will be killed. |
| `spark.blacklist.application.fetchFailure.enabled`      | false | (Experimental) If set to "true", Spark will blacklist the executor immediately when a fetch failure happens. If external shuffle service is enabled, then the whole node will be blacklisted. |
|spark.yarn.blacklist.executor.launch.blacklisting.enabled	|false| Flag to enable blacklisting of nodes having YARN resource allocation problems. The error limit for blacklisting can be configured by spark.blacklist.application.maxFailedExecutorsPerNode.|