---
title: elasticsearch之按照维度聚合&按照维度排序
date: 2019-12-02 11:47:53
tags:
- elasticsearch
- es
- elastic
- 聚合
- 排序
categories:
- elasticsearch
---

es按照某个指标聚合之后，再进行排序取top，返回的结果可能是不准的。

比如说`select user_id,sum(amount) as amount from index where date between 20191101 and 20191201 group by user_id order by amount limit 10`，对于es来说，这个查询是按照`user_id`分桶，分桶之后对`amount`进行metric聚合。es为了加快查询速度并且较少内存占用，在每个分片内部进行这个查询的时候，都会进行上面排序去前十个的查询，每个分片获取前十个结果，汇总到协调节点之后，对每个分片返回的结果进行汇总排序，取前十个。就是因为在每个分片内部都是取前十个，造成了结果的不准确，因为某个`user_id`在分片1中`amount`很大，作为top10返回了，但是在分片2中可能`amount`很小，因此分片2中的这个`user_id`的结果最终没有返回到协调节点，那么这个`user_id`最终返回给用户的时候，`amount`的值是丢失了分片2中的部分结果的。

<mark>上面说的是按照聚合指标进行排序，那么问题来了，如果按照分桶的字段进行排序，返回的top结果是准确的吗？？？</mark>

类似于这样的sql：`select user_id,sum(amount) as amount from index where date between 20191101 and 20191201 group by user_id order by user_id limit 10`

在每个分片内部，查询原理跟上面提到的相同，按照`user_id`聚合之后，按照`user_id`排序，取top10，由于是按照分桶字段排序，top10中没有这个字段的结果，那就是该分片中确实不存在这个字段的记录，或者这个字段在这个分片中不是top10的，不像上面提到的按照聚合metric排序，虽然最终某个`user_id`属于top10，但是在某个分片中他对应的`amount`非常小，这部分值就被丢失了。

- 该分片中确实不存在这个字段的记录：由于分片中不存在某个`user_id`，没有返回给协调节点正常，不会影响最终结果
- 这个字段在这个分片中不是top10：某个最终返回给用户的`user_id`，在某个分片中存在，但是不属于top10，那么这个`user_id`在该分片中的记录岂不是丢失了？？？
  - **仔细分析下，上面这种情况是不存在的，如果某个`user_id`在某个分片中不是top10，那么汇总到协调节点之后，这个`user_id`一定不是top10，所以肯定不会作为最终结果返回给用户的，所以上面这种情况不存在的**

<mark>综上，这种按照分桶字段进行排序top的查询结果不会存在问题，是准确的。</mark>
