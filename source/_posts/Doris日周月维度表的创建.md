---
title: Doris日周月维度表的创建
date: 2021-05-17 15:11:29
categories:
- doris
tags: 
- doris
- 维度
- 建模
---

> 日、周、月维度是拆分成单独的表存储还是一张表存储

根儿上是一个写入和查询的权衡问题（存储压力其实可以忽略，因为周月维度数据量不大）：

- 多张表：写入压力大、查询压力小；
- 一张表：写入压力小、查询压力大。

只能通过具体的案例来得到一个经验值：

- 数据量超过多大之后，适合用一张表，此时压力主要在写入上
- 查询性能要求多少，适合多张表，此时压力主要在查询上

案例分析：

# 卖家多维分析多张表

## 查询性能对比

1. 通过单独的月表查询月维度

```sql
# 消耗时间0.9s
SELECT t44926_0.`mo_num` `mo_num`,
       t44926_0.`bu_name` `buname`,
       t44926_0.`seller_id` `seller_id`,
       t44926_0.`seller_name` `seller_name`,
       sum(t44926_0.arranged_weight) `arranged_weight_3p`,
       sum(t44926_0.arranged_cnt) `arranged_cnt_3p`,
       sum(t44926_0.billing_amt)/(sum(t44926_0.arranged_amt)+sum(t44926_0.coupon_amt)) `billing_rate_3p`,
       bitmap_union_count(t44926_0.quality_case_customer_str)/bitmap_union_count(t44926_0.ord_customer_str) `quality_case_cust_rate_3p`,
       sum(t44926_0.arranged_amt) `arranged_amt_3p`,
       bitmap_union_count(t44926_0.order_code) `order_cnt_3p`,
       sum(t44926_0.arranged_amt)/bitmap_union_count(t44926_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_mo t44926_0
 where t44926_0.`mo_num` = 24257
   and t44926_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 GROUP BY
  mo_num,bu_name,seller_id,seller_name
 order by `arranged_amt_3p` DESC nulls last
 limit 20;
```

2. 通过日表出月维度的数据

```sql
# 消耗时间1.8s
SELECT 
       t44926_0.`bu_name` `buname`,
       t44926_0.`seller_id` `seller_id`,
       t44926_0.`seller_name` `seller_name`,
       sum(t44926_0.arranged_weight) `arranged_weight_3p`,
       sum(t44926_0.arranged_cnt) `arranged_cnt_3p`,
       sum(t44926_0.billing_amt)/(sum(t44926_0.arranged_amt)+sum(t44926_0.coupon_amt)) `billing_rate_3p`,
       bitmap_union_count(t44926_0.quality_case_customer_str)/bitmap_union_count(t44926_0.ord_customer_str) `quality_case_cust_rate_3p`,
       sum(t44926_0.arranged_amt) `arranged_amt_3p`,
       bitmap_union_count(t44926_0.order_code) `order_cnt_3p`,
       sum(t44926_0.arranged_amt)/bitmap_union_count(t44926_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_dt t44926_0
 where t44926_0.`dt` >= 20210501 and `dt`<=20210531
   and t44926_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 GROUP BY
  bu_name,seller_id,seller_name
 order by `arranged_amt_3p` DESC nulls last
 limit 20;
```

## 写入时间对比

1. 单独的月表的导入任务的时间消耗

> 日表导入需要时间为：2.7小时，月表的导入时间：2.9小时，月表的导入时间是更长的

月表的导入数据量会更大一些，天表导入只需要60天，但是月表需要严格的2个月，配置的delta为60，但是真正对应的月份大多是3个月的情况。

2. 单独的月表的存储空间的占用

> 天表占用空间：23G，月表占用空间4.1G，额外空间的占用确实可以忽略，因为即使是一张表，也需要创建rollup也需要消耗空间

# 卖家多维分析单张表

创建月分区表，dt维度通过创建rollup的方式来

```sql
CREATE TABLE `app_invite_business_analysis_view_all` (
  `mo_num` int(11) NULL COMMENT "月份",
  `mtd_tag` int(11) NULL COMMENT "mtd:0否、1是",
  `bu_id` bigint(20) NULL COMMENT "事业部",
  `cat1_id` bigint(20) NULL COMMENT "一级类目ID",
  `cat2_id` bigint(20) NULL COMMENT "二级类目ID",
  `cat3_id` bigint(20) NULL COMMENT "三级类目ID",
  `cat4_id` bigint(20) NULL COMMENT "四级类目ID",
  `seller_id` bigint(20) NULL COMMENT "卖家ID",
  `sku_id` bigint(20) NULL COMMENT "商品ID",
  `bu_name` varchar(256) NULL COMMENT "事业部名称",
  `cat1_name` varchar(256) NULL COMMENT "一级类目名称",
  `cat2_name` varchar(256) NULL COMMENT "二级类目名称",
  `cat3_name` varchar(256) NULL COMMENT "三级类目名称",
  `cat4_name` varchar(256) NULL COMMENT "四级类目名称",
  `seller_name` varchar(256) NULL COMMENT "卖家名称",
  `sku_name` varchar(256) NULL COMMENT "商品名称",
  `brand_name` varchar(256) NULL COMMENT "品牌名称",
  `sku_unit_desc` varchar(256) NULL COMMENT "规格描述",
  `dt` int(11) NULL COMMENT "日期类型:自然日",
  `customer_id` bitmap BITMAP_UNION NULL COMMENT "成交客户ID",
  `dt_customer_id` bitmap BITMAP_UNION NULL COMMENT "日期&成交客户ID",
  `order_code` bitmap BITMAP_UNION NULL COMMENT "订单号",
  `arranged_amt` decimal(27, 6) SUM NULL COMMENT "销售金额",
  `coupon_amt` decimal(27, 6) SUM NULL COMMENT "优惠金额",
  `arranged_cnt` decimal(27, 6) SUM NULL COMMENT "销量",
  `arranged_weight` decimal(27, 6) SUM NULL COMMENT "销售重量：单位斤",
  `billing_amt` decimal(27, 6) SUM NULL COMMENT "佣金金额",
  `bu_onshelf_sku_id` bitmap BITMAP_UNION NULL COMMENT "事业部&在售商品ID",
  `bu_arranged_sku_id` bitmap BITMAP_UNION NULL COMMENT "事业部&动销商品ID",
  `dt_bu_onshelf_sku_id` bitmap BITMAP_UNION NULL COMMENT "日期&事业部&在售商品ID",
  `dt_bu_arranged_sku_id` bitmap BITMAP_UNION NULL COMMENT "日期&事业部&动销商品ID",
  `ord_customer_str` bitmap BITMAP_UNION NULL COMMENT "3p履约客户id",
  `quality_case_customer_str` bitmap BITMAP_UNION NULL COMMENT "3p商品质量客诉客户id",
  `sku_bu_id` bitmap BITMAP_UNION NULL COMMENT "商品指标bu_id"
) ENGINE = OLAP AGGREGATE KEY(`mo_num`,`mtd_tag`,`bu_id`,`cat1_id`,`cat2_id`,`cat3_id`,`cat4_id`,`seller_id`,`sku_id`,`bu_name`,`cat1_name`,`cat2_name`,`cat3_name`,`cat4_name`,`seller_name`,`sku_name`,`brand_name`,`sku_unit_desc`,`dt`
) COMMENT "卖家多维分析数据表-包括所有时间维度" PARTITION BY RANGE(`mo_num`) (
  PARTITION p202012 VALUES [("24252"), ("24253")), 
  PARTITION p202101 VALUES [("24253"), ("24254")), 
  PARTITION p202102 VALUES [("24254"), ("24255")), 
  PARTITION p202103 VALUES [("24255"), ("24256")), 
  PARTITION p202104 VALUES [("24256"), ("24257")), 
  PARTITION p202105 VALUES [("24257"), ("24258")), 
  PARTITION p202106 VALUES [("24258"), ("24259"))
) DISTRIBUTED BY HASH(`bu_id`) BUCKETS 5 
PROPERTIES ( "replication_num" = "3", "in_memory" = "false", "storage_format" = "V2" );
```

创建rollup

- rollup_cat3_dt( dt,bu_id,cat1_id,cat2_id,cat3_id,seller_id,bu_name,cat1_name,cat2_name,cat3_name,seller_name,customer_id,dt_customer_id,order_code,arranged_amt,coupon_amt,arranged_cnt,arranged_weight,billing_amt,bu_onshelf_sku_id,bu_arranged_sku_id,dt_bu_onshelf_sku_id,dt_bu_arranged_sku_id,ord_customer_str,quality_case_customer_str,sku_bu_id),
- rollup_sku_dt( dt,bu_id,cat1_id,cat2_id,cat3_id,cat4_id,seller_id,sku_id,bu_name,cat1_name,cat2_name,cat3_name,cat4_name,seller_name,sku_name,brand_name,sku_unit_desc,customer_id,dt_customer_id,order_code,arranged_amt,coupon_amt,arranged_cnt,arranged_weight,billing_amt,bu_onshelf_sku_id,bu_arranged_sku_id,dt_bu_onshelf_sku_id,dt_bu_arranged_sku_id,ord_customer_str,quality_case_customer_str,sku_bu_id),
- rollup_mo( mo_num,mtd_tag,dt,customer_id,dt_customer_id,order_code,arranged_amt,coupon_amt,arranged_cnt,arranged_weight,billing_amt,bu_onshelf_sku_id,bu_arranged_sku_id,dt_bu_onshelf_sku_id,dt_bu_arranged_sku_id,ord_customer_str,quality_case_customer_str,sku_bu_id),
- rollup_cat3_mo( mo_num,mtd_tag,bu_id,cat1_id,cat2_id,cat3_id,seller_id,bu_name,cat1_name,cat2_name,cat3_name,seller_name,dt,customer_id,dt_customer_id,order_code,arranged_amt,coupon_amt,arranged_cnt,arranged_weight,billing_amt,bu_onshelf_sku_id,bu_arranged_sku_id,dt_bu_onshelf_sku_id,dt_bu_arranged_sku_id,ord_customer_str,quality_case_customer_str,sku_bu_id),
- rollup_dt( dt,customer_id,dt_customer_id,order_code,arranged_amt,coupon_amt,arranged_cnt,arranged_weight,billing_amt,bu_onshelf_sku_id,bu_arranged_sku_id,dt_bu_onshelf_sku_id,dt_bu_arranged_sku_id,ord_customer_str,quality_case_customer_str,sku_bu_id)

## 数据导入

1. 导入任务执行时间：2.6小时，但是导入的时间是下午，不是凌晨执行的，可能有偏差，但是感觉应该导入时间不是瓶颈
2. 最终表的数据量：43G，数据范围为20210301到当天，天表和月表的数据范围为20210220到当天，天月表的总的数据量为28G；可以看出一张表的情况下（月分区，创建多个rollup的方式），数据量更大

## 数据查询

### 月维度查询对比

月维度的查询时间上没有差别，因为新表的分桶更合理，所以新表更快些

```sql
SELECT t44926_0.`mo_num` `mo_num`,
       t44926_0.`bu_name` `buname`,
       t44926_0.`seller_id` `seller_id`,
       t44926_0.`seller_name` `seller_name`,
       sum(t44926_0.arranged_weight) `arranged_weight_3p`,
       sum(t44926_0.arranged_cnt) `arranged_cnt_3p`,
       sum(t44926_0.billing_amt)/(sum(t44926_0.arranged_amt)+sum(t44926_0.coupon_amt)) `billing_rate_3p`,
       bitmap_union_count(t44926_0.quality_case_customer_str)/bitmap_union_count(t44926_0.ord_customer_str) `quality_case_cust_rate_3p`,
       sum(t44926_0.arranged_amt) `arranged_amt_3p`,
       bitmap_union_count(t44926_0.order_code) `order_cnt_3p`,
       sum(t44926_0.arranged_amt)/bitmap_union_count(t44926_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_mo t44926_0
 where t44926_0.`mo_num` = 24256
   and t44926_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 GROUP BY
  mo_num,bu_name,seller_id,seller_name
 order by `arranged_amt_3p` DESC nulls last
 limit 20;
```

### 日维度查询对比

卖家表现查询：

- 天分区的表查询时间：0.5s（多次查询基本稳定这个时间）
- 月分区的一张表查询时间：0.9s、0.5s、0.6s、0.7s、0.5s，平均在0.7s左右

```sql
SELECT t44926_0.`dt` `dt`,
       t44926_0.`bu_name` `buname`,
       t44926_0.`seller_id` `seller_id`,
       t44926_0.`seller_name` `seller_name`,
       sum(t44926_0.arranged_weight) `arranged_weight_3p`,
       sum(t44926_0.arranged_cnt) `arranged_cnt_3p`,
       sum(t44926_0.billing_amt)/(sum(t44926_0.arranged_amt)+sum(t44926_0.coupon_amt)) `billing_rate_3p`,
       bitmap_union_count(t44926_0.quality_case_customer_str)/bitmap_union_count(t44926_0.ord_customer_str) `quality_case_cust_rate_3p`,
       sum(t44926_0.arranged_amt) `arranged_amt_3p`,
       bitmap_union_count(t44926_0.order_code) `order_cnt_3p`,
       sum(t44926_0.arranged_amt)/bitmap_union_count(t44926_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_dt t44926_0
 where t44926_0.`dt` between 20210501 and 20210507
   and t44926_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 GROUP BY
  dt,bu_name,seller_id,seller_name
 order by `arranged_amt_3p` DESC nulls last
 limit 20;
```

sku明细查询：

- 天分区表查询时间：3.7s、6s、4.3s、5s、3.9
- 月分区一张表查询时间：4.9s、4s、4.3s、3.9s、5s

```sql
SELECT t44923_0.`dt` `dt`,
       t44923_0.`bu_name` `buname`,
       t44923_0.`seller_id` `seller_id`,
       t44923_0.`seller_name` `seller_name`,
       t44923_0.`sku_id` `sku_id`,
       t44923_0.`sku_name` `sku_name`,
       t44923_0.`brand_name` `brand_name`,
       t44923_0.`sku_unit_desc` `sku_unit_desc`,
       t44923_0.`cat1_id` `cat1_id`,
       t44923_0.`cat1_name` `cat1_name`,
       sum(t44923_0.arranged_weight) `arranged_weight_3p`,
       sum(t44923_0.arranged_cnt) `arranged_cnt_3p`,
       sum(t44923_0.billing_amt)/(sum(t44923_0.arranged_amt)+sum(t44923_0.coupon_amt)) `billing_rate_3p`,
       bitmap_union_count(t44923_0.quality_case_customer_str)/bitmap_union_count(t44923_0.ord_customer_str) `quality_case_cust_rate_3p`,
       sum(t44923_0.arranged_amt) `arranged_amt_3p`,
       bitmap_union_count(t44923_0.order_code) `order_cnt_3p`,
       sum(t44923_0.arranged_amt)/bitmap_union_count(t44923_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_dt t44923_0
 where t44923_0.`dt` = 20210523
   and t44923_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 GROUP BY
dt,bu_name,seller_id,seller_name,sku_id,sku_name,brand_name,sku_unit_desc,cat1_id,cat1_name
 order by `arranged_amt_3p` DESC nulls last
  limit 20 offset 2000;
```

总体表现：查询性能差距不是非常大，大查询在1s左右，小查询在200ms左右

# 总结&思考

感觉多张表的代价有点大，最好还是创建一张表，以月维度为分区，然后日、周的查询通过创建rollup来加速。

## 明细表的思考

从上面的性能测试可以看出，sku明细的查询基本需要5s开外了，比较慢，而且明细也是不需要聚合的，如果直接采用明细结果查询，是不是更快？

后续是不是考虑明细和类目粒度的分开存储？

```sql
#不group by直接查询明细，只需要1.5s，如果使用es，性能应该会更好
SELECT t44923_0.`dt` `dt`,
       t44923_0.`bu_name` `buname`,
       t44923_0.`seller_id` `seller_id`,
       t44923_0.`seller_name` `seller_name`,
       t44923_0.`sku_id` `sku_id`,
       t44923_0.`sku_name` `sku_name`,
       t44923_0.`brand_name` `brand_name`,
       t44923_0.`sku_unit_desc` `sku_unit_desc`,
       t44923_0.`cat1_id` `cat1_id`,
       t44923_0.`cat1_name` `cat1_name`,
       t44923_0.arranged_weight `arranged_weight_3p`,
       t44923_0.arranged_cnt `arranged_cnt_3p`,
       t44923_0.billing_amt/(t44923_0.arranged_amt+t44923_0.coupon_amt) `billing_rate_3p`,
       bitmap_count(t44923_0.quality_case_customer_str)/bitmap_count(t44923_0.ord_customer_str) `quality_case_cust_rate_3p`,
       t44923_0.arranged_amt `arranged_amt_3p`,
       bitmap_count(t44923_0.order_code) `order_cnt_3p`,
       t44923_0.arranged_amt/bitmap_count(t44923_0.dt_customer_id) `day_cust_order_amt_3p`
  FROM kldp_data_stat.app_invite_business_analysis_view_dt t44923_0
 where t44923_0.`dt` = 20210523 
   and t44923_0.`bu_id` in(11002915,11000217,11000215,11000216,11000213,11000214,11000178,11000211,11000179,11000212,11000176,11000177,11000210,11000185,11000186,11000183,11000184,11000181,11000182,120100,11000180,11000189,11000187,11000188,11000075,11000196,11000197,11000073,11000194,11000195,11000192,11000193,11000190,11000191,11000039,11000037,11000158,11000159,11000156,11000157,11000033,11000154,11000155,11000042,11000163,11000043,11000164,11000040,11000161,11000041,11000162,11000160,340100,11000208,11000209,11000206,11000207,11000204,11000205,11000169,11000202,11000203,11000046,11000167,11000200,11000047,11000168,11000201,11000044,11000165,11000045,11000166,11000174,11000175,11000172,11000173,11000170,11000171,710400,11000019,110100,11000017,11000138,11000018,11000139,11002954,11000136,11000016,11000137,11000134,11000135,11000011,11000132,11000012,11000133,11000020,11000141,11000021,11000142,11000140,11000028,11000149,11000147,11000027,11000148,11000024,11000145,11000146,11002961,11000143,11000144,11000031,11000152,11000032,11000153,11000150,11000151,11000116,11000198,310100,11000111,11000199,11000008,11000129,11000009,11000006,11000125,11000126,11000003,11000088,11000010)
 order by `arranged_amt_3p` DESC nulls last
  limit 20 offset 0;
```

