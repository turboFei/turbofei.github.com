---
layout: post
category: spark
tagline: ""
summary: limit 0 与 where i=0有什么区别
tags: [sql]

---

{% include JB/setup %}
目录

[TOC]

### Background

{{ page.summary }}

执行计划:

```sql
select * from tpcds_1t_ext.et_store_sales order by ss_sold_date_sk limit 0;

plan
== Physical Plan == 
TakeOrderedAndProject(limit=0, orderBy=[ss_sold_date_sk#806305L ASC NULLS FIRST], output=[ss_sold_date_sk#806305L,ss_sold_time_sk#806306L, ...])

+- HiveTableScan [ss_sold_date_sk#806305L, ss_sold_time_sk#806306L, ...], 
HiveTableRelation
`tpcds_1t_ext`.`et_store_sales`,org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [ss_sold_date_sk#806305L, ss_sold_time_sk#806306L, ...]


select * from tpcds_1t_ext.et_store_sales where 1=0 order by ss_sold_date_sk;
plan
== Physical Plan == 
LocalTableScan <empty>, 
[ss_sold_date_sk#806332L, ss_sold_time_sk#806333L, ...]


```

