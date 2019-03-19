---
title: Hbase学习笔记
date: 2018-05-20 22:39:32
tags:
- hbase
categories:
- hbase
---

---
# Hbase的架构
---

## zookeeper

HBase利用ZooKeeper维护集群中服务器的状态并协调分布式系统的工作。ZooKeeper维护服务器（例如master节点和region server节点）是否存活、是否可访问的状态并提供服务器故障/宕机的通知。ZooKeeper同时还使用一致性算法来保证服务器之间数据的同步，同时也负责Master选举的工作。

## master

HMaster负责region的分配，数据库的创建和删除操作。

具体来说，HMaster的职责包括：

- 调控Region server的工作 
   - 在集群启动的时候分配region，根据恢复服务或者负载均衡的需要重新分配region。
   - 监控集群中的Region server的工作状态。（通过监听zookeeper对于ephemeral node状态的通知）。
- 管理数据库 
   - 提供创建，删除或者更新表格的接口。

## region Server

写入Hbase的数据，通过region server节点创建索引，并且最终保存到HDFS上，region server进行数据存储的概念如下：

- region：hbase的数据抽象，按照Table隔离，也就是说一个region中不会同时存储两个table的数据，一般也只是存储一个表的部分数据，因为一个table需要通过多个region将数据分布到不同的服务器上来实现负载均衡
- store：region中具体的存储，一个store存储的是同一个列簇的数据。region存储的是同一个table的数据，region内部按照列簇分别存储在不同的store中
- HFile：store在hdfs中的具体的物理存储是HFile，由于HFile有大小限制，所以随着数据量的增大，store中会出现越来越多的HFile

数据输入到region server之后，首先进入MemStore（内存缓冲区），当 MemStore达到上限的时候，Hbase会将内存中的数据输出为有序的StoreFile文件数据（根据Rowkey、版本、列名排序，这里已经和列簇无关了因为Store里都属于同一个列簇）。

---
# hbase的存储结构
---

## B+ Tree和LMS（Log-Structured Merge-Tree）

**B+ Tree：**![B+ Tree](http://upload-images.jianshu.io/upload_images/3151600-677e2a977fb37e5c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

无SQL数据库通常使用LSM树作为数据存储过程体系结构，HBase也不例外。众所周知，RDBMS采用B+树来组织其索引，如图《B+ Tree》所示。这些B+树通常是3级n向平衡树，B+树中的节点对应磁盘上的块。所以对于RDBMS的更新，它可能需要5倍的磁盘操作（B+树查找目标行块3次，目标块读取1次，数据更新1次）。 

在RDBMS中，如果没有B+树，数据随机无序写在磁盘块中，读操作需要扫全表，性能会很低。B+树对于数据读操作能很好地提高性能，但对于数据写，效率不高。对于大型分布式数据系统，B+树还无法与LSM树相抗衡。

LSM树（Log-Structured Merge Tree）采用的索引结构与B+Tree相同，而且通过批量存储技术规避磁盘随机写入问题，因为数据过来之后，首先会在内存中进行排序，构建索引，当到达一定的量的时候，flush到磁盘中，由于数据已经排序好了，所以批量flush的时候能有效减少磁盘的随机写。当然凡事有利有弊，LSM树和B+树相比，LSM树牺牲了部分读性能，用来大幅提高写性能。

LSM树可以被看作n级合并树，它使用日志文件和内存存储将随机写入转换为顺序写入。首先将数据顺序写入日志文件，然后写入内存中，数据组织为排序树，如B+树。当内存中的存储已满时，内存中的树将被刷新到磁盘上的小的存储文件。当这些小的File数量达到一个阀值的时候，Hbase会用一个线程来把这些小File合并成一个大的File。这样，Hbase就把效率低下的文件中的插入、移动操作转变成了单纯的文件输出、 合并操作。

对于mysql来说，每来一条数据，都会去更新索引，所以写入量大的时候，速度会很慢；但是hbase会首先在内存中形成小的索引文件，到达一定大小之后落盘，后台会将生成的这些小的索引文件进行合并。

## 列式存储——按照rowkey进行排序

Hbase的存储关系：

- 一个HRegion对应多个HStore：一个HRegion中会按照列簇被拆分成多个HStore。
- 一个HStore对应一个MemStore：数据的更改首先在MemStore中执行。
- 一个HStore对应多个StoreFile，一个StoreFile对应一个HFile：HFile是不能修改的，从MemStore中flush到HDFS上的文件。

**HStore的数据结构：**![HStore的数据结构](https://upload-images.jianshu.io/upload_images/3151600-8be332a4a60e5627.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Store数据结构中：

- 每个StoreFile内的数据是有序的，
- 但是StoreFile之间不一定是有序的，
- Store只需要管理StoreFile（即HFile）的索引就可以了。

这里也可以看出为什么指定版本和Rowkey可以加强查询的效率，因为指定版本和Rowkey的查询可以利用 StoreFile的索引跳过一些肯定不包含目标数据的数据。

查询的时候，通过store定位到row key所在的HFile之后，首先通过Trailer读取其中的索引（Data Block Index），如果索引存在于内存中，直接从内存中查找row key对应的数据。

假设一种最简单的情况：HBase现在只有一个region server，现在只有一个表，只有一个列簇，在server上只有一个region，region中只有一个store。

随着数据的写入，内存中创建索引，到达一定大小之后，flush到磁盘上形成一个HFile，假设数据最终写完之后，磁盘上总共形成了3个HFile。

这三个HFile最终合并成一个大的HFile，以提高查询效率。合并完成之后，这个HFile中包含了该表的所有的数据，我们查询某个row key的时候，定位到这个region的这个store，到store的HFile中的B+Tree索引中很快定位到该row key对应的数据所在的block。HBase会将HFile中的索引缓存到内存中，除此之外，通过索引定位到row key在磁盘中的位置之后，HBase不是简单的直接将该数据取出来，而是将该数据所在的Block整个缓存到内存中，然后从内存中将该row key对应的值取出来。

---
# HFile的数据结构
---

## HFile的数据结构(v1)

**HFile结构：**![HFile结构](http://upload-images.jianshu.io/upload_images/3151600-22ba18e01baba0e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**DataBlock的kv存储结构：**![DataBlock的kv存储结构](http://upload-images.jianshu.io/upload_images/3151600-abd1f1b464b61f84.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数据最终在磁盘中的保存形式是HFile，分为5个段：

- DataBlock：保存列簇中的数据，按照rowkey进行了排序，可以被压缩。见图《DataBlock的kv存储结构》
- MetaBlock：保存用户自定义的Kv数据，例如bloom filter，可以用于查询过程中过滤掉部分一定不存在待查KeyValue的数据文件。
- FileInfo：用来保存HFile的元数据信息，例如最大的SequenceId，时间范围信息等，也可以用来避免读取某些不存在某个key的文件。
- Data Block Index：Data block的索引文件。
- Meta Block Index：Meta Block的索引文件。
- Trailer：保存上述各个block的的偏移量，server加载HFile信息的时候，首先需要读取Trailer的内容，获取各个Block的位置。

## HFile的数据结构(v2)

HFile是HBase存储数据的文件组织形式，参考BigTable的SSTable和Hadoop的TFile实现。从HBase开始到现在，HFile经历了三个版本，其中V2在0.92引入，V3在0.98引入。HFileV1版本的在实际使用过程中发现它占用内存多，HFile V2版本针对此进行了优化，HFile V3版本和V2版本差异不是特别大，下面介绍HFile(v2)的数据结构。

### block的概念

**图 HFile(v2) 逻辑结构01：**![HFile(v2)](https://upload-images.jianshu.io/upload_images/3151600-b20fbfc644e36732.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Scanned block section: 在 HBase 顺序扫描 HFiles 的时候需要被读取的块内容，包括Data Block、leaf index block以及bloom filter block（图中没有画出）；
- Non-scanned block section: 在 HBase 顺序扫描 HFiles 的时候不会被读取的块内容，主要包括Meta Block和Intermediate Level Data Index Blocks两部分；
- Load-on-open-section: 这部分数据在region server启动时，需要被加载到 BlockCache 中。包括FileInfo、Bloom filter block、data block index和meta block index；
- Trailer: 这部分主要记录了HFile的基本信息、各个部分的偏移值和寻址信息。

HFile会被切分为多个大小相等的block块，每个block的大小可以在创建表列簇的时候通过参数blocksize ＝> ‘65535’进行指定，默认为64k，大号的Block有利于顺序Scan，小号Block利于随机查询，因而需要权衡。各个类型的Block的作用如下：

**各个类型的Block的作用：**![各个类型的Block的作用](http://upload-images.jianshu.io/upload_images/3151600-48a851ce8e5b4805.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### index block的作用

HFile中索引结构根据索引层级的不同分为两种：single-level和mutil-level，前者表示单层索引，后者表示多级索引，一般为两级或三级。HFile V1版本中只有single-level一种索引结构，V2版本中引入多级索引。之所以引入多级索引，是因为随着HFile文件越来越大，Data Block越来越多，索引数据也越来越大，已经无法全部加载到内存中（V1版本中一个Region Server的索引数据加载到内存会占用几乎6G空间），多级索引可以只加载部分索引，降低内存使用空间。

#### Root Index Block

Root Index Block表示索引树根节点索引块，可以作为bloom的直接索引，也可以作为data索引的根索引。而且对于single-level和mutil-level两种索引结构对应的Root Index Block略有不同，本文以mutil-level索引结构为例进行分析（single-level索引结构是mutual-level的一种简化场景），在内存和磁盘中的格式如下图所示：

**Root Index Block的结构：**![Root Index Block的结构](http://upload-images.jianshu.io/upload_images/3151600-121b4992f33f4c1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中Index Entry表示具体的索引对象，每个索引对象由3个字段组成：

- Block Offset表示索引指向数据块的偏移量
- BlockDataSize表示索引指向数据块在磁盘上的大小
- BlockKey表示索引指向数据块中的第一个key

除此之外，还有另外3个字段用来记录MidKey的相关信息：

- MidLeafBlockOffset：索引的叶子节点中，中间的节点对应的磁盘的offset
- MidLeafBlockOndiskSize：索引的叶子节点中，中间的节点占用的磁盘空间
- MidKeyEntry：索引的叶子节点中，中间的节点的mid-key

其中，MidKey表示HFile所有Data Block中中间的一个Data Block，用于在对HFile进行split操作时，快速定位HFile的中间位置。需要注意的是single-level索引结构和mutil-level结构相比，就只缺少MidKey这三个字段。

Root Index Block会在HFile解析的时候直接加载到内存中，此处需要注意在Trailer Block中有一个字段为dataIndexCount，就表示此处Index Entry的个数。因为Index Entry并不定长，只有知道Entry的个数才能正确的将所有Index Entry加载到内存。

#### NonRoot Index Block

当HFile中Data Block越来越多，single-level结构的索引已经不足以支撑所有数据都加载到内存，需要分化为mutil-level结构。mutil-level结构中NonRoot Index Block作为中间层节点或者叶子节点存在，无论是中间节点还是叶子节点，其都拥有相同的结构，如下图所示：

**NonRoot Index Block的结构：**![NonRoot Index Block的结构](http://upload-images.jianshu.io/upload_images/3151600-494ec2f6b1f70036.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

和Root Index Block相同，NonRoot Index Block中最核心的字段也是Index Entry，用于指向叶子节点块或者数据块。不同的是，NonRoot Index Block结构中增加了block块的内部索引entry Offset字段，entry Offset表示index Entry在该block中的相对偏移量（相对于第一个index Entry)，用于实现block内的二分查找。所有非根节点索引块，包括Intermediate index block和leaf index block，在其内部定位一个key的具体索引并不是通过遍历实现，而是使用二分查找算法，这样可以更加高效快速地定位到待查找key。

---
# Hbase的索引与查询
---

root index在HFile解析的时候直接被加载到内存中，root index entry的个数信息通过trailer block获取，都加载到内存中。查询的时候：

- 在内存中很快定位到某个row key对应的root节点的下一级节点intermediate index中某个block的位置
- 在该non-root index block中，通过二分查找找到某个index entry，进而找到row key对应的叶子节点的block的位置
- 在该non-root index block中，通过二分查找找到某个index entry，进而找到row key对应的data block，将整个数据块加载到内存，通过遍历的方式找到对应的value。

**图 HFile 查询：**![HFile查询](https://upload-images.jianshu.io/upload_images/3151600-b6863426090ee72e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
# hbase和mysql对比
---

两者都用B+tree作为自己的索引结构，但是应用的场景却不同，这里对比起来可能没有太大的意义，在各自的场景下有各自的优势。

- Hbase适用于数据写入量大的场景，在B+Tree的基础上使用了LMT作为索引结构。它使用日志文件和内存存储将随机写入转换为顺序写入，首先将数据顺序写入日志文件，然后写入内存中，数据组织为B+树。当内存中的存储已满时，内存中的树将被刷新到磁盘上的小的存储文件。当这些小的File数量达到一个阀值的时候，Hbase会用一个线程来把这些小File合并成一个大的File。这样，Hbase就把效率低下的文件中的插入、移动操作转变成了单纯的文件输出、 合并操作。另外HBase的分布式特性，能够有效的实现集群的容错和机器的扩容等。
- mysql在数据量小的情况下，查询更快，尤其是对于随机查询的场景，因为通过B+Tree能够快速定位到数据的位置，但是HBase涉及到从zookeeper中获取数据所在的位置，再从region对应的store中进行查找，如果数据没有被缓存，还需要先将数据所在的Block加载到内存，再在内存中查找出来。而且mysql支持多索引，查询场景多的列都可以创建索引，但是HBase只支持按照row key进行索引，如果需要过滤的内容不在row key中，需要遍历块中的所有的数据进行查找。

---

### 参考博客

- [深入理解HBase的系统架构](https://blog.csdn.net/Yaokai_AssultMaster/article/details/72877127)
- [HBase底层存储原理](https://www.cnblogs.com/panpanwelcome/p/8716652.html)
- [HBase – 探索HFile索引机制](http://hbasefly.com/2016/04/03/hbase_hfile_index/)
- [HBase – 存储文件HFile结构解析](http://hbasefly.com/2016/03/25/hbase-hfile/)


   
