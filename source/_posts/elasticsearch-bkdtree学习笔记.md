---
title: elasticsearch bkdtree学习笔记
date: 2021-04-12 15:08:54
tags:
- elastic
- elasticsearch
- bkdtree
- 原理
categories:
- elasticsearch
---

# 为什么需要bdk tree

普通的字符串类型的term，倒排索引采用的数据结构是FST前缀索引和后缀词块，针对某个term的查询能够很快的定位到对应的文档列表；但是该数据结构不适合于数字类型的字段的range查询，早期的es对于这种查询，也是转化成范围内的所有term的查询，比如说range[1，50]，它实际上是转换成了bool中的or，就像（1 or 2 or 3 or 4 … or 50）。；对于数字类型的字段，范围内的term会非常多，因此查询性能较差；

bdk tree既然被称为树，对于范围查询一定是友好的，因为树的一大特性就是节点之间是有顺序的，这也是采用bdk tree的最根本的原因。一个简单的bkd tree如下图所示：

![bkd tree简单示意图](bkd_tree简单示意图.png)

比如说我要查询范围在1-50的文档，可以通过树二分查找，快速定位到所有的文档列表。

上图展示的是一个简单的一维的bkdtree，在es中或者说在lucene中，针对地理位置的经纬度等信息会存储成二维的bkdtree，下图展示的是针对二维坐标系(x,y)的一个bkd tree：

![二维bdk tree示意图](二维bdk_tree示意图.png)

# bdk tree的优势和劣势是啥

bdk tree的优势上面已经提到的，能够高效的进行范围查询，尤其对于数字类型的字段，这是tree的天然优势。

但是对于es来说，不单单是对某个字段进行查询，更多的是类似于mysql的and查询，需要进行多个条件之间的交集或者并集查询，这个场景下，bkd tree的表现就有些欠佳了，具体为啥，可以从普通的前缀索引讲起。

es的前缀索引+后缀词块的方式，可以很快定位到term对应的文档列表，而且，这个文档列表是有序的，也就是按照文档id从小到达排序的，因为有序，所以lucene底层采用了跳表的形式进行了文档id的组织，跳表的好处就在于多个文档列表的合并；比如说一个and查询对应的两个term的查询条件，第一个term查询出5个文档，第二个term查询出500个文档，我可以遍历5个文档去500个文档中匹配是否存在，因为文档列表是跳表，所以这5次匹配可以非常快速的完成，这也是后来es（Elasticsearch5.1.1之后）取消了对于普通的term查询的缓存的原因，因为直接基于磁盘已经能够很快的完成了。

我们回头看bdk tree的示意图：

![bkd tree简单示意图](bkd_tree简单示意图.png)

可以发现，范围1-50拿到的文档列表并不是有序的，因为tree是按照term排序的，也就是1-50这个范围内的数字是有序的，但是term对应的文档列表就变成无序的了，带来的一个问题就是，多个列表进行合并的时候，无法像跳表那样二分查找高效完成。因此需要在合并之前对于bkd tree返回的文档列表进行特殊处理。

第一种处理方式（对应lucene的org.apache.lucene.search.PointRangeQuery类）：将文档列表转换成bitset，也就是将id映射到一个bit数组的对应位置上，数组查询复杂度O(1)，可以有效解决合并过程中查询慢的问题，但是如果文档列表很大（range查询很容易返回大量的文档id），会造成这个bitset的构建很慢，而且对于内存的占用也比较大，所以这种方式是有待优化的。

优化的处理方式（indexOrDocValuesQuery，Elasticsearch从5.4开始加入）：这个Query包装了上面的PointRangeQuery和SortedSetDocValuesRangeQuery，并且会根据Rang查询的数据集大小，以及要做的合并操作类型，决定用哪种Query。
如果Range的代价小，可以用来引领合并过程，就走PointRangeQuery，直接构造bitset来进行迭代。 而如果range的代价高，构造bitset太慢，就使用SortedSetDocValuesRangeQuery。 这个Query利用了DocValues这种全局docid序，并包含每个docid对应value的数据结构来做文档的匹配。 当给定一个docid的时候，一次随机磁盘访问就可以定位到该id对应的value，从而可以判断该doc是否match。这样从其他过滤条件返回的小结果集中一次调用docid来判断value是否匹配。

举个例子，比如说一个and查询的两个条件，term1对应结果集5条，term2对应结果集50000条，我不需要把term2对应的文档id转化成bitset再去跟term1的合并，直接遍历term1的文档id，从term2的docvalues中查询文档id对应的term2是多少，满足条件就返回这个文档id，不满足的话这个id就过滤掉。

### 参考文档

- [Elasticsearch干货（三）：对于数值类型索引优化](https://blog.csdn.net/xiaoyu_BD/article/details/82706190)
- [lucene查询原理](https://segmentfault.com/a/1190000014480912?utm_medium=referral&utm_source=tuicool)
- [Better Query Planning for Range Queries in Elasticsearch](https://www.elastic.co/cn/blog/better-query-planning-for-range-queries-in-elasticsearch)
