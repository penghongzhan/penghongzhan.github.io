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

![doris索引](doris索引.jpeg)

# bitmap全局字典

- [Apache Doris 基于 Bitmap的精确去重和用户行为分析](https://blog.bcmeng.com/post/doris-bitmap.html)
- [Doris 基于 Hive 表的全局字典设计与实现](https://xie.infoq.cn/article/308cf309bf3034727c6b1df06)

| 客户id | skuid | 销售额   |
| ------ | ----- | -------- |
| 10001  | 123   | 10001123 |

客户id和skuid的组合非常多，所以排序和生成顺序号的时候单个节点压力非常大。所以需要考虑，如果对这些结果进行拆分，然后放到不同的task中进行排序和顺序号的生成。

两个维度的组合非常多，但是单个维度的值不多，所以可以先对单个维度（基于单个维度的维表）进行拆分，比如说都拆分成20份，这样组合之后是400份，可以放到400个task中执行。

遍历数据表中的所有记录，按照上述拆分的400份，将组合拆分到不同的分组中，然后到不同的task中并发的进行排序和顺序编号的生成。最终再将各个task的结果进行合并，当然还需要给顺序号添加一个偏移量

# rollup

## rollup与分区分桶

rollup是基于doris的物理存储文件来构建的，不会跨物理存储文件来聚合数据生成rollup，所以rollup是到分区+分桶级别的。不单单多个分区之间不能聚合rollup，一个分区的多个分桶下也不能聚合rollup


## 多表join时聚合多表的指标无法走rollup

```sql
# bitmap_union_count(t65418_0.customer_id)/bitmap_union_count(t1000001773_2.customer_id) `cust_cover_rate_930749`指标计算的时候，计算字段位于两张表中，导致两个表都不能走rollup
explain 
SELECT
t1000001773_2.`mo_num` `dt`,
bitmap_union_count(t65418_0.customer_id)/bitmap_union_count(t1000001773_2.customer_id) `cust_cover_rate_930749`
FROM
kldp_data_stat.app_invite_business_analysis_integrate_view_p1 t65418_0
 left
JOIN kldp_data_stat.app_invite_business_analysis_integrate_customer t1000001773_2 ON
t65418_0.bu_id = t1000001773_2.bu_id
and t65418_0.dt = t1000001773_2.dt
and t65418_0.mo_num = t1000001773_2.mo_num
and t65418_0.region_id = t1000001773_2.region_id
and t65418_0.week_num_wt = t1000001773_2.week_num_wt
where
t65418_0.`region_id` in(22306,
22307,
22302,
22303,
22304)
and t65418_0.`bu_id` in(11000019)
and t65418_0.`dt` in(20210831,
20210830,
20210901,
20210903,
20210902,
20210905,
20210904,
20210906)
GROUP BY
t1000001773_2.`mo_num`
# 只计算了bitmap_union_count(t65418_0.customer_id) 这一个字段可以走rollup
explain 
SELECT
t1000001773_2.`mo_num` `dt`,
bitmap_union_count(t65418_0.customer_id) `cust_cover_rate_930749`
FROM
kldp_data_stat.app_invite_business_analysis_integrate_view_p1 t65418_0
 left
JOIN kldp_data_stat.app_invite_business_analysis_integrate_customer t1000001773_2 ON
t65418_0.bu_id = t1000001773_2.bu_id
and t65418_0.dt = t1000001773_2.dt
and t65418_0.mo_num = t1000001773_2.mo_num
and t65418_0.region_id = t1000001773_2.region_id
and t65418_0.week_num_wt = t1000001773_2.week_num_wt
where
t65418_0.`region_id` in(22306,
22307,
22302,
22303,
22304)
and t65418_0.`bu_id` in(11000019)
and t65418_0.`dt` in(20210831,
20210830,
20210901,
20210903,
20210902,
20210905,
20210904,
20210906)
GROUP BY
t1000001773_2.`mo_num`
```

在子查询中先group by，然后对结果在join，子查询是可以走rollup的

```sql
select
  m94460_0.`dt`,
  m94460_0.`cust_cnt_954218` / m94460_1.`cust_cnt_954218` `cust_cover_rate`
from
  (
    SELECT
      t65418_0.`dt` `dt`,
      bitmap_union_count(t65418_0.customer_id) `cust_cnt_954218`
    FROM
      kldp_data_stat.app_invite_business_analysis_integrate_view_p1 t65418_0
    GROUP BY
      t65418_0.`dt`
  ) m94460_0
  left join (
    SELECT
      t65418_0.`dt` `dt`,
      bitmap_union_count(t65418_0.customer_id) `cust_cnt_954218`
    FROM
      kldp_data_stat.app_invite_business_analysis_integrate_view_p1 t65418_0
    GROUP BY
      t65418_0.`dt`
  ) m94460_1 on m94460_0.`dt` = m94460_1.`dt`
```

## 函数对rollup的影响

聚合函数中增加if函数，会导致无法走rollup

```sql
explain SELECT 
    t116147_0.`cat1_id` AS `cat1_id`,
    sum(IF(t116147_0.`dt` = 20211101, t116147_0.`arranged_origin_amt`, NULL)) AS `arranged_origin_amt`
FROM
    kldp_data_stat.app_prod_sku_detail_data t116147_0
GROUP BY t116147_0.`cat1_id`
```

聚合函数中增加case when函数，可以走rollup

```sql
explain SELECT 
    t116147_0.`cat1_id` AS `cat1_id`,
    sum(CASE WHEN t116147_0.`dt` = 20211101 THEN t116147_0.`arranged_origin_amt` END) AS `arranged_origin_amt`
FROM
    kldp_data_stat.app_prod_sku_detail_data t116147_0
GROUP BY t116147_0.`cat1_id`
```

# cube与union all

cube（包括：grouping set），在doris中查询性能非常差，不管是粗粒度的聚合还是细粒度聚合到粗粒度的聚合，性能都很差。

实际查询中，需要将各种维度组合的sql生成好，然后再union all到一起进行查询，union all的时候注意多个sql最外层的查询字段的顺序的一致性。
