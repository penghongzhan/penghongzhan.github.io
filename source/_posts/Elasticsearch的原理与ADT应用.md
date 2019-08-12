---
title: Elasticsearch的原理与ADT应用
date: 2019-08-12 19:18:23
tags:
- elasticsearch
- elastic
- 原理
- ADT
categories:
- elasticsearch
---

---
# 简介
---

Elasticsearch（简称ES）是一个分布式、可扩展、实时的搜索与数据分析引擎。ES不仅仅只是全文搜索，还支持结构化搜索、数据分析、复杂的语言处理、地理位置和对象间关联关系等。

ES的底层依赖Lucene，Lucene可以说是当下最先进、高性能、全功能的搜索引擎库。但是Lucene仅仅只是一个库。为了充分发挥其功能，你需要使用Java并将Lucene直接集成到应用程序中。更糟糕的是，您可能需要获得信息检索学位才能了解其工作原理，因为Lucene非常复杂——《ElasticSearch官方权威指南》。

鉴于lucene如此强大却难以上手的特点，诞生了ES。ES也是使用Java编写的，它的内部使用Lucene做索引与搜索，它的目的是隐藏lucene的复杂性，取而代之的提供一套简单一致的RESTful API。

总体来说，ES具有如下特点：

- 一个分布式的实时文档存储引擎，每个字段都可以被索引与搜索
- 一个分布式实时分析搜索引擎，支持各种查询和聚合操作
- 能胜任上百个服务节点的扩展，并可以支持PB级别的结构化或者非结构化数据

---
# 架构
---

## 节点类型

ES的架构很简单，集群的HA不需要依赖任务外部组件（例如zookeeper、HDFS等），master节点的主备依赖于内部自建的选举算法，通过副本分片的方式实现了数据的备份的同时，也提高了并发查询的能力。

ES集群的服务器分为以下四种角色：

- master节点，负责保存和更新集群的一些元数据信息，之后同步到所有节点，所以每个节点都需要保存全量的元数据信息：
  - 集群的配置信息
  - 集群的节点信息
  - 模板template设置
  - 索引以及对应的设置、mapping、分词器和别名
  - 索引关联到的分片以及分配到的节点
- datanode：负责数据存储和查询
- coordinator：
  - 路由索引请求
  - 聚合搜索结果集
  - 分发批量索引请求
- ingestor：
  - 类似于logstash，对输入数据进行处理和转换

### 如何配置节点类型

- 一个节点的缺省配置是：主节点+数据节点两属性为一身。对于3-5个节点的小集群来讲，通常让所有节点存储数据和具有获得主节点的资格。
- 专用协调节点（也称为client节点或路由节点）从数据节点中消除了聚合/查询的请求解析和最终阶段，随着集群写入以及查询负载的增大，可以通过协调节点减轻数据节点的压力，可以让数据节点更多专注于数据的写入以及查询。

## master选举

### 选举策略

- 如果集群中存在master，认可该master，加入集群
- 如果集群中不存在master，从具有master资格的节点中选id最小的节点作为master

### 选举时机

- 集群启动：后台启动线程去ping集群中的节点，按照上述策略从具有master资格的节点中选举出master
- 现有的master离开集群：后台一直有一个线程定时ping master节点，超过一定次数没有ping成功之后，重新进行master的选举

### 选举流程

![ES的master选举流程图](http://upload-images.jianshu.io/upload_images/3151600-e27bceb411f83f11.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

### 避免脑裂

脑裂问题是采用master-slave模式的分布式集群普遍需要关注的问题，脑裂一旦出现，会导致集群的状态出现不一致，导致数据错误甚至丢失。

ES避免脑裂的策略：过半原则，可以在ES的集群配置中添加一下配置，避免脑裂的发生

```
#一个节点多久ping一次，默认1s
discovery.zen.fd.ping_interval: 1s
##等待ping返回时间，默认30s
discovery.zen.fd.ping_timeout: 10s
##ping超时重试次数，默认3次
discovery.zen.fd.ping_retries: 3
##选举时需要的节点连接数，N为具有master资格的节点数量
discovery.zen.minimum_master_nodes=N/2+1
```

### 注意问题

- 配置文件中加入上述避免脑裂的配置，对于网络波动比较大的集群来说，增加ping的时间和ping的次数，一定程度上可以增加集群的稳定性
- 动态的字段field可能导致元数据暴涨，新增字段mapping映射需要更新mater节点上维护的字段映射信息，master修改了映射信息之后再同步到集群中所有的节点，这个过程中数据的写入是阻塞的。所以建议关闭自动mapping，没有预先定义的字段mapping会写入失败
- 通过定时任务在集群写入的低峰期，将索引以及mapping映射提前创建好

## 负载均衡

ES集群是分布式的，数据分布到集群的不同机器上，对于ES中的一个索引来说，ES通过分片的方式实现数据的分布式和负载均衡。创建索引的时候，需要指定分片的数量，分片会均匀的分布到集群的机器中。分片的数量是需要创建索引的时候就需要设置的，而且设置之后不能更改，虽然ES提供了相应的api来缩减和扩增分片，但是代价是很高的，需要重建整个索引。

考虑到并发响应以及后续扩展节点的能力，分片的数量不能太少，假如你只有一个分片，随着索引数据量的增大，后续进行了节点的扩充，但是由于一个分片只能分布在一台机器上，所以集群扩容对于该索引来说没有意义了。

但是分片数量也不能太多，每个分片都相当于一个独立的lucene引擎，太多的分片意味着集群中需要管理的元数据信息增多，master节点有可能成为瓶颈；同时集群中的小文件会增多，内存以及文件句柄的占用量会增大，查询速度也会变慢。

## 数据副本

ES通过副本分片的方式，保证集群数据的高可用，同时增加集群并发处理查询请求的能力，相应的，在数据写入阶段会增大集群的写入压力。

数据写入的过程中，首先被路由到主分片，写入成功之后，将数据发送到副本分片，为了保证数据不丢失，最好保证至少一个副本分片写入成功以后才返回客户端成功。

### 相关配置

#### 5.0之前通过consistency来设置

consistency参数的值可以设为 ：

- one ：只要主分片状态ok就允许执行写操作
- all：必须要主分片和所有副本分片的状态没问题才允许执行写操作
- quorum：默认值为quorum，即大多数的分片副本状态没问题就允许执行写操作，副本分片数量计算方式为`int( (primary + number_of_replicas) / 2 ) + 1`

#### 5.0之后通过`wait_for_active_shards`参数设置

- 索引时增加参数：`?wait_for_active_shards=3`
- 给索引增加配置：`index.write.wait_for_active_shards=3`

---
# 数据写入
---

## 写入过程

几个概念：

- 内存buffer
- translog
- 文件系统缓冲区
- refresh
- segment（段）
- commit
- flush

![ES的数据写入过程(参考ElasticSearch权威指南)](https://upload-images.jianshu.io/upload_images/3151600-e398187e5316f5e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### translog

写入ES的数据首先会被写入translog文件，该文件持久化到磁盘，保证服务器宕机的时候数据不会丢失，由于顺序写磁盘，速度也会很快。

- 同步写入：每次写入请求执行的时候，translog在fsync到磁盘之后，才会给客户端返回成功
- 异步写入：写入请求缓存在内存中，每经过固定时间之后才会fsync到磁盘，写入量很大，对于数据的完整性要求又不是非常严格的情况下，可以开启异步写入

### refresh

经过固定的时间，或者手动触发之后，将内存中的数据构建索引生成segment，写入文件系统缓冲区

### commit/flush

超过固定的时间，或者translog文件过大之后，触发flush操作：

- 内存的buffer被清空，相当于进行一次refresh
- 文件系统缓冲区中所有segment刷写到磁盘
- 将一个包含所有段列表的新的提交点写入磁盘
  - 启动或重新打开一个索引的过程中使用这个提交点来判断哪些segment隶属于当前分片
- 删除旧的translog，开启新的translog

### merge

上面提到，每次refresh的时候，都会在文件系统缓冲区中生成一个segment，后续flush触发的时候持久化到磁盘。所以，随着数据的写入，尤其是refresh的时间设置的很短的时候，磁盘中会生成越来越多的segment：

- segment数目太多会带来较大的麻烦。 每一个segment都会消耗文件句柄、内存和cpu运行周期。
- 更重要的是，每个搜索请求都必须轮流检查每个segment，所以segment越多，搜索也就越慢。

merge的过程大致描述如下：

- 磁盘上两个小segment：A和B，内存中又生成了一个小segment：C
- A,B被读取到内存中，与内存中的C进行merge，生成了新的更大的segment：D
- 触发commit操作，D被fsync到磁盘
- 创建新的提交点，删除A和B，新增D
- 删除磁盘中的A和B

## 删改操作

### segment的不可变性的好处

- segment的读写不需要加锁
- 常驻文件系统缓存（堆外内存）
- 查询的filter缓存可以常驻内存（堆内存）

### 删除

磁盘上的每个segment都有一个.del文件与它相关联。当发送删除请求时，该文档未被真正删除，而是在.del文件中标记为已删除。此文档可能仍然能被搜索到，但会从结果中过滤掉。当segment合并时，在.del文件中标记为已删除的文档不会被包括在新的segment中，也就是说merge的时候会真正删除被删除的文档。

### 更新

创建新文档时，Elasticsearch将为该文档分配一个版本号。对文档的每次更改都会产生一个新的版本号。当执行更新时，旧版本在.del文件中被标记为已删除，并且新版本在新的segment中写入索引。旧版本可能仍然与搜索查询匹配，但是从结果中将其过滤掉。

## 版本控制

通过添加版本号的乐观锁机制保证高并发的时候，数据更新不会出现线程安全的问题，避免数据更新被覆盖之类的异常出现。

- 使用内部版本号：删除或者更新数据的时候，携带`_version`参数，如果文档的最新版本不是这个版本号，那么操作会失败，这个版本号是ES内部自动生成的，每次操作之后都会递增一。
```
PUT /website/blog/1?version=1 
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
```
- 使用外部版本号：ES默认采用递增的整数作为版本号，也可以通过外部自定义整数（long类型）作为版本号，例如时间戳。通过添加参数`version_type=external`，可以使用自定义版本号。内部版本号使用的时候，更新或者删除操作需要携带ES索引当前最新的版本号，匹配上了才能成功操作。但是外部版本号使用的时候，可以将版本号更新为指定的值。
```
PUT /website/blog/2?version=5&version_type=external
{
  "title": "My first external blog entry",
  "text":  "Starting to get the hang of this..."
}
```

---
# 原始文档存储（行式存储）
---

![原始文档的存储结构](https://upload-images.jianshu.io/upload_images/3151600-c206a6f3bb91ed52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## fdt文件

文档内容的物理存储文件，由多个chunk组成，Lucene索引文档时，先缓存文档，缓存大于16KB时，就会把文档压缩存储。

![fdt文件的存储逻辑视图](http://upload-images.jianshu.io/upload_images/3151600-df38fde012faa8c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

## fdx文件

文档内容的位置索引，由多个block组成：

- 1024个chunk归为一个block
- block记录chunk的起始文档ID，以及chunk在fdt中的位置

![fdx文件的存储逻辑视图](http://upload-images.jianshu.io/upload_images/3151600-8d7ac2be300e722c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

## fnm文件

文档元数据信息，包括文档字段的名称、类型、数量等。

## 原始文档的查询

![原始文档的查询过程](https://upload-images.jianshu.io/upload_images/3151600-94b64df0f3f797f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注意问题**

lucene对原始文件的存放是行式存储，并且为了提高空间利用率，是多文档一起压缩，因此取文档时需要读入和解压额外文档，因此取文档过程非常依赖CPU以及随机IO。

## 相关设置

### 压缩方式的设置

原始文档的存储对应`_source`字段，是默认开启的，会占用大量的磁盘空间，上面提到的chunk中的文档压缩，ES默认采用的是`LZ4`，如果想要提高压缩率，可以将设置改成`best_compression`。

```
index.codec: best_compression
```

### 特定字段的内容存储

查询的时候，如果想要获取原始字段，需要在`_source`中获取，因为所有的字段存储在一起，所以获取完整的文档内容与获取其中某个字段，在资源消耗上几乎相同，只是返回给客户端的时候，减少了一定量的网络IO。

ES提供了特定字段内容存储的设置，在设置mappings的时候可以开启，默认是false。如果你的文档内容很大，而其中某个字段的内容又需要经常获取，可以设置开启，将该字段的内容单独存储。

```
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "title": {
          "type": "text",
          "store": true 
        }
      }
    }
  }
}
```

---
# 倒排索引
---

![倒排索引-文件结构.png](https://upload-images.jianshu.io/upload_images/3151600-028d7fa806b41f45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

倒排索引中记录的信息主要有：

- 文档编号：segment内部文档编号从0开始，最大值为int最大值，文档写入之后会分配这样一个顺序号
- 字典：字段内容经过分词、归一化、还原词根等操作之后，得到的所有单词
- 单词出现位置：
  - 分词字段默认开启，提供对于短语查询的支持
  - 对于非常常见的词，例如the，位置信息可能占用很大空间，短语查询需要读取的数据量很大，查询速度慢
- 单词出现次数：单词在文档中出现的次数，作为评分的依据
- 单词结束字符到开始字符的偏移量：记录在文档中开始与结束字符的偏移量，提供高亮使用，默认是禁用的
- 规范因子：对字段长度进行规范化的因子，给予较短字段更多权重

倒排索引的查找过程本质上是通过单词找对应的文档列表的过程，因此倒排索引中字典的设计决定了倒排索引的查询速度，字典主要包括前缀索引（.tip文件）和后缀索引（.tim）文件。

## 字典前缀索引（.tip文件）

一个合格的词典结构一般有以下特点：

-查询速度快
-内存占用小
-内存+磁盘相结合

![FST前缀索引](https://upload-images.jianshu.io/upload_images/3151600-da06eb42db47e07b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Lucene采用的前缀索引数据结构为FST，它的优点有：

- 词查找复杂度为O(len(str))
- 共享前缀、节省空间、内存占用率低，压缩率高，模糊查询支持好
- 内存存放前缀索引，磁盘存放后缀词块

缺点：结构复杂、输入要求有序、更新不易

## 字典后缀（.tim文件）

后缀词块主要保存了单词后缀，以及对应的文档列表的位置。

## 文档列表（.doc文件）

![文档列表的存储结构](http://upload-images.jianshu.io/upload_images/3151600-d5ad4f4d0f0fa0bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

lucene对文档列表存储进行了很好的压缩，来保证缓存友好：

- 差分压缩：每个ID只记录跟前面的ID的差值
- 每256个ID放入一个block中
- block的头信息存放block中每个ID占用的bit位数，因为经过上面的差分压缩之后，文档列表中的文档ID都变得不大，占用的bit位数变少

上图经过压缩之后将6个数字从原先的24bytes压缩到7bytes

### 文档列表的合并

![跳表](https://upload-images.jianshu.io/upload_images/3151600-d1ef4889a7379f51.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ES的一个重要的查询场景是bool查询，类似于mysql中的and操作，需要将两个条件对应的文档列表进行合并。为了加快文档列表的合并，lucene底层采用了跳表的数据结构，合并过程中，优先遍历较短的链表，去较长的列表中进行查询，如果存在，则该文档符合条件。

![倒排索引-文档列表的合并](https://upload-images.jianshu.io/upload_images/3151600-879560f14ff15a72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 倒排索引的查询过程

![倒排索引的查询过程](http://upload-images.jianshu.io/upload_images/3151600-1154c5e25333e2ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

- 内存加载tip文件，通过FST匹配前缀找到后缀词块位置
- 根据词块位置，读取磁盘中tim文件中后缀块并找到后缀和相应的倒排表位置信息
- 根据倒排表位置去doc文件中加载倒排表
- 借助跳表结构，对多个文档列表进行合并

## filter查询的缓存

对于filter过滤查询的结果，ES会进行缓存，缓存采用的数据结构是`RoaringBitmap`，在match查询中配合filter能有效加快查询速度。

- 普通bitset的缺点：内存占用大，RoaringBitmap有很好的压缩特性
- 分桶：解决文档列表稀疏的情况下，过多的0占用内存，每65536个docid分到一个桶，桶内只记录docid%65536
- 桶内压缩：4096作为分界点，小余这个值用short数组，大于这个值用bitset，每个short占两字节，4096个short占用65536bit，所以超过4096个文档id之后，是bitset更节省空间。

![filter缓存的数据结构](http://upload-images.jianshu.io/upload_images/3151600-a1b75e2b246dfd33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

---
# DocValues（正排索引&列式存储）
---

![DocValues文件结构.png](https://upload-images.jianshu.io/upload_images/3151600-d2436a9c050bd4bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

倒排索引保存的是词项到文档的映射，也就是词项存在于哪些文档中，DocValues保存的是文档到词项的映射，也就是文档中有哪些词项。

## 相关设置

`keyword`字段默认开启

## ES6.0（lucene7.0）之前

DocValues采用的数据结构是bitset，bitset对于稀疏数据的支持不好：

- 对于稀疏的字段来说，绝大部分的空间都被0填充，浪费空间
- 由于字段的值之间插入了0，可能本来连续的值被0间隔开来了，不利于数据的压缩
- 空间被一堆0占用了，缓存中缓存的有效数据减少，查询效率也会降低

查询逻辑很简单，类似于数组通过下标进行索引，因为每个value都是固定长度，所以读取文档id为N的value直接从`N*固定长度`位置开始读取`固定长度`即可。

![ES6.0之前DocValues的数据结构](https://upload-images.jianshu.io/upload_images/3151600-4b30618461ddb0d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ES6.0（lucene7.0）

- docid的存储的通过分片加快映射到value的查询速度
- value存储的时候不再给空的值分配空间

因为value存储的时候，空值不再分配空间，所以查询的时候不能通过上述通过文档id直接映射到在bitset中的偏移量来获取对应的value，需要通过获取docid的位置来找到对应的value的位置。

所以对于DocValues的查找，关键在于DocIDSet中ID的查找，如果按照简单的链表的查找逻辑，那么DocID的查找速度将会很慢。lucene7借用了RoaringBitmap的分片的思想来加快DocIDSet的查找速度：

![DocValues的数据结构-lucene7的结构](https://upload-images.jianshu.io/upload_images/3151600-c3565372719565cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 分片容量为2的16次方，最多可以存储65536个docid
- 分片包含的信息：
  - 分片ID
  - 存储的docid的个数（值不为空的DocIDSet）
  - DocIDSet明细，或者标记分片类型（ALL或者NONE）
- 根据分片的容量，将分片分为四种不同的类型，不同类型的查找逻辑不通：
  - ALL：该分片内没有不存在值的DocID
  - NONE：该分片内所有的DocID都不存在值
  - SPARSE：该分片内存在值的DocID的个数不超过4096，DocIDSet以short数组的形式存储，查找的时候，遍历数组，找到对应的ID的位置
  - DENSE：该分片内存在值的DocID的个数超过4096，DocIDSet以bitset的形式存储，ID的偏移量也就是在该分片中的位置

最终DocIDSet的查找逻辑为：

- 计算DocID/65536，得到所在的分片N
- 计算前面N-1个分片的DocID的总数
- 找到DocID在分片N内部的位置，从而找到所在位置之前的DocID个数M
- 找到N+M位置的value即为该DocID对应的value

---
# 数据查询
---

## 查询过程（query then fetch）

- 协调节点将请求发送给对应分片
- 分片查询，返回from+size数量的文档对应的id以及每个id的得分
- 汇总所有节点的结果，按照得分获取指定区间的文档id
- 根据查询需求，像对应分片发送多个get请求，获取文档的信息
- 返回给客户端

### get查询更快

默认根据id对文档进行路由，所以指定id的查询可以定位到文档所在的分片，只对某个分片进行查询即可。当然非get查询，只要写入和查询的时候指定routing，同样可以达到该效果。

### 主分片与副本分片

ES的分片有主备之分，但是对于查询来说，主备分片的地位完全相同，平等的接收查询请求。这里涉及到一个请求的负载均衡策略，6.0之前采用的是轮询的策略，但是这种策略存在不足，轮询方案只能保证查询数据量在主备分片是均衡的，但是不能保证查询压力在主备分片上是均衡的，可能出现慢查询都路由到了主分片上，导致主分片所在的机器压力过大，影响了整个集群对外提供服务的能力。

新版本中优化了该策略，采用了基于负载的请求路由，基于队列的耗费时间自动调节队列长度，负载高的节点的队列长度将减少，让其他节点分摊更多的压力，搜索和索引都将基于这种机制。

## get查询的实时性

ES数据写入之后，要经过一个refresh操作之后，才能够创建索引，进行查询。但是get查询很特殊，数据实时可查。

ES5.0之前translog可以提供实时的CRUD，get查询会首先检查translog中有没有最新的修改，然后再尝试去segment中对id进行查找。5.0之后，为了减少translog设计的负责性以便于再其他更重要的方面对translog进行优化，所以取消了translog的实时查询功能。

get查询的实时性，通过每次get查询的时候，如果发现该id还在内存中没有创建索引，那么首先会触发refresh操作，来让id可查。

## 查询方式

两种查询上下文：

- query：例如全文检索，返回的是文档匹配搜索条件的相关性，常用api：match
- filter：例如时间区间的限定，回答的是是否，要么是，要么不是，不存在相似程度的概念，常用api：term、range

过滤（filter）的目标是减少那些需要进行评分查询（scoring queries）的文档数量。

### 分析器(analyzer)

当索引一个文档时，它的全文域被分析成词条以用来创建倒排索引。当进行分词字段的搜索的时候，同样需要将查询字符串通过相同的分析过程，以保证搜索的词条格式与索引中的词条格式一致。当查询一个不分词字段时，不会分析查询字符串，而是搜索指定的精确值。

可以通过下面的命令查看分词结果：

```
GET /_analyze
{
  "analyzer": "standard",
  "text": "Text to analyze"
}
```

### 相关性

默认情况下，返回结果是按相关性倒序排列的。每个文档都有相关性评分，用一个正浮点数字段score来表示。score的评分越高，相关性越高。

ES的相似度算法被定义为检索词频率/反向文档频率(TF/IDF)，包括以下内容：

- 检索词频率：检索词在该字段出现的频率，出现频率越高，相关性也越高。字段中出现过5次要比只出现过1次的相关性高。
- 反向文档频率：每个检索词在索引中出现的频率，频率越高，相关性越低。检索词出现在多数文档中会比出现在少数文档中的权重更低。
- 字段长度准则：字段的长度是多少，长度越长，相关性越低。 检索词出现在一个短的title要比同样的词出现在一个长的content字段权重更大。

查询的时候可以通过添加`?explain`参数，查看上述各个算法的评分结果。

---
# ADT的应用
---

## 日志查询工具

移动广告监测（ADTracking，简称ADT）的系统会接收媒体发过来的点击数据以及SDK发过来的激活和各种效果点数据，这些数据的处理过程正确与否至关重要，例如，用户的一条激活数据为啥没有归因到点击，这类问题的排查在ADT中很常见，通过将数据流中的各个处理环节的重要日志统一发送到ES，可以很方便的进行查询，客服同学可以通过拼写简单的查询条件排查客户的问题。

- 索引按天创建：定时关闭历史索引，释放集群资源
- 别名查询：数据量增大之后，可以通过拆分索引减轻写入压力，拆分之后的索引采用相同的别名，查询服务不需要修改代码
- 索引重要的设置：
```
{
    "settings": {
        "index": {
            "refresh_interval": "120s",
            "number_of_shards": "12",
            "translog": {
                "flush_threshold_size": "2048mb"
            },
            "merge": {
                "scheduler": {
                    "max_thread_count": "1"
                }
            },
            "unassigned": {
                "node_left": {
                    "delayed_timeout": "180m"
                }
            }
        }
    }
}
```
- 索引mapping的设置
```
{
    "properties": {
        "action_content": {
            "type": "string",
            "analyzer": "standard"
        },
        "time": {
            "type": "long"
        },
        "trackid": {
            "type": "string",
            "index": "not_analyzed"
        }
    }
}
```
- sql插件，通过拼sql的方式，比起拼json更简单

## 点击数据存储（kv存储场景）

ADTracking的点击数据是用户进行广告投放的直接相关数据，应用安装之后，SDK会上报激活事件，我们的系统会去找这个激活事件是否来自于之前用户点击的某个广告，如果是，那么该激活就是一个推广量，也就是用户投放的广告带来的激活。激活后续的效果点数据也都会去查找点击，从点击中获取广告投放的一些信息，所以点击查询在ADTracking的业务中至关重要。

业务的前期，点击数据是存储在Mysql中的，随着后续点击量的暴增，由于mysql不能横向扩展，所以需要更换为新的存储。由于ES拥有横向扩展和强悍的搜索能力，并且之前日志查询工具中也一直使用ES，所以决定使用ES来进行点击的存储。

### 重要的设置

- `"refresh_interval": "1s"`
- `"translog.flush_threshold_size": "2048mb"`
- `"merge.scheduler.max_thread_count": 1`
- `"unassigned.node_left.delayed_timeout": "180m"`

### 结合业务进行系统优化

#### 结合业务定期关闭索引释放资源

ADTracking的点击数据具有有效期的概念，超过有效期的点击，激活不会去归因。点击有效期最长一个月，所以理论上每天创建的索引在一个月之后才能关闭。

但是用户配置的点击有效期大部分都是一天，这大部分点击在集群中保存30天是没有意义的，而且会占用大部分的系统资源。所以根据点击的这个业务特点，将每天创建的索引拆分成两个，一个是有效期是一天的点击，一个是超过一天的点击，有效期一天的点击的索引在一天之后就可以关闭，从而保证集群中打开的索引的数据量维持在一个较少的水平。

#### 结合业务将热点数据单独索引

激活和效果点数据都需要去ES中查询点击，但是两者对于点击的查询场景是有差异的，因为效果点事件（例如登录、注册等）归因的时候不是去直接查找点击，而是查找激活进而找到点击，效果点要找的点击一定是之前激活归因到的，所以激活归因到的这部分点击也就是热点数据。

激活归因到点击之后，将这部分点击单独存储到单独的索引中，由于这部分点击量少很多，所以效果点查询的时候很快。

#### 索引拆分

ADT的点击数据按天进行存储，但是随着点击量的增大，单天的索引大小持续增大，尤其是晚上的时候，merge需要合并的segment数量以及大小都很大，造成了很高的IO压力，导致集群的写入受限。

后续采用了拆分索引的方案，每天的索引按照上午9点和下午5点两个时间点将索引拆分成三个，由于索引之间的segment合并是相互独立的，只会在一个索引内部进行segment的合并，所以在每个小索引内部，segment合并的压力就会减少。

## 其他调优

### 分片的数量

经验值：

- 每个节点的分片数量保持在低于每1GB堆内存对应集群的分片在20-25之间。
- 分片大小为50GB通常被界定为适用于各种用例的限制。

### JVM设置

- 堆内存设置：不要超过32G，在Java中，对象实例都分配在堆上，并通过一个指针进行引用。64为操作系统而言，默认使用64位指针，指针本身对于空间的占用很大，Java使用一个叫作内存指针压缩（compressed oops）的技术来解决这个问题，简单理解，使用32位指针也可以对对象进行引用，但是一旦堆内存超过32G，这个压缩技术不再生效，实际上失去了更多的内存。
- 预留一半内存空间给lucene用，lucene会使用大量的堆外内存空间。
- 如果你有一台128G的机器，一半内存也是64G，超过了32G，可以通过一台机器上启动多个ES实例来保证ES的堆内存小于32G。
- ES的配置文件中加入`bootstrap.mlockall: true`，关闭内存交换。

### 通过`_cat` api获取任务执行情况

```
GET http://localhost:9201/_cat/thread_pool?v&h=host,search.active,search.rejected,search.completed
```

- 完成(completed)
- 进行中(active)
- 被拒绝(rejected)：需要特别注意，说明已经出现查询请求被拒绝的情况，可能是线程池大小配置的太小，也可能是集群性能瓶颈，需要扩容。

### 小技巧

- 重建索引或者批量想ES写历史数据的时候，写之前先关闭副本，写入完成之后，再开启副本。
- ES默认用文档id进行路由，所以通过文档id进行查询会更快，因为能直接定位到文档所在的分片，否则需要查询所有的分片。
- 使用ES自己生成的文档id写入更快，因为ES不需要验证一次自定义的文档id是否存在
