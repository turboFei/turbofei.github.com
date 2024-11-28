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

最新的Celeborn代码对[RESTful API](https://celeborn.apache.org/docs/latest/restapi/) 进行了优化，将于0.6.0版本发布，本文介绍下我们如何基于最新的RESTful API集成自动化工具来管理Celeborn集群。

对于Celeborn Master，我们进行Cloud Native部署，部署在三个k8s集群(2, 2, 1) 共5台。 Celeborn Worker是混合部署，和原有的计算节点的NodeManager部署在一起，使用 systemctl 管理服务。









