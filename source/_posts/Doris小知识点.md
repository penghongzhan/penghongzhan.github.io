---
title: Doris小知识点
date: 2021-03-06 11:29:03
tags:
- doris
- partition
- bucket
categories:
- doris
---

# doris分区

## partition

range分区，根据时间范围进行分区，这种分区是物理隔离的，可以理解成单独的一张小表，删除、查询等操作非常快

## bucket

在Doris的存储引擎中，用户数据被水平划分为若干个数据分片（Tablet，也称作数据分桶）。每个Tablet包含若干数据行。各个Tablet之间的数据没有交集，并且在物理上是独立存储的。

多个Tablet在逻辑上归属于不同的分区（Partition）。一个Tablet只属于一个Partition。而一个Partition包含若干个Tablet。因为Tablet在物理上是独立存储的，所以可以视为Partition在物理上也是独立。Tablet是数据移动、复制等操作的最小物理存储单元。

若干个Partition组成一个Table。Partition可以视为是逻辑上最小的管理单元。数据的导入与删除，都可以或仅能针对一个Partition进行。

一个 Partition 的 Bucket 数量一旦指定，不可更改。所以在确定 Bucket 数量时，需要预先考虑集群扩容的情况。比如当前只有 3 台 host，每台 host 有 1 块盘。如果 Bucket 的数量只设置为 3 或更小，那么后期即使再增加机器，也不能提高并发度。

# doris查询条件

- 一个有所有三级类目权限的人，查询的时候，会添加三级类目in的查询条件，这是非常影响性能的，去掉这个条件之后，查询性能提升一倍；到底是in的影响，还是三级类目没在索引中的影响？

# doris前缀索引

- http://doris.apache.org/master/zh-CN/getting-started/data-model-rollup.html#rollup
- https://km.sankuai.com/page/28276900
