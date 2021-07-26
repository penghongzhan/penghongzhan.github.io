---
title: Doris VS Es之索引原理
date: 2021-07-26 13:21:07
categories:
- doris
tags:
- doris
- es
---

在olap场景下，doris的性能要强于es，很大一部分原因在于，doris可以通过聚合模型以及创建rollup，来针对olap的各种维度分析进行预聚合处理；那么，如果我们不考虑doris的agg模型以及rollup，假设就是使用duplicate模型，这时候，doris和es的聚合查询性能会孰强孰弱呢？

![doris vs es索引原理](doris-vs-es索引原理.jpg)
