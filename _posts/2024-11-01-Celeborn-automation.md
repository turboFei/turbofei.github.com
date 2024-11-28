---
layout: post
category: celeborn
tagline: ""
summary: 基于最新的Celeborn RESTful 进行automation
tags: [celeborn]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 前言

Apache Celeborn 是一个统一的大数据中间服务，致力于提高不同MapReduce引擎的效率和弹性，并为中间数据（包括shuffle数据、溢出数据、结果数据等）提供弹性、高效的管理服务。目前，Celeborn专注于shuffle数据。

为了Spark on Kubernetes的弹性以及解决External Shuffle Service的灵活性和稳定性不足，eBay引入 Celeborn 作为Remote Shuffle Service。

Celeborn 集群本身分为两个组件，Celeborn Master 和 Celeborn Worker, Worker负责处理数据的读写，并通过心跳将各种信息汇报给 Master, 然后Master通过raft 协议保证集群数据的一致性。

对于Celeborn Master，我们进行Cloud Native部署， Celeborn Worker与现有的计算节点的NodeManager进行混布，使用systemctl管理Worker服务进程, 目前最大的集群大概有接近6000台 Celeborn Worker。

为了更好地管理Celeborn集群，我们对 Celeborn 的[RESTful API](https://celeborn.apache.org/docs/latest/restapi/) 进行了优化，以便更好地与自动化工具进行集成，并将于0.6.0版本发布。

本文将介绍我们如何基于最新的RESTful API集成自动化工具来管理Celeborn集群，其他方面不再详细说明。











