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

我们将一行数据的前 36 个字节 作为这行数据的前缀索引。当遇到 VARCHAR 类型时，前缀索引会直接截断

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

# doris前缀索引数序

一个unique模型的表，begin_date作为分区字段，date_scale是真正的时间维度的时间字段

**第一种前缀索引的顺序：**

- 因为date_scale是各种时间类型的时间值，必须是字符串类型，但是字符串类型的字段会截断前缀索引，后续的字段不会构建在前缀索引中了，所以想着把其他必查的字段放到前面

```sql
CREATE TABLE kldp_data_stat.app_management_dashboard_product_with3p_merge_test (
  `business_type` INT NULL COMMENT "业务类型 ：-1 全部 ， 1 KA ， 2 中小B",
  `time_tag` INT NULL COMMENT "时间标记，0:日1:自然周2:自然月3:周WTD4:月MTD 5:近7D 6:每天月MTD",
  `cat1_id` BIGINT NULL COMMENT "后台一级类目 -1 全部类目",
  `cat2_id` BIGINT NULL COMMENT "后台二级类目 -1 全部类目",
  `cat3_id` BIGINT NULL COMMENT "后台三级类目 -1 全部类目",
  `cat4_id` BIGINT NULL COMMENT "后台四级类目 -1 全部类目",
  `bu_id` BIGINT NULL COMMENT "买家事业部ID",
  `org_tag` INT NULL COMMENT "维度标记，0:全国1:管理城市",
  `date_scale` VARCHAR(64) NULL COMMENT "日周月W-1W-2M-1M-2MTD-20191223",
  `begin_date` INT NULL COMMENT "时间周期的起始日期",
  `bu_name` VARCHAR(255) NULL COMMENT "买家事业部名称",
  `catering_arranged_amt` DECIMAL(27,6) NULL COMMENT "餐饮销售额"
) ENGINE = OLAP
UNIQUE KEY(`business_type`, `time_tag`, `cat1_id`, `cat2_id`, `cat3_id`, `cat4_id`, `bu_id`, `org_tag`, `date_scale`, `begin_date`)
COMMENT "测试表-勿用！！！"
PARTITION BY RANGE(`begin_date`)
(
  PARTITION p20210423 VALUES  [("20210422"),("20210423"))
)
DISTRIBUTED BY HASH(`date_scale`) BUCKETS 3
PROPERTIES (
"storage_format" = "v2")
```

**第二种前缀索引顺序：**

- date_scale放在前面其实也有好处，因为我们的查询条件中并没有添加分区字段，在扫描所有分区的时候，不在因为date_scale在最前，所以没有匹配到的日期的分区可以很快过滤掉
- 但是每个分区中都会有其他字段的各种信息，如果其他字段放在最前面，需要扫很多列才能扫到date_scale列，猜测这部分时间是比较长的

```sql
CREATE TABLE kldp_data_stat.app_management_dashboard_product_with3p_merge (
  `date_scale` VARCHAR(64) NULL COMMENT "日周月W-1W-2M-1M-2MTD-20191223",
  `cat1_id` BIGINT NULL COMMENT "后台一级类目 -1 全部类目",
  `cat2_id` BIGINT NULL COMMENT "后台二级类目 -1 全部类目",
  `cat3_id` BIGINT NULL COMMENT "后台三级类目 -1 全部类目",
  `bu_id` BIGINT NULL COMMENT "事业部iD",
  `time_tag` INT NULL COMMENT "时间标记，0:日1:自然周2:自然月3:周WTD4:月MTD 5:近7D 6:每天月MTD",
  `cat4_id` BIGINT NULL COMMENT "后台四级类目 -1 全部类目",
  `business_type` INT NULL COMMENT "业务类型 ：-1 全部 ， 1 KA ， 2 中小B",
  `org_tag` INT NULL COMMENT "维度标记，0:全国1:管理城市",
  `begin_date` INT NULL COMMENT "时间周期的起始日期",
  `bu_name` VARCHAR(255) NULL COMMENT "管理城市名称",
  `catering_arranged_amt` DECIMAL(27,6) NULL COMMENT "餐饮销售额"
) ENGINE = OLAP
UNIQUE KEY(`date_scale`, `cat1_id`, `cat2_id`, `cat3_id`, `bu_id`, `time_tag`, `cat4_id`, `business_type`, `org_tag`, `begin_date`)
COMMENT "经营大盘-商品主题"
PARTITION BY RANGE(`begin_date`)
(
  PARTITION p20210423 VALUES  [("20210422"),("20210423"))
)
DISTRIBUTED BY HASH(`date_scale`) BUCKETS 3
PROPERTIES (
"storage_format" = "v2")
```

**测试性能的sql：**

```sql
select *
  from app_management_dashboard_product_with3p_merge
 where 1 = 1
   and time_tag=0
   and cat1_id = -1
   and cat2_id = -1
   and cat3_id = -1
   and cat4_id = -1
   and date_scale in ('20210421', '20210420', '20210419', '20210418', '20210417', '20210416', '20210415', '20210414', '20210414')
   and bu_id =-1
   and org_tag != 2
   and business_type = -1
 order by date_scale asc;
```

发现两个表的查询时间基本一样，都是200ms左右，可能跟这个表结构比较简单有关系，如果数据量比较大，可能会有差异的比较明显，但是从这个表的测试看，两种情况基本相同

**效果最明显的还是添加分区字段到查询条件中，表中的begin_date：**

```
select *
  from app_management_dashboard_product_with3p_merge
 where 1 = 1
   and time_tag=0
   and cat1_id = -1
   and cat2_id = -1
   and cat3_id = -1
   and cat4_id = -1
   and date_scale in ('20210421', '20210420', '20210419', '20210418', '20210417', '20210416', '20210415', '20210414', '20210414')
   and begin_date in (20210421, 20210420, 20210419, 20210418, 20210417, 20210416, 20210415, 20210414, 20210414)
   and bu_id =-1
   and org_tag != 2
   and business_type = -1
 order by date_scale asc;
```

加上分区条件，查询时间到了几十ms，效果十分明显~