---
title: elasticsearch的查询与评分
date: 2019-10-09 20:11:53
categories:
- elasticsearch
tags:
- elasticsearch
- 查询
- 过滤
- 评分
- filter
- term
- bool
- match
- should
---

# 关键词

- match
- term
- filter
- bool
- text
- keyword

# 什么时候会进行评分

- 分词字段&match：会评分
- 分词字段&term：会评分
- 分词字段&filter&term：不会评分
- 分词字段&filter&match：不会评分
- 不分词字段&match：会评分
- 不分词字段&term：会评分
- 不分词字段&filter&term：不会评分
- 不分词字段&filter&match：不会评分

所以是否进行评分，跟字段是否分词、查询使用match还是term没关系，只跟**查询上下文是query还是term又关系**

- <mark>query查询会评分</mark>
- <mark>filter查询不会评分</mark>
- <mark>must会评分</mark>
- <mark>should会评分</mark>
- <mark>must_not不评分</mark>
- <mark>filter&must不评分</mark>
- <mark>filter&should不评分</mark>
- <mark>filter&must_not不评分</mark>

**<u>评分会减慢查询速度，所以如果查询场景中都是精确匹配，需要在filter的上下文中进行查询，这样能加快查询速度。</u>**

# 测试结果

```
curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"match":{"name":"doe"}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 6,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.25811607,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vePsw3WGo94TO752D",
        "_score": 0.25811607,
        "_source": {
          "name": "John Doe"
        }
      }
    ]
  }
}


curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"term":{"name":"doe"}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.25811607,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vePsw3WGo94TO752D",
        "_score": 0.25811607,
        "_source": {
          "name": "John Doe"
        }
      }
    ]
  }
}

curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"bool":{"filter":{"match":{"name":"doe"}}}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vePsw3WGo94TO752D",
        "_score": 0,
        "_source": {
          "name": "John Doe"
        }
      }
    ]
  }
}

curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"match":{"kw_col":"xxx"}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.2876821,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0.2876821,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}

curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"term":{"kw_col":"xxx"}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.2876821,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0.2876821,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}

curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"bool":{"filter":{"term":{"kw_col":"xxx"}}}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 11,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}

curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"bool":{"filter":{"match":{"kw_col":"xxx"}}}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}



curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"bool":{"filter":{"bool":{"should":[{"term":{"kw_col":"xxx"}},{"term":{"kw_col":"yyy"}}]}}}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}


curl -s -H 'Content-Type: application/json' -X POST  -d '{"query":{"bool":{"should":[{"term":{"kw_col":"xxx"}},{"term":{"kw_col":"xxx"}}]}}}' 'http://localhost:9200/hong_test02/doc/_search' | jq
{
  "took": 7,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.5753642,
    "hits": [
      {
        "_index": "hong_test02",
        "_type": "doc",
        "_id": "AW2vkirj3WGo94TO752Q",
        "_score": 0.5753642,
        "_source": {
          "kw_col": "xxx"
        }
      }
    ]
  }
}
```
