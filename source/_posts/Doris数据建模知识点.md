---
title: Doris数据建模知识点
date: 2021-04-21 19:48:02
tags:
- doris
- 建模
categories:
- doris
---

# 拒绝聚合函数中增加if函数

doris如果在聚合函数中增加其他函数，是没有办法走rollup的。

通过explain查看执行计划，会发现ScanNode中PreAggregation对应的值是OFF，OFF就表示不能走rollup

## 对维度列使用聚合函数，不会影响rollup的使用

```sql
# 表是到sku粒度的，同时创建了到三级类目的rollup
# 加不加count(distinct bu_id)，下面的sql都可以走rollup
explain select count(distinct bu_id),sum(sku_cnt) from app_vendor_board_basesku_day where dt=20210501;
```



# 不需要的列不要查询

```sql
# 下面这个曲线的查询，每次最多展示2个曲线，这里一次把所有的结果都查询出来了，查询时间在1.3s左右
select *
  from (
        select date_scale as dt,
               max(begin_date_id) as begin_date_id,
               max(end_date_id) as end_date_id,
               sum(arranged_amt) as arranged_amt,
               sum(arranged_sale_amt) as arranged_sale_amt,
               sum(arranged_cnt) as arranged_cnt,
               sum(arranged_coupon_amt) as arranged_coupon_amt,
               sum(arranged_profit) as arranged_profit,
               sum(arranged_sale_profit) as arranged_sale_profit,
               sum(stock_cost_not_presale) as stock_cost_not_presale,
               sum(base_sku_code_stock_cost_not_presale) as base_sku_code_stock_cost_not_presale,
               sum(base_sku_code_ord_outbound_cost_not_presale) as base_sku_code_ord_outbound_cost_not_presale,
               sum(d30_no_move_stock_cost) as d30_no_move_stock_cost,
               max(days_count) as days_count,
               sum(arranged_headquarters_coupon_amt) as arranged_headquarters_coupon_amt,
               sum(arranged_cost) as arranged_cost,
               sum(gy_days_count) as gy_days_count,
               sum(gy_days_count_unpresale) as gy_days_count_unpresale,
               sum(gy_outstock_days_count) as gy_outstock_days_count,
               sum(gy_offshelf_days_count) as gy_offshelf_days_count,
               sum(gy_offshelf_days_count_unpresale) as gy_offshelf_days_count_unpresale,
               sum(outstock_days_count) as outstock_days_count,
               sum(onshelf_days_count) as onshelf_days_count,
               sum(onshelf_days_count_unpresale) as onshelf_days_count_unpresale,
               sum(case_nopur_coupon) as case_nopur_coupon,
               sum(case_nopur_rtn_amt) as case_nopur_rtn_amt,
               sum(scrapped_amt) as scrapped_amt,
               sum(zonghe_kaohe_numerator_amt) as zonghe_kaohe_numerator_amt,
               sum(zonghe_numerator_amt) as zonghe_numerator_amt,
               sum(background_gross_profit) as background_gross_profit,
               sum(view_uv) as view_uv,
               sum(outstock_uv) as outstock_uv,
               sum(view_pv) as view_pv,
               sum(outstock_pv) as outstock_pv,
               sum(view_uv_unpresale) as view_uv_unpresale,
               sum(outstock_uv_unpresale) as outstock_uv_unpresale,
               sum(view_pv_unpresale) as view_pv_unpresale,
               sum(outstock_pv_unpresale) as outstock_pv_unpresale,
               sum(old_stock_cost_not_presale) as old_stock_cost_not_presale,
               sum(old_ord_outbound_cost_not_presale) as old_ord_outbound_cost_not_presale
          from app_prod_sku_purchase_board_sale_kpi_cat3
         where 1 =1
           AND date_scale BETWEEN '20210321' AND '20210420'
           AND business_type = 2
           AND bu_id IN (110100, 120100, 310100, 340100, 710400, 11000003, 11000006, 11000008, 11000009, 11000010, 11000011, 11000012, 11000016, 11000017, 11000018, 11000019, 11000020, 11000021, 11000024, 11000027, 11000028, 11000031, 11000032, 11000033, 11000037, 11000039, 11000040, 11000041, 11000042, 11000043, 11000044, 11000045, 11000046, 11000047, 11000073, 11000075, 11000088, 11000111, 11000116, 11000125, 11000126, 11000129, 11000132, 11000133, 11000134, 11000135, 11000136, 11000137, 11000138, 11000139, 11000140, 11000141, 11000142, 11000143, 11000144, 11000145, 11000146, 11000147, 11000148, 11000149, 11000150, 11000151, 11000152, 11000153, 11000154, 11000155, 11000156, 11000157, 11000158, 11000159, 11000160, 11000161, 11000162, 11000163, 11000164, 11000165, 11000166, 11000167, 11000168, 11000169, 11000170, 11000171, 11000172, 11000173, 11000174, 11000175, 11000176, 11000177, 11000178, 11000179, 11000180, 11000181, 11000182, 11000183, 11000184, 11000185, 11000186, 11000187, 11000188, 11000189, 11000190, 11000191, 11000192, 11000193, 11000194, 11000195, 11000196, 11000197, 11000198, 11000199, 11000200, 11000201, 11000202, 11000203, 11000204, 11000205, 11000206, 11000207, 11000208, 11000209, 11000210, 11000211, 11000212, 11000213, 11000214, 11000215, 11000216, 11000217, 11002915, 11002954, 11002961)
           AND time_tag = 0
         group by dt
       ) t1
  left join(
        select date_scale as dt,
               count(distinct if(is_arranged = 1, bu_sku_id, null)) as has_sale_sku_cnt,
               count(distinct if(is_onself = 1, bu_sku_id, null)) as on_shelf_sku_cnt,
               count(distinct if(is_onself = 1 and is_presale_no = 0, bu_sku_id, null)) as on_shelf_sku_cnt_unpresale,
               count(distinct if(is_outstock = 1, bu_sku_id, null)) as outstock_sku_cnt,
               count(distinct if(is_unsalable = 1, bu_sku_id, null)) as dull_sku_cnt
          from app_prod_sku_purchase_board_sale_kpi_cat3
         where 1 =1
           AND date_scale BETWEEN '20210321' AND '20210420'
           AND business_type = 2
           AND bu_id IN (110100, 120100, 310100, 340100, 710400, 11000003, 11000006, 11000008, 11000009, 11000010, 11000011, 11000012, 11000016, 11000017, 11000018, 11000019, 11000020, 11000021, 11000024, 11000027, 11000028, 11000031, 11000032, 11000033, 11000037, 11000039, 11000040, 11000041, 11000042, 11000043, 11000044, 11000045, 11000046, 11000047, 11000073, 11000075, 11000088, 11000111, 11000116, 11000125, 11000126, 11000129, 11000132, 11000133, 11000134, 11000135, 11000136, 11000137, 11000138, 11000139, 11000140, 11000141, 11000142, 11000143, 11000144, 11000145, 11000146, 11000147, 11000148, 11000149, 11000150, 11000151, 11000152, 11000153, 11000154, 11000155, 11000156, 11000157, 11000158, 11000159, 11000160, 11000161, 11000162, 11000163, 11000164, 11000165, 11000166, 11000167, 11000168, 11000169, 11000170, 11000171, 11000172, 11000173, 11000174, 11000175, 11000176, 11000177, 11000178, 11000179, 11000180, 11000181, 11000182, 11000183, 11000184, 11000185, 11000186, 11000187, 11000188, 11000189, 11000190, 11000191, 11000192, 11000193, 11000194, 11000195, 11000196, 11000197, 11000198, 11000199, 11000200, 11000201, 11000202, 11000203, 11000204, 11000205, 11000206, 11000207, 11000208, 11000209, 11000210, 11000211, 11000212, 11000213, 11000214, 11000215, 11000216, 11000217, 11002915, 11002954, 11002961)
         group by dt
       ) t2
    on t1.dt = t2.dt
  left join(
        select dt as dt,
               count(distinct customer_id) as customer_cnt,
               count(distinct dt_customer_id) as dt_customer_cnt
          from app_prod_sku_purchase_board_customer_order_cat3
         where 1 = 1
           AND dt BETWEEN '20210321' AND '20210420'
           AND business_type = 2
           AND bu_id IN (110100, 120100, 310100, 340100, 710400, 11000003, 11000006, 11000008, 11000009, 11000010, 11000011, 11000012, 11000016, 11000017, 11000018, 11000019, 11000020, 11000021, 11000024, 11000027, 11000028, 11000031, 11000032, 11000033, 11000037, 11000039, 11000040, 11000041, 11000042, 11000043, 11000044, 11000045, 11000046, 11000047, 11000073, 11000075, 11000088, 11000111, 11000116, 11000125, 11000126, 11000129, 11000132, 11000133, 11000134, 11000135, 11000136, 11000137, 11000138, 11000139, 11000140, 11000141, 11000142, 11000143, 11000144, 11000145, 11000146, 11000147, 11000148, 11000149, 11000150, 11000151, 11000152, 11000153, 11000154, 11000155, 11000156, 11000157, 11000158, 11000159, 11000160, 11000161, 11000162, 11000163, 11000164, 11000165, 11000166, 11000167, 11000168, 11000169, 11000170, 11000171, 11000172, 11000173, 11000174, 11000175, 11000176, 11000177, 11000178, 11000179, 11000180, 11000181, 11000182, 11000183, 11000184, 11000185, 11000186, 11000187, 11000188, 11000189, 11000190, 11000191, 11000192, 11000193, 11000194, 11000195, 11000196, 11000197, 11000198, 11000199, 11000200, 11000201, 11000202, 11000203, 11000204, 11000205, 11000206, 11000207, 11000208, 11000209, 11000210, 11000211, 11000212, 11000213, 11000214, 11000215, 11000216, 11000217, 11002915, 11002954, 11002961)
         group by dt
       ) t3
    on t1.dt = t3.dt
  left join(
        select dt as dt,
               count(distinct customer_id) as bu_customer_cnt,
               sum(arranged_amt) as bu_arranged_amt
          from app_prod_sku_purchase_board_customer_order_cat3
         where 1 = 1
           AND dt BETWEEN '20210321' AND '20210420'
           AND business_type = 2
           AND bu_id IN (110100, 120100, 310100, 340100, 710400, 11000003, 11000006, 11000008, 11000009, 11000010, 11000011, 11000012, 11000016, 11000017, 11000018, 11000019, 11000020, 11000021, 11000024, 11000027, 11000028, 11000031, 11000032, 11000033, 11000037, 11000039, 11000040, 11000041, 11000042, 11000043, 11000044, 11000045, 11000046, 11000047, 11000073, 11000075, 11000088, 11000111, 11000116, 11000125, 11000126, 11000129, 11000132, 11000133, 11000134, 11000135, 11000136, 11000137, 11000138, 11000139, 11000140, 11000141, 11000142, 11000143, 11000144, 11000145, 11000146, 11000147, 11000148, 11000149, 11000150, 11000151, 11000152, 11000153, 11000154, 11000155, 11000156, 11000157, 11000158, 11000159, 11000160, 11000161, 11000162, 11000163, 11000164, 11000165, 11000166, 11000167, 11000168, 11000169, 11000170, 11000171, 11000172, 11000173, 11000174, 11000175, 11000176, 11000177, 11000178, 11000179, 11000180, 11000181, 11000182, 11000183, 11000184, 11000185, 11000186, 11000187, 11000188, 11000189, 11000190, 11000191, 11000192, 11000193, 11000194, 11000195, 11000196, 11000197, 11000198, 11000199, 11000200, 11000201, 11000202, 11000203, 11000204, 11000205, 11000206, 11000207, 11000208, 11000209, 11000210, 11000211, 11000212, 11000213, 11000214, 11000215, 11000216, 11000217, 11002915, 11002954, 11002961)
         group by dt
       ) t4
    on t1.dt = t4.dt
    
# 如果子查询一个指标，性能非常快，查询时间在0.3s左右
select date_scale as dt,
       max(begin_date_id) as begin_date_id,
       max(end_date_id) as end_date_id,
       sum(arranged_amt) as arranged_amt
  from app_prod_sku_purchase_board_sale_kpi_cat3
 where 1 =1
   AND date_scale BETWEEN '20210321' AND '20210420'
   AND business_type = 2
   AND bu_id IN (110100, 120100, 310100, 340100, 710400, 11000003, 11000006, 11000008, 11000009, 11000010, 11000011, 11000012, 11000016, 11000017, 11000018, 11000019, 11000020, 11000021, 11000024, 11000027, 11000028, 11000031, 11000032, 11000033, 11000037, 11000039, 11000040, 11000041, 11000042, 11000043, 11000044, 11000045, 11000046, 11000047, 11000073, 11000075, 11000088, 11000111, 11000116, 11000125, 11000126, 11000129, 11000132, 11000133, 11000134, 11000135, 11000136, 11000137, 11000138, 11000139, 11000140, 11000141, 11000142, 11000143, 11000144, 11000145, 11000146, 11000147, 11000148, 11000149, 11000150, 11000151, 11000152, 11000153, 11000154, 11000155, 11000156, 11000157, 11000158, 11000159, 11000160, 11000161, 11000162, 11000163, 11000164, 11000165, 11000166, 11000167, 11000168, 11000169, 11000170, 11000171, 11000172, 11000173, 11000174, 11000175, 11000176, 11000177, 11000178, 11000179, 11000180, 11000181, 11000182, 11000183, 11000184, 11000185, 11000186, 11000187, 11000188, 11000189, 11000190, 11000191, 11000192, 11000193, 11000194, 11000195, 11000196, 11000197, 11000198, 11000199, 11000200, 11000201, 11000202, 11000203, 11000204, 11000205, 11000206, 11000207, 11000208, 11000209, 11000210, 11000211, 11000212, 11000213, 11000214, 11000215, 11000216, 11000217, 11002915, 11002954, 11002961)
   AND time_tag = 0
 group by dt
```

## 排序对于性能的影响也很大

```sql
# 查询某天的sku的指标，需要根据销售额排序，查询列是*，发现查询很慢，需要8s+
select *
 from app_prod_analyze_dashboard_cube_dt
WHERE date_scale in (20210425)
  and time_tag = 0
  and bu_id = -1
  and sku_id > 0
  and sku_band = -1
  and activity_type = -1
  and sku_life_status = -1
  and is_gy = -1
  and is_new = -1
  and business_type = 2
order by arranged_amt desc
limit 20
offset 0
# 去掉order by之后，查询性能很快，在0.2s左右
select *
 from app_prod_analyze_dashboard_cube_dt
WHERE date_scale in (20210425)
  and time_tag = 0
  and bu_id = -1
  and sku_id > 0
  and sku_band = -1
  and activity_type = -1
  and sku_life_status = -1
  and is_gy = -1
  and is_new = -1
  and business_type = 2
# 不去掉order by，但是查询的列不是*，是具体的列，查询时间在0.9s左右，也会快很多
  select arranged_amt
 from app_prod_analyze_dashboard_cube_dt
WHERE date_scale in (20210425)
  and time_tag = 0
  and bu_id = -1
  and sku_id > 0
  and sku_band = -1
  and activity_type = -1
  and sku_life_status = -1
  and is_gy = -1
  and is_new = -1
  and business_type = 2
order by arranged_amt desc
limit 20
offset 0
```

# 查询的时候明确字段类型

字段类型如果是int，但是sql中使用的条件是字符串类型的值，比如说`dt='20210423'`，这种是没有办法走分区查询的，explain查看执行计划，发现分区扫描还是会扫面全部分区，而不是只扫描查询条件中对应的分区

# 将维度名称字段作为维度列对查询性能的影响

目前针对维度名称，有三种处理方式：

- doris表中不存储名称字段，名称字段通过维表查询
- doris表中存储名称字段，存储的是作为指标，用max的聚合方式
- doris表中存储名称字段，作为维度，跟id字段并列

第一种跟doris没关系了，先不考虑；后面两种第一种经过了验证，作为指标，对查询性能基本上没有影响；

但是最后一种，作为维度，要想查询出名称，需要在group by后面加上名称列，这个需要验证是否会影响性能，增加分组列还是对性能有一定影响的。

# grouping set

```sql
SELECT
  id,
  name,
  GROUPING(id),
  GROUPING(name),
  GROUPING_ID(id, name),
  SUM(amount) as res
FROM
  myTest
GROUP BY
  GROUPING SETS ((id, name), (id))
  order by GROUPING(name) desc ,res desc
```

![grouping-set-排序](grouping-set-排序.png)

- GROUPING SETS对应两个，其中(id)这个grouping，对应的结果，name一定是null
- 但是(id, name)这个grouping，如果数据中name本来就有null，那么也可能是null
- 怎么区分name的null到底是本来的null还是因为聚合函数中没有它才返回的null，通过GROUPING(name)函数可以判断，这个函数的返回值是1，说明一定是(id)这个grouping生成的，通常这个grouping生成的是聚合结果，需要放在前面

相关文档：

- http://doris.apache.org/master/zh-CN/internal/grouping_sets_design.html#_1-grouping-sets-%E7%9B%B8%E5%85%B3%E8%83%8C%E6%99%AF%E7%9F%A5%E8%AF%86