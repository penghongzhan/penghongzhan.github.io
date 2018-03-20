---
title: druid基础架构
date: 2018-03-17 12:47:22
categories:
- druid
tags:
- druid
---

---
# 重要概念
---

- dimension: 维度列
- metrics: 指标列

![druid的维度和指标.png](https://upload-images.jianshu.io/upload_images/3151600-031d423929ad737d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
# 集群角色
---

## Historical

历史节点，从deep storage中加载数据，并且创建索引

## deep storage

数据存储，例如HDFS

## broker

接收client过来的查询请求，路由到Historical节点和Realtime节点，进行数据的查询。

Broker节点负责将查询路由到历史节点和实时节点，Broker节点通过zk来知道哪些Segment存在哪个节点上。Broker也会把查询的结果进行Merge。

Broker节点会维护一个LRU缓存，缓存存着每个Segment的结果，缓存可以是一个本地的缓存或者多个节点共用的外部的缓存如 memcached。当Broker收到查询时候，它首先将查询映射成一堆Segment的查询，其中的一个子集的结果可能已经存在缓存中，他们可以直接从缓存中拉出来，那些没在缓存中的将被发送到相应节点。

## Realtime

接收些数据请求，实时获取数据并且创建索引

## Coordinator

协调节点负责Segment的管理和分发，协调节点指挥历史节点来加载或者删除Segment，以及Segment的冗余和平衡Segment。协调节点会周期性的进行扫描，每次扫描会根据集群当前的状态来决定进一步的动作。

例如，集群中某个history节点离线，那么Coordinator会默认该机器上的所有segments已经失效，会通知集群中其它的history节点去接在这些segments。实际运行中，Coordinator不会在发现history离线的时候，立即进行segment的重新加载，这样势必会集中整个集群的负担，因此，Coordinator会为集群内所有已丢弃的Segment保存一个生存时间(lifetime)，这个生存时间表示Coordinator Node在该Segment被标记为丢弃后，允许不被重新分配最长等待时间，如果该Historical Node在该时间内重新上线，则Segment会被重新置为有效，如果超过该时间则会按照加载规则重新分配到其他Historical Node上。 

## Zookeeper

每个历史节点维持一个和Zookeeper的长连接监测一组path来获取新的Segment信息。历史节点互相不进行通信，他们依靠zk来等待协调节点来协调，协调节点负责把新的Segment分发给历史节点.

当历史节点发现一个新的entry出现在path中，它首先会检查本地文件缓存看有有没Segment信息，如果没有Segment信息，历史节点会从zk上下载新的Segment的元信息。Segment的元信息包括Segment存在Deep Storage的位置和如何解压和处理Segment。一旦一个历史节点完成对一个Segment的处理，这个历史节点会在zk上的一个路径声明对这个Segment提供查询服务，此刻这个Segment就可以查询了

## mysql

集群的元数据存储

---
# 数据结构
---

## segment

Druid 把它的索引存储到一个Segment文件中，Segment文件是通过时间来分割的。

segment中索引的采用的列式存储，假设发送给druid两条记录：

timestamp | country | city | count
---|---|---|---
2018-03-15T01:00:00 | china | shandong | 1
2018-03-15T02:00:00 | america | newyork | 2
2018-03-15T02:00:00 | america | Washington | 3

## 列式存储与bitmap

现在行存储的结构是：3行2列

**注意：这里的两列是指维度有两列，因为查询的时候是根据维度来进行聚合，所以转化成列式存储的时候只需要考虑维度列即可。**

druid会将发送过来的一条条记录转化成列式的索引，对于country这一列来说，一共有2个值，记录总共有3行，转化成列存储之后，变成2行3列，这3列的列标为之前行存储的行号：

country | 1 | 2 | 3
---|---|---|---
china | 1 | 0 | 0
america | 0 | 1 | 1

加上city维度之后形成的索引如下：

country | 1 | 2 | 3
---|---|---|---
china | 1 | 0 | 0
america | 0 | 1 | 1
shandong | 1 | 0 | 0
newyork | 0 | 1 | 0
Washington | 0 | 0 | 1

china出现在第一行，所以现在的第1列对应的值是1，2列和3列对应的值是0。

实际上的是按照bitmap进行存储的：china -> [1, 0, 0]; shandong -> [1, 0, 0]

查询的时候，例如我要进行这样的查询：select * from table where country = 'china' and city = 'shandong'

索引上进行的操作是：[1, 0, 0] & [1, 0, 0] = [1, 0, 0]，所以从文件中将第一行都出来返回即可。

再例如查询：select sum(count) from table where country = 'china' and city = 'newyork'

索引上进行的操作是：[1, 0, 0] & [0, 1, 0] = [0, 0, 0]，所以返回结果为空

---
# 索引的创建
---

加载segment的过程实际上也就是在内存中创建索引的过程，索引创建的组建包括三个：Peon， Middle Manager ，Overlord。

- Peon。运行一个单独的任务。 
- Middle Manager。管理多个Peon。 
- Overlord。接收任务，协调任务分发，对任务创建锁，最后对请求者返回状态。

在远程模式中，Overlord对于任务的调度是通过zookeeper进行的，会将任务信息注册到Zookeeper的/task目录下所有在线的Middle Manager对应的目录中，由Middle Manager去感知产生的新任务。产生新的任务实际上是创建Peon的过程，创建完Peon之后，Middle Manager又会将Peon信息同步到zookeeper中的/Status目录
