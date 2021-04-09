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

# doris本地join

- 本地join的时候，两个表的查询都不能有group by
- 如果是两个事实表，如果想使用本地join，那么最好两张表的维度是一致的，这种情况下，两个表可以直接join，不需要先group by之后再join；假设有一张表是到类目维度，另一张表到事业部维度，计算销售额占比等指标，类目表可以直接关联事业部表，因为本身的计算就是需要两张表的维度不一样
- 如果是事实表和维表，例如一个事实表和时间维表，可以直接本地join

# doris的分区查询

```sql
select dim.dt,
       vendor_id,
       bitmap_union_count(temp_bu_id) 
from (select * from time_dim where nature_mo = 202103) dim inner join app_vendor_board_basesku_day sku on dim.dt = sku.dt
group by vendor_id,
    dim.dt;
#####
select dt,
       vendor_id,
       bitmap_union_count(temp_bu_id)
from app_vendor_board_basesku_day where dt in (select dt from time_dim where nature_mo = 202103)
group by vendor_id,
    dt;
#####
select dim.dt,
       tt1.vendor_id,
       tt1.bitmap_union_count(temp_bu_id)
from app_vendor_board_basesku_day as tt1 left join time_dim as dim on tt1.dt=time_dim.dt
where time_dim.nature_mo = 202103
group by vendor_id,
    dt;
```

上面的这三种方式都无法走分区查询，最难理解的就是第二种，但是这个是doris的机制，如果想要走分区查询，那么需要查询两次：

- 从维表中查询日期的list
- 事实表的dt查询in这个list