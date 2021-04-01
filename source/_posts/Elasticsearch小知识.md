---
title: Elasticsearch小知识
date: 2021-03-15 14:41:50
tags:
- elastic
- elasticsearch
- 小知识
categories:
- elasticsearch
---

- elastic5.6.3集群中只要有一台节点fullgc了，那么cat api会卡住，无法通过cat api获取集群的一些状态；好像任何一台机器fullgc之后，都会导致这样的情况，感觉不是非常合理。
- 限制es的查询节点，查询中添加preference，rest client中指定的机器，需要排除有问题的机器
