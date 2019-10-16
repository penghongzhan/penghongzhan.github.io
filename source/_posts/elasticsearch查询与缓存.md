---
title: elasticsearch查询与缓存
date: 2019-10-16 20:59:38
tags:
- elasticsearch
- es
- elastic
- filter
- query
- must
- cache
- 缓存
categories:
- elasticsearch
---

在[【转载】Elasticsearch2.x Filter执行流程及缓存原理](https://penghongzhan.github.io/2019/10/16/%E3%80%90%E8%BD%AC%E8%BD%BD%E3%80%91Elasticsearch2-x-Filter%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%8F%8A%E7%BC%93%E5%AD%98%E5%8E%9F%E7%90%86/)一文中提到：filter查询上下文中对应的查询的结果会以bitmap的方式进行缓存，后续再进行查询的时候，直接通过内存中的bitmap进行文档id的过滤。

bool查询中must和filter比较像，其中都会经常用到term和range等条件，两者的区别在于filter查询不会进行评分，must查询会进行评分。

但是在实际的查询场景中，经过测试发现，must和filter的查询速度基本一致，查询场景如下：

- must
```
{
    "size": 30000,
    "query": {
        "bool": {
            "must": [
                {
                    "term": {
                        "tag": {
                            "value": 1,
                            "boost": 1
                        }
                    }
                },
                {
                    "term": {
                        "scale": {
                            "value": "20191013",
                            "boost": 1
                        }
                    }
                }
            ]
        }
    }
}
```
- filter
```
{
    "size": 30000,
    "query": {
        "bool": {
            "filter": [
                {
                    "term": {
                        "tag": {
                            "value": 1,
                            "boost": 1
                        }
                    }
                },
                {
                    "term": {
                        "scale": {
                            "value": "20191013",
                            "boost": 1
                        }
                    }
                }
            ]
        }
    }
}
```

其中tag、scale字段都是keyword类型。

## 为什么must不加载缓存

首先为什么filter可以查询结果可以缓存，但是must不行呢，关键在于是否评分。

评分不进需要考虑到term在所在文档中的频次、term在整个索引中出现的次数、还需要考虑所在文档的长度，以及其他文档的长度等信息，虽然term所在的文档是不变的，但是索引中的其他文档可能发生变化，比如说有新的文档写入，所以文档评分不是稳定的，即使当前的segment没有发生变化。

所以需要评分的结果，感觉不是非常适合做缓存，而且评分的查询结果需要返回对应文档的评分，而内存中的bitmap缓存只能保存文档的id信息，还需要重新通过其他一下信息进行评分，所以只是通过bitmap进行缓存也是不够的。而filter查询不需要评分，只需要返回文档id，所以非常适合用bitmap。

所以must查询不能加载进缓存的原因在于：需要评分！

## 为什么must需要评分，性能跟filter确差不多

首先filter下面都是term查询，elasticsearch 5.1.1开始取消了Terms filter Cache，所以实际上上面的查询场景中，filter对应的查询是没有加载内存的。

之所以取消term查询的缓存，是因为term查询直接从倒排索引中进行查询也已经很快了，取消缓存省下来的内存可以用在其他更有用的地方。

就算两者都不走缓存，但是must需要评分，也应该更慢才是，原因在于字段都是keyword类型，起始评分需要计算的东西很少，虽然多进行了评分，但是消耗的时间基本上可以忽略了。
