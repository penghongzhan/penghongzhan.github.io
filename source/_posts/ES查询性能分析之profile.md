---
title: ES查询性能分析之profile
date: 2019-08-30 15:34:05
tags:
- elastic
- profile
- query
categories:
- elasticsearch
---

# 重要指标

## query

- type：查询类型
- time：查询消耗的时间

### breakdown

query的详细信息，单位是纳秒

- create_weight：创建用于查询的临时上下文
- build_scorer：构建用户评分的评分器
- next_doc：**统计得到的查询下一个文档id需要的时间，这个不太懂啥意思，有啥用，难道是为了加快查询速度的预查询？？？**
- advance：**跟next_doc的作用好像差不多，但是不是所有的查询都能使用next_doc，此时会用到advance，所以advance能干啥也不是特别清楚。。。**
- matches：两阶段查询会统计的指标，例如短语查询，首先需要确保文档中包含这两个单词，其次再确定这两个单词在文档中是不是相邻的，大部分查询来说，这个时间一般是0
- score：使用上面的评分器进行评分消耗的时间
- *_count：上面的这些指标后面添加_count，标识这些指标被执行的次数，上面的时间是统计的所有这些次数的总的执行时间

## collector

对应lucene底层的高阶执行细节，称为collector，collector的种类有以下几种：

- search_sorted：打分并且进行排序
- search_count：计算文档数量，不会获取source，当size为0的时候会显示这个
- search_terminate_after_count：找到n个匹配文档后终止搜索执行的collector。在指定了terminate_after_count查询参数时会出现这种情况
- search_min_score：只查询分数大于某个最小值的文档
- search_multi：该collector组装了其他多个collector，当在单个搜索中同时有搜索，聚合，全局aggs和post_filters时，可以看到这种情况。
- search_timeout：查询执行超时，当在查询时指定了timeout参数时可能出现
- aggregation：聚合查询的场景会出现
- global_aggregation：全量聚合，没有配合其他的查询或者过滤条件的聚合场景会出现

## aggregations

- type：聚合类型
- time：花费时间

### breakdown

下面的这些指标有一定的先后顺序，能够有助于很好的理解聚合过程中各个环节的执行时间

- initialise：创建和初始化聚合需要的时间
- collect：收集聚合需要的字段的值，应该是去查询doc_values
- build_aggregation：对字段的值进行聚合，得到分片级别的聚合结果
- reduce：各个分片的结果最终reduce的时间，这个时间现在都是0
- *_count：这面的这些环节执行的次数

# 使用方式

查询语句中添加`"profile": true`

```
{
    "from" : 0,
    "size" : 0,
    "timeout" : "120s",
    "profile": true,
    "query" : {
      "bool" : {
        "filter" : [
          {
            "term" : {
              "dt" : "20190803"
            }
          }
        ]
    }
  }
}
```
