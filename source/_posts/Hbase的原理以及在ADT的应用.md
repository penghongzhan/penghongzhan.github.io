---
title: Hbase的原理以及在ADT的应用
date: 2019-04-03 09:00:00
tags:
- hbase
- 原理
- 概论
categories:
- hbase
---

**目录**

[toc]

---
# 1 Hbase的应用场景
---

HBase是一个分布式的、面向列的开源数据库，该技术来源于 Fay Chang 所撰写的Google论文“Bigtable：一个结构化数据的分布式存储系统”。就像Bigtable利用了Google文件系统（File System）所提供的分布式数据存储一样，HBase在Hadoop之上提供了类似于Bigtable的能力。

**图：HBase的使用场景**![HBase的使用场景.jpg](https://upload-images.jianshu.io/upload_images/3151600-0e21a154005cb221.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.1 对比Mysql

KV存储，通过rowkey进行查询，类似于mysql的唯一主键，这种简单的查询场景下，可以理解为分布式的mysql，由于hbase的数据预排序以及其内部的索引结构，决定了Hbase对于rowkey的随机查询和范围查询，速度都很快。但是没有mysql那么强大的功能，例如聚合，各种联合索引，事务等。

hbase是为大数据而生的，支持千万并发，PB存储，它内部采用列式存储，决定他适合存储以下类型的数据：

- 非结构化数据
- 稀疏表

---
# 2 架构
---

**图：Hbase的基本架构**![hbase基本架构.png](https://upload-images.jianshu.io/upload_images/3151600-dcdeee461be28e88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- master：
   - 表的操作，例如修改列族配置等
   - region的分配，合并，分割
- zookeeper：
   - 维护服务器存活、是否可访问的状态
   - master的HA
   - 记录Hbase的元数据信息的存储位置
- region server：数据的写入与查询
- hdfs：数据的存储，region不直接跟磁盘打交道，通过hdfs实现数据的落盘和读取

---
# 3 数据的写入
---

## 3.1 数据的写入过程

**图：数据的写入概览**![数据的写入概览.jpg](https://upload-images.jianshu.io/upload_images/3151600-3ebe61d7f9d82f28.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- WAL：write ahead log，数据首先写入log，保证数据不丢失，该log也是保存在hdfs上
- MemStore：数据进入内存，按照rowkey进行排序
- HFile：MemStore中的数据到达一定量或者一定时间，创建HFile落盘

**图：数据的写入过程**![数据的写入过程.jpg](https://upload-images.jianshu.io/upload_images/3151600-83d206f69f1705d6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.2 数据格式

HBase存储的所有内容都是byte数组，所以只要能转化成byte数组的数据都可以存储在HBase中。

---
# 4 存储模型
---

**图：HBase的存储概念模型**![HBase的存储概念模型.jpg](https://upload-images.jianshu.io/upload_images/3151600-6c3f6a2385ca9ac3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 表
- 行：每一行都有唯一主键rowkey
- 列族：列族包含若干列，这些列在物理上存储在一起，所以列族内的列一般是在查询的时候需要一起读取
- 列
- 单元格：列的内容保存在单元格中，如果有过更新操作，会有多个版本

---
# 5 存储实现
---

**图：HBase的存储结构**![数据存储结构.jpg](https://upload-images.jianshu.io/upload_images/3151600-8802e6a1e2983dab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 5.1 region

Table的数据以region的形式分布在所有的服务器上。region的存在是为了解决横向扩展问题。

### 5.1.1 region的拆分

通过将数据均衡的分布到所有机器上，可以充分利用各个服务器的能力，提高查询速度。随着数据的不断写入，region会不断增大，region太大会影响查询性能，所以hbase会自动对region进行拆分。

region的拆分策略：

- ConstantSizeRegionSplitPolicy：老版本的hbase使用的拆分策略，按照固定的大小进行拆分，默认为10G，一个region大小超过10G之后，会被拆分成两个。缺点：太死板、太简单，无论是数据写入量大还是小，都是通过这个固定的值来判断
- IncreasingToUpperBoundRegionSplitPolicy：新版本的默认策略，这个策略随着数据增长，动态改变拆分的阈值，公式为`min(tableRegionCount^3*initialSize, defaultRegionMaxFileSize)`
   - tableRegionCount：当前region的数量，开始region数量少的时候，阈值也小；后期region数量变多，阈值随之变大
   - initialSize：一个固定值，如果没有设置，默认为mem flush的大小，一般flush配置的越大，写入吞吐量也大，所以阈值也应该更大
   - defaultRegionMaxFileSize：最大值，默认为10G，后期随着分片数量增多，`tableRegionCount^3*initialSize`可能超过10G，但是region上限是10G
- ...

#### 5.1.2 region合并

场景：region中大量数据被删除，不需要开始那么多region，可以手动进行region的合并

#### 5.1.3 注意

region的合并英文为`merge`，不要跟后续的HFile的合并混淆，HFile的合并英文单词为`compaction`

## 5.2 store

一个region内部有多个store，store是列族级别的概念，一个表有三个列族，那么在一台服务器上的region中会有三个store。

### 5.2.1 MemStore

每个列族/store对应一个独立的MemStore，也就是一块内存空间，数据写入之后，列族的内容进入对应的MemStore，会按照rowkey进行排序，并创建类似于Btree的索引——LMS-Tree。

#### LMS-Tree

LMS-Tree（Log-Structured Merge Tree）理解为merge tree即可。

**图：mysql的B+树：**![mysql的B+树](https://upload-images.jianshu.io/upload_images/3151600-9568289e924e9a4b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


B+tree能有效的提高mysql的查询性能，但是对于高并发大数据量的写入场景，面对大量的随机写，性能无法满足要求。因此产生了LSM-Tree，LSM树采用的索引结构与B+Tree相同，而且通过批量存储技术规避磁盘随机写入问题，因为数据过来之后，首先会在内存中进行排序，构建索引，当到达一定的量的时候，flush到磁盘中，随着磁盘中的小文件的增多，后台进行会自动进行合并，过多的小文件合并为一个大文件，能够有效加快查询速度。

**图：LSM树的合并：**![LSM树的合并](https://upload-images.jianshu.io/upload_images/3151600-c5be11d52a138dcd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### flush时机

- 大小达到刷写阀值
- 整个RegionServer的memstore总和达到阀值
- Memstore达到刷写时间间隔
- WAL的数量大于maxLogs
- 手动触发flush

### 5.2.2 HFile

Hbase的数据文件，Hbase的所有数据都保存在HFile中，查询的时候也是从HFile中进行查询。

#### HFile的结构

**图：Hfile的存储结构**![Hfile的存储结构.png](https://upload-images.jianshu.io/upload_images/3151600-e4662832a50d56fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- scan block：scan查询的时候需要读取的部分
   - data block：数据KV存储
   - leaf index block：Btree的叶子节点
   - bloom block：布隆过滤器
- none scan block
   - meta block
   - intermediate index block：Btree的中间节点
- load on open：HFile加载的时候，需要加载到内存的部分
   - root index block：Btree的根节点
   - meta index
   - file info
   - bloom filter metadata：布隆过滤器的索引
- trailer：记录上面各个部分的偏移量，HFile读取的时候首先读取该部分，然后获取其他部分所在的位置

#### data block的物理结构

**图：无压缩的KV存储**![无压缩的KV存储](http://upload-images.jianshu.io/upload_images/3151600-dc65568890288d16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于key经过排序并且通常非常相似，因此可以设计比通用压缩算法能够做得更好的压缩方式。

**前缀压缩**

**图：前缀压缩**![前缀压缩](http://upload-images.jianshu.io/upload_images/3151600-10144b5d8fb2c0fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前缀编码的主要思想是只存储一次公共前缀，因为行被排序，所以开始部分通常是相同的。

#### Hfile的合并

每次memstore的刷写都会产生一个新的HFile，而HFile毕竟是存储在硬盘上的东西，凡是读取存储在硬盘上的东西都涉及一个操作：寻址，如果是传统硬盘那就是磁头的移动寻址，这是一个很慢的动作。当HFile一多，你每次读取数据的时候寻址的动作就多了，效率就低了。所以为了防止寻址的动作过多，我们要适当地减少碎片文件，所以需要继续合并操作。

合并分类：

- 小合并：小的合并成大的
- 大合并：大的最终合并成一个，注意：只有在大合并之后，标记删除的文档才会真正被删除

合并过程：

- 读取合并列表中的hfile
- 创建数据读取的scanner
- 读取hfile中的内容到一个临时文件中
- 临时文件替换合并之前的多个hfile

---
# 6 数据查询
---

## 6.1 查询顺序

- [1] block cache：HFile的load on open部分是常驻内存的，data block是在磁盘上的，查询的时候，定位到某个data block之后，Hbase会将整个data block加载到block cache中，后续查询的时候，先检查是否存在block cache中，如果是，优先查询block cache。之所以可以这么放心的使用block cache，是基于Hfile的不可变性，后续的修改和删除操作不会直接修改HFile，而是追加新的文件，所以只要HFile还在，对应的block cache就是不变的。
   - 堆内缓存：LRUBlockCache，设计思想有点类似GC的分代回收
   - 堆外缓存：
      - SlabCache：有点不好用
      - BucketCache：阿里大牛基于SlabCache的思想开发的堆外缓存版本，不能说比LRUBlockCache好多少，只能说适用于不同的场景
- [2] region -> memstore + hfile：通过hbase的元数据表，找到需要查询的rowkey所在的region server，从而定位到memstore和hfile

## 6.2 region的查找过程

**图：region的查找过程**![region的查找过程.jpg](https://upload-images.jianshu.io/upload_images/3151600-ef223206721e523e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一个表有多个region，分布在不同机器上，需要一定的机制来确定需要查找的region

- 通过zk找到meta所在的sever：meta表的位置保存在zk中，meta中保存了每个region的rowkey范围，以及region所在的位置
- 通过meta查询出需要查询的region所在的服务器
- 到服务器上进行查询

客户端会对meta信息进行缓存，加快查询速度

## 6.3 hfile的查询过程

hfile中有两种数据结构，能够加快查询

- 数据索引btree：最多三层节点
- 布隆：用于快速的排除不存在的数据

**图：HFile查询：**![HFile查询](https://upload-images.jianshu.io/upload_images/3151600-b6863426090ee72e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

root index在HFile解析的时候直接被加载到内存中。查询的时候：

- 在内存中的root index中通过二分查找快速定位到某个row key对应的下一级节点intermediate index所在的block
- 将intermediate index block读取到内存中，通过二分查找找到某个index entry，进而找到row key对应的叶子节点的block的位置
- 将leaf block读取到内存中，通过二分查找找到某个index entry，进而找到row key对应的data block，将整个数据块加载到内存，通过遍历的方式找到对应的value

## 6.4 查询API

- get：查询某个rowkey对应的列
- scan：指定rowkey范围的扫描（setStartRow, setStopRow）
- filter：scan过程中，对内容进行过滤

**其中指定rowkey范围是最有效的加快查询速度的方式，不限定rowkey的范围则需要全表扫**

---
# 7 rowkey的设计
---

查询速度和数据均衡分布的权衡：

- 完全按照业务字段来设计rowkey，例如使用`用户id+产品id+时间`作为rowkey，相同用户相同产品的信息保存在同一个region中，查询快，但是由于用户的数据量不同，可能导致热点数据，造成某台机器负载过高，影响机群正常工作
- 完全采用随机的rowkey，跟业务不相关，查询的时候只能全表扫，查询效率低

**rowkey长度设计建议：**

- 1. 数据的持久化文件 HFile 中是按照 KeyValue 存储的，如果 rowkey 过长，比如超过 100 字节，1000w 行数据，光 rowkey 就要占用 100*1000w=10 亿个字节，将近 1G 数据，这样会极大影响 HFile 的存储效率；
- 2. MemStore 将缓存部分数据到内存，如果 rowkey 字段过长，内存的有效利用率就会降低，系统不能缓存更多的数据，这样会降低检索效率；
- 3. 目前操作系统都是 64 位系统，内存 8 字节对齐，rowkey长度建议控制在 16 个字节（8 字节的整数倍），充分利用操作系统的最佳特性。

## 7.1 rowkey设计方式-加盐

**图：rowkey设计方式-加盐**![rowkey设计-加盐.png](https://upload-images.jianshu.io/upload_images/3151600-c149d69039c638e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


使用固定的随机前缀：

- 优点：数据均衡
- 缺点：因为前缀是随机的，所以无法快速get；scan的速度还可以

## 7.2 rowkey设计方式-hash

**图：rowkey设计方式-哈希**![rowkey设计-hash.png](https://upload-images.jianshu.io/upload_images/3151600-d693d62f7a4b815b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rowkey hash之后取md5的前五位：

- 优点：打散数据，前缀在查询的时候能通过rowkey得到，可以很快get
- 缺点：相同前缀的rowkey被打散，scan变慢

## 7.3 rowkey设计方式-反转

**图：rowkey设计方式-反转**![rowkey设计-翻转.png](https://upload-images.jianshu.io/upload_images/3151600-2d07791fb2ec64b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


反转一段固定长度的rowkey，或者整个反转。上图中三个网址属于相同域名下的，但是如果不反转，会完全分散到不同的region中，不利于查询。

---
# ADT应用场景
---

- 用户的数据导出服务
- 将产品，推广活动等信息作为rowkey，同一个用户的信息由于rowkey的有序性，存储在一起，从而加快查询速度
- 实时性要求不高

## 替换回HDFS+Spark任务

- 业务上实时性要求不高，但是现在使用hbase查询，实时写入实时可以查询到，如此实时，有点浪费，T+1即可
- HBase根据rowkey进行排序，通过将业务字段作为rowkey，可以加快查询速度。但是感觉**rowkey排序+HFile整个扫描**的方式在查询速度上应该可以满足要求；HBase的数据结构设计，包括分层的BTree索引以及布隆过滤器，很大程度上是为了rowkey的随机查询的场景，尤其是布隆过滤器，在查找rowkey的时候，能够很快排除不存在这个rowkey的block，从而减少磁盘IO。但是对于数据量较大的scan来说，感觉没啥必要。所以感觉，HBase用于ADT的数据导出，还有有些性能上的浪费，如果替换回HDFS+Spark可以节省一部分机器资源。

## Hbase资源占用较大的几个点

- 内存对rowkey排序，构建索引（内存操作，很快）
- WAL：对于数据完整性不是非常严格的情况下，可以开启异步
- rowkey以及列族列明冗余存储（磁盘空间占用大，尤其是rowkey设计不合理的情况）
- hfile merge：最消耗IO和CPU，尤其是大合并，在集群使用低谷时间段进行
- HFile中的btree索引，根节点常驻内存
- HFile中的布隆过滤器，索引常驻内存

### 参考链接

- [hbase-io-hfile-input-output](http://blog.cloudera.com/blog/2012/06/hbase-io-hfile-input-output/)
- [深入理解HBase的系统架构](https://blog.csdn.net/Yaokai_AssultMaster/article/details/72877127)
- [HBase底层存储原理](https://www.cnblogs.com/panpanwelcome/p/8716652.html)
- [HBase – 探索HFile索引机制](http://hbasefly.com/2016/04/03/hbase_hfile_index/)
- [HBase – 存储文件HFile结构解析](http://hbasefly.com/2016/03/25/hbase-hfile/)