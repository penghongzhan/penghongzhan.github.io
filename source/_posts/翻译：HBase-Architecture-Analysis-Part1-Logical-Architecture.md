---
title: 翻译：HBase-Architecture-Analysis-Part1-Logical-Architecture
date: 2018-04-19 22:34:06
tags:
- hbase
categories:
- hbase
---

[原文链接](https://www.cyanny.com/2014/03/13/hbase-architecture-analysis-part1-logical-architecture/)

---
# 1. 总览
---

Apache HBase是一个开源的面向列的数据库。它通常被描述为一个稀疏的，一致的，分布式的，多维排序的Map。 HBase模仿Google的“Bigtable：一种分布式结构化数据存储系统”，它可以容纳数十亿行，X百万列的超大型表格。

HBase是一个no-SQL数据库，它与传统的RDBMS有着不同的范式，传统的RDBMS强调SQL以及数据之间的关系，而HBase存储的是具有键值风格的结构化和半结构化数据。 HBase中的每一行都由<Rowkey，Column Family，Column Qualifier，Version>定位。

HBase的数据存储在分布式的HDFS上，具有很高的可靠性，可扩展性和可用性，但缺点是HDFS将数据拆分成块（64MB或128MB），可以很好地处理大块数据，但对于小块数据和低延迟数据访问却不太好。换句话说，HDFS不适合在线实时读取和写入，这也就是HBase存在的意义。HBase的目标是利用分布式数据存储来实现在线实时数据访问。

---
# 2. 使用场景
---

**图2.1 Hbase的使用场景：**![Hbase的使用场景](http://i60.tinypic.com/346rmhh.jpg)

```
Data random online, real time access: HBase provides block cache and index on files for real time queries.
```

数据随机在线，实时访问：HBase为文件提供块缓存和索引以实时查询。

---
# 3. 逻辑架构
---

**图3.1 Hbase架构-HighLevel：**![Hbase架构-HighLevel](http://i60.tinypic.com/2eezt6x.jpg)

## 1. 分层架构

- 客户端层：各种API
- HBase层：两个重要的组件，HMaster 和 RegionServer
- HDFS：分布式文件存储和备份

## 2. 主从架构

对于分布式应用程序，如何处理数据块是一个关键点。有两种典型风格：集中式和非集中式。集中式模式非常适用于可扩展性和负载平衡器，但主节点上存在单点失效（SPOF）问题和性能瓶颈。非集中模式没有SPOF问题和性能瓶颈，但负载均衡并不容易，并且存在数据一致性问题。

HBase采用集中模式：主从风格。顺便说一句，Hadoop也是主从。 Hadoop生态系统中的架构风格是一种一致的主从风格。

**图3.2 Master-Slave架构：**![Master-Slave架构](http://i62.tinypic.com/ieff35.jpg)

- HMaster：负责监控RegionServer，region分配，元数据操作，RegionServer故障转移等。在分布式集群中，HMaster在HDFS NameNode上运行。
- RegionServer：负责服务和管理region。在分布式集群中，它在HDFS DataNode上运行。
- Zookeeper：Zookeeper将跟踪RegionServer的状态，其中托管ROOT表，可以查询region在server上的分布。自HBase 0.90.x以来，它引入了与Zookeeper更紧密的集成。

---
# 3.2 Hbase的数据模型
---

基于列簇的数据模型：

- kv存储：表中的每个单元格都由四个维度键组成，Rowkey，Column Family，Column Qualifier，Version，表不会像RDBMS那样存储空值，非常适合于存储大型稀疏表格。
- 按键排序，只对键创建索引：row key设计是HBase Schema Design中唯一最重要的事情。
- 每个cell可以存储不同版本的数据：通过时间戳来区别不同的版本
- 列的限定符是无模式的，可以在运行时更改
- 列限定符和值按照字节码存储。

**表3.1 数据模型的逻辑视图：**![数据模型的逻辑视图](https://upload-images.jianshu.io/upload_images/3151600-5bec08f551f2c8fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如表3.1中所示，列按照列簇来组织，每行都有唯一的键row key，某个row key对应的某个列的值可以有多个版本，默认通过时间戳来标识。

**表3.2 数据模型的物理视图：**![数据模型的物理视图](https://upload-images.jianshu.io/upload_images/3151600-243d746a3cda3975.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果按照表3.1中的结构进行存储，也就是按照行存储，会保存很多null值，但是在HBase中，通过列示存储，通过kv的形式只保存非空的值。实际的物理存储中，一个表通过列簇被拆分，每个列族被保存在RegionServer的region中的HStore中。

---
# 3.3 HBase的分布式模型
---

在实际的分布式存储过程中，HBase的Table被拆分存储在集群中。

**图3.3 Table的拆分：**![Table的拆分](http://i62.tinypic.com/16h0rdk.jpg)

表被拆分成大致相同的region，分布到集群的不同的server中。

## 3.3.2 Region server

**图3.4 Region Server的架构：**![Region Server的架构](http://i60.tinypic.com/c28l.jpg)

- 一个 Block Cache：用于数据读取的LRU优先级缓存。
- 一个 One WAL(Write Ahead Log)：HBase使用Log-Structured-Merge-Tree(LSM tree)来处理数据，创建索引，数据写到MemStore之前会先写入WAL，WAL会被持久化到HDFS上，这里注意一个Region Server只有一个WAL，不会为每个Table单独创建一个WAL。
- 多个HRegions：table被拆分成多个HRegions。
- 一个HRegion对应多个HStore：一个HRegion中会按照列簇被拆分成多个HStore。
- 一个HStore对应一个MemStore：数据的更改首先在MemStore中执行。
- 一个HStore对应一个HFile：不能修改的，从MemStore中flush到HDFS上的文件。

## 3.3.3 -ROOT- and .META 表

数据被拆分成不同的region，查找数据的时候如何定位到正确的region上，-ROOT- and .META两个目录表就是干这件事。

- .META. 表：保存region的位置信息，region中保存了执行范围的row key的数据，.META. 表会被拆分到不同的region server中。
- -ROOT- 表：保存.META. 表的信息，-ROOT- 表只有一个，不会被拆分。

总体上讲就是：-ROOT-中保存.META.所在的服务器，.META.中保存region所在的服务器。

## 3.3.4 region查找

**图3.6 region查找：**![region查找](http://i60.tinypic.com/30ml56q.jpg)

通过zookeeper来实现region的查找，假设需要找row T10006：

- 通过zk找到-ROOT-所在的服务器，假设找到RS1
- 到RS1上的-ROOT-找到哪个.META.包含T10006，假设找到META1 on RS2
- 到RS2的META1中查找T10006位于哪个region，假设找到Region on RS3
- 去RS3的region上找到T10006
- 如果该region存在与缓存中，优先从缓存中取

---
# 3.5 Log-Structured Merge-Trees(LSM-trees)
---

**图3.8 B+树：**![B+树](http://i58.tinypic.com/7286l0.jpg)

NO-SQL数据库通常使用LSM树作为数据存储过程体系结构， HBase也不例外。众所周知，RDBMS采用B树来组织其索引，如Fugure 3.8所示。这些B树通常是3级n向平衡树。 B+树的节点是磁盘上的块，所以对于RDBMS的更新，它可能需要5倍的磁盘操作（B+树查找目标行块1次，目标块读取1次，数据更新1次）。B+树非常适合读取数据，但对数据更新效率不高。考虑到大量的分布式数据，到目前为止，B树并不是LSM树的竞争对手。

**图3.9 LSM树的合并：**![LSM树的合并](http://i59.tinypic.com/2mw8nky.jpg)

LSM树可以被看作n级合并树。它使用日志文件和内存存储将随机写入转换为顺序写入。图3.9显示了LSM-tree的数据写入过程。

首先将数据顺序写入日志文件，然后写入内存中，数据组织为排序树，如B+树。当内存中的存储已满时，内存中的树将被刷新到磁盘上的存储文件。磁盘上的存储文件像B+树一样排列，如图3.9所示的C1树。但是存储文件针对顺序磁盘访问进行了优化。

LSM-tree的数据更新在内存中运行，不需要磁盘访问，它比B+树更快。当数据读取总是在最近写入的数据集上时，LSM树将减少磁盘搜索并提高性能。当磁盘IO是我们必须考虑的成本时，LSM树比B+树更合适。
