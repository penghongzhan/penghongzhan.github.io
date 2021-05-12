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

# es集群稳定性需要关注的点

- segment太多，定时需要merge
- 索引的查询qps，是不是某个索引的查询量非常大
- 索引的删除文档数量，删除的文档，如果没有merge的情况下，删除的数据也会占用磁盘和内存
- 别名下的索引太多，是不是也会影响查询性能

# es增加&删除master节点注意事项

核心是要注意新增删除节点的时候，需要动态的修改集群的minus_master的配置，否在会有问题

- 比如说，原来有5个master，设置的最小节点数为3，现在修改成2个master，如果不修改这个配置，那么master选举会出现异常
- 修改配置的方式可以通过提交命令动态修改，修改配置文件是没有用的

```json
// PUT /_cluster/settings
{
  "transient": {
    "discovery.zen.minimum_master_nodes": 3
  }
}
```

# bool查询的filter和should

filter和should并列在bool查询中，should条件跟没有一样，should中的条件即使一个也没有命中，也算是匹配的，如果想要命中，需要给bool查询添加一个参数：

> "minimum_should_match": 1

# search after查询

对比：普通的from+size分页查询

- search after性能更好，不用每次全量的查了再排序，获取某一部分数据
- 查询局限性：不支持随机的页数查询，只支持一页接一页的查询，因为需要提供上一次查询的最后一个值

参考文档：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-search-after.html