---
title: doris总体知识点总结
date: 2021-07-29 16:31:04
categories:
- doris
tags:
- doris
- 面试
- 原理
- 架构
---

# 知识点概要

- [Broadcast/Shuffle Join](http://doris.apache.org/master/zh-CN/getting-started/advance-usage.html#_2-%E6%95%B0%E6%8D%AE%E8%A1%A8%E7%9A%84%E6%9F%A5%E8%AF%A2)
- 聚合模型的select count(*) 效率很低

# 分桶

分桶列的选择，是在 **查询吞吐** 和 **查询并发** 之间的一种权衡：

- 如果选择多个分桶列，则数据分布更均匀。如果一个查询条件不包含所有分桶列的等值条件，那么该查询会触发所有分桶同时扫描，这样查询的吞吐会增加，单个查询的延迟随之降低。这个方式适合大吞吐低并发的查询场景。
- 如果仅选择一个或少数分桶列，则对应的点查询可以仅触发一个分桶扫描。此时，当多个点查询并发时，这些查询有较大的概率分别触发不同的分桶扫描，各个查询之间的IO影响较小（尤其当不同桶分布在不同磁盘上时），所以这种方式适合高并发的点查询场景。

单个 Tablet 的数据量理论上没有上下界，但建议在 1G - 10G 的范围内。如果单个 Tablet 数据量过小，则数据的聚合效果不佳，且元数据管理压力大。如果数据量过大，则不利于副本的迁移、补齐，且会增加 Schema Change 或者 Rollup 操作失败重试的代价（这些操作失败重试的粒度是 Tablet）。

# 各种join

[Bucket Shuffle Join](http://doris.apache.org/master/zh-CN/administrator-guide/bucket-shuffle-join.html#%E5%8E%9F%E7%90%86)

- broadcast join
- shuffle join
- Bucket Shuffle Join
- Colocate Join

举个例子，当前存在A表与B表的Join查询，它的Join方式为HashJoin，不同Join类型的开销如下：

- **Broadcast Join**: 如果根据数据分布，查询规划出A表有3个执行的HashJoinNode，那么需要将B表全量的发送到3个HashJoinNode，那么它的网络开销是`3B`，它的内存开销也是`3B`。
- **Shuffle Join**: Shuffle Join会将A，B两张表的数据根据哈希计算分散到集群的节点之中，所以它的网络开销为 `A + B`，内存开销为`B`。

## Runtime Filter

[Runtime Filter](http://doris.apache.org/master/zh-CN/administrator-guide/runtime-filter.html#%E5%90%8D%E8%AF%8D%E8%A7%A3%E9%87%8A)

针对join查询的关联条件，添加运行时过滤器，减少数据扫描和计算的数量

# doris存储文件和索引

[doris存储文件和索引](http://doris.apache.org/master/zh-CN/internal/doris_storage_optimization.html#%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)

