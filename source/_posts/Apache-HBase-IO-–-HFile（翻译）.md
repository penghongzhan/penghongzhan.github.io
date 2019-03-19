---
title: Apache HBase IO – HFile（翻译）
date: 2018-05-19 14:19:22
tags:
- hbase
- hfile
categories:
- hbase
---

[原文链接](http://blog.cloudera.com/blog/2012/06/hbase-io-hfile-input-output/)

---
# 介绍
---

Apache HBase是Hadoop开源，分布式，版本化的存储管理器，非常适合随机，实时读/写访问。

等等？随机的、实时的读写？

怎么可能？Hadoop不仅仅是一个连续的读/写批处理系统吗？

是的，我们在谈论同样的事情，在接下来的几段中，我将向您解释HBase如何实现随机I / O，它如何存储数据以及HBase HFile格式的演变。

---
# Apache Hadoop I/O file formats
---

Hadoop使用SequenceFile文件格式，您可以使用它来追加键/值对，但由于hdfs仅具有附加功能，文件格式不允许修改或删除插入的值。允许的唯一操作是追加，如果您想查找指定的key，则必须通读文件直到找到key。

*正如你所看到的，你不得不遵循顺序读/写模式......但是如何在HBase的基础上构建一个随机，低延迟的读写访问系统？*

为了帮助你解决这个问题，Hadoop有另一种文件格式，称为MapFile，它是SequenceFile的扩展。MapFile实际上是一个包含两个SequenceFiles的目录：数据文件“/ data”和索引文件“/ index”。MapFile允许您添加排序的键/值对，每经过N个keys之后（其中N是一个可配置的时间间隔），它将索引中的键和偏移量存储在index中。通过索引的查找会相当快，因为不是扫描所有记录，而是扫描具有较少条目的索引。一旦找到你的区块，你就可以跳入真实的数据文件。

MapFile很好，因为您可以快速查找键/值对，但仍然存在两个问题：

- 我如何删除或修改键/值？
- 当我的输入没有排序时，我无法使用MapFile。

---
# HBase & MapFile
---

HBase的rowkey由以下部分组成：行键，列族，列限定符，时间戳和类型。

![HBase的rowkey](http://upload-images.jianshu.io/upload_images/3151600-bc6fc8de21e744d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决删除键/值对的问题，想法是使用“type”字段将键标记为已删除（墓碑标记）。解决修改键/值对的问题只是选择较晚的时间戳（正确的值接近文件末尾，接近末尾的记录对应的时间戳也就更晚）。

为了解决“无序”关键问题，我们将最后添加的键值保存在内存中（内存中的key是经过排序的）。当你达到一个阈值时，HBase将其刷新到一个MapFile。通过这种方式，最终将排序后的键/值添加到MapFile中。

HBase正是这样做的：当你用table.put()添加一个值时，你的键/值被添加到MemStore中（底层的MemStore是一个排序的ConcurrentSkipListMap）。当达到per-memstore阈值（hbase.hregion.memstore.flush.size）或者RegionServer对memstores使用的内存过多时（hbase.regionserver.global.memstore.upperLimit），数据作为新的MapFile刷新到磁盘上。

每次刷新的结果都是一个新的MapFile，这意味着要找到一个key，您必须在多个文件中进行搜索。这需要更多的资源，并且可能会更慢。

每次进行查找或扫描时，HBase都会扫描每个文件以查找结果，为避免跳过太多文件，有一个线程会检测您何时到达一定数量的文件（hbase.hstore.compaction的.max）。然后它会尝试将这些文件添加到一个名为压缩的过程中，这个过程基本上是将已有的文件合并而创建一个新的大文件。

HBase有两种类型的压缩：一种称为“轻微压缩”，将两个或更多小文件合并为一个，另一种称为“主压缩”，用于找到区域中的所有文件，合并它们并执行一些清理。在主压缩中，已删除的键/值将被删除，此新文件不包含逻辑删除标记，并且所有重复的键/值（更新值操作）都将被删除。

截至0.20版本，HBase使用MapFile格式存储数据，但在0.20版本中引入了新的特定于HBase的MapFile（HBASE-61）。

---
# HFile v1
---

在HBase 0.20中，MapFile被替换为HFile：HBase的特定映射文件实现。这个想法与MapFile很相似，但它增加了更多的功能，而不仅仅是一个普通的键/值文件。诸如对元数据和索引的支持等功能现在保存在同一个文件中。

数据块包含实际的键/值作为MapFile。对于每个“块关闭操作”，将第一个键添加到索引，并将索引写入HFile。

HFile格式还增加了两个额外的“元数据”块类型：Meta和FileInfo。这两个键/值块在HFile关闭时写入。

Meta block设计用于将大量数据与其key保存为一个字符串，而FileInfo是一个简单的Map，针对少量信息的保存，其中的键和值都是字节数组。 Regionserver的StoreFile使用Meta block来存储Bloom Filter，FileInfo用于Max SequenceId，Major compaction key和时间范围信息。如果文件对于我们查找的信息来说太旧（Max SequenceId）或者太新（Timerange），不可能包含我们正在查找的key（布尔过滤器），则此信息对于避免读取某些文件非常有用。

![HFile v1](http://upload-images.jianshu.io/upload_images/3151600-fb2ea80a97ea623b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![HFile v1(彩图)](http://upload-images.jianshu.io/upload_images/3151600-34b54c77dd5039ce.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![HFile v1 DataBlock的kv存储结构](http://upload-images.jianshu.io/upload_images/3151600-7e07ce54b9be2d5f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
# HFile v2
---

在HBase 0.92中，HFile格式稍作修改（HBASE-3857）以提高存储大量数据时的性能。 HFile v1的主要问题之一是需要在内存中加载所有的单个索引和大型Bloom Filters，为了解决这个问题，v2引入了多级索引和块级Bloom Filter。因此，HFile v2具有更高的速度以及内存和缓存使用率。

[图片上传失败...(image-ae7ad0-1526710715918)]

这个v2的主要特点是“内联块”，其思想是将索引和Bloom Filter拆分到每个块中，而不是将整个文件的整个索引和布隆过滤器放在内存中。通过这种方式，您可以只将需要的加载到内存中。

由于索引被移动到块级别，因此会产生多级索引，每个块都有自己的索引（叶索引）。每个块的最后一个键被用来创建中间索引，从而使多级索引像一棵b+树。

![block header](http://upload-images.jianshu.io/upload_images/3151600-5a9914ebf466d818.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

块头中包含一些信息，原来的"block magic"被"block type"替代，用于描述块的“数据”、叶子节点、布隆、元数据、根节点等信息。同时，压缩/解压缩大小和前一个块的偏移量三个信息被加入，用于快速的向前或向后的搜索。

---
# Data Block Encodings
---

由于key经过排序并且通常非常相似，因此可以设计比通用算法能够做得更好的压缩。

![无压缩的KV存储](http://upload-images.jianshu.io/upload_images/3151600-dc65568890288d16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

HBASE-4218试图解决这个问题，在HBase 0.94中，你可以选择几种不同的算法：前缀和差分编码。

![前缀压缩](http://upload-images.jianshu.io/upload_images/3151600-10144b5d8fb2c0fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前缀编码的主要思想是只存储一次公共前缀，因为行被排序，所以开始部分通常是相同的。

![差分压缩](http://upload-images.jianshu.io/upload_images/3151600-e5f78a13ce4f5e56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Diff编码进一步加强了压缩。 Diff编码器不是将key视为不透明的字节序列，而是将每个key字段分开，以便以更好的方式压缩每个部分。这样的话列族只存储一次。如果密钥长度、值的长度和类型与之前的行相同，则该字段被省略。此外，为了提高压缩率，存储的时间戳将作为前一个的Diff存储。

请注意，此功能默认为关闭，因为写入和扫描速度较慢，但会缓存更多数据。要启用此功能，您可以设置表信息的`DATA_BLOCK_ENCODING = PREFIX | DIFF | FAST_DIFF`。
