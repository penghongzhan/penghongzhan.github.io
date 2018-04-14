---
title: Druid的segment
date: 2018-04-13 21:43:48
tags:
- druid
- segment
categories:
- druid
---

一个datasource的每个维度（列）会单独创建索引——倒排索引，每个维度有自己独立的倒排索引；指标维度不需要创建倒排索引，只需要保存起来，由于不需要进行聚合，所以保存的时候可以进行压缩。在最终保存的时候，所有的维度和指标对应的数据会组织在一个segment中，segment是按照时间段进行创建的。

segment首先按照规定的时间段进行数据的聚合，时间区间内的数据都会被聚合到一个segment中，但是如果数据量很大，segment的大小超过一定限制（推荐大小为300M~700M），segment会进行拆分成不同的partition，表现在文件名上是后面追加不同的partition号。

---
# segment的核心数据结构
---

图1：数据结构示例

![数据结构示例](https://upload-images.jianshu.io/upload_images/3151600-0e349c1fae92437d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每个维度列的存储需要三种数据结构：

- 维度的值映射成一个ID：上图中Page列的`Justin Bieber, Ke$ha`被映射成`0, 1`
- 维度列的所有值对应对应一连串ID构成的List，即维度值列表：Page列的内容对应的IDList为`[0, 0, 1, 1]`
- 通过bitmap表示哪一行存在该ID：Page列生成的bitmap为`[1, 1, 0, 0]`

通过ID映射维度的值实际上是对维度值的一种压缩。通过bitmap运算可以快速的获取到行号。对于简单的条件聚合，不需要用到2中的ID列表，但是group by等聚合操作需要用到2中的列表。

```
Note that the bitmap is different from the first two data structures: whereas the first two grow linearly in the size of the data (in the worst case), the size of the bitmap section is the product of data size * column cardinality. Compression will help us here though because we know that for each row in 'column data', there will only be a single bitmap that has non-zero entry. This means that high cardinality columns will have extremely sparse, and therefore highly compressible, bitmaps. Druid exploits this using compression algorithms that are specially suited for bitmaps, such as roaring bitmap compression.
```

传统的bitmap的数据结构有很高的压缩空间，druid通过roaing bitmap实现了对基本bitmap的优化压缩。

---
# segment的命名
---

```
Identifiers for segments are typically constructed using the segment datasource, interval start time (in ISO 8601 format), interval end time (in ISO 8601 format), and a version. If data is additionally sharded beyond a time range, the segment identifier will also contain a partition number.
```

随着数据追加到一个segment中，如果数据量超过了一定的限制（300M~700M），但是时间没有超过聚合的时间区间，那么新的数据会生成一个新的segment。因为两个segment对应相同的时间区间，所以在segment命名的基础上需要增加一个partition编号。

所以一个完整的segment明明示例为：`datasource_intervalStart_intervalEnd_version_partitionNum，其中时间的格式为`ISO 8601 format`。

---
# segment的组成成分
---

```
Behind the scenes, a segment is comprised of several files, listed below.

version.bin

4 bytes representing the current segment version as an integer. E.g., for v9 segments, the version is 0x0, 0x0, 0x0, 0x9

meta.smoosh

A file with metadata (filenames and offsets) about the contents of the other smoosh files

XXXXX.smoosh

There are some number of these files, which are concatenated binary data

The smoosh files represent multiple files "smooshed" together in order to minimize the number of file descriptors that must be open to house the data. They are files of up to 2GB in size (to match the limit of a memory mapped ByteBuffer in Java). The smoosh files house individual files for each of the columns in the data as well as an index.drd file with extra metadata about the segment.

There is also a special column called __time that refers to the time column of the segment. This will hopefully become less and less special as the code evolves, but for now it’s as special as my Mommy always told me I am.

In the codebase, segments have an internal format version. The current segment format version is v9.
```

Segments在HDFS上的物理存储包含两个文件：

- descriptor.json文件：是此Segment的描述文件，内容也保存在元信息库的druid_segments表中；
- index.zip：Segment的完整内容统一命名为index.zip，index.zip压缩文件中包含三个文件：`version.bin, metasmoosh, XXXXX.smoosh`。

### version.bin

Segment内部格式版本号，目前有效版本号标识只有v8和v9，默认生成v8格式，再统一转换成v9格式，用户也可以设置直接生成v9格式的Segments，v9格式相对于v8版本，加载进内存更节省空间。对于v9格式来说，version.bin中内容为00000009

### XXXXX.smoosh

xxxxx从00000开始编号，最大为2G，是为了满足Java内存文件映射MappedByteBuffer限制；以图1[数据结构示例]为例，在Indexing Service阶段，Druid会为每列数据生成对应的索引，并将压缩编码后的原始数据以及索引存入到以列名称命名的*.drd文件中(比如图1[数据结构示例]中_time.drd; page.drd; username.drd; ... )，所有的*.drd文件最终会被合并成为*.smoosh文件。其中压缩编码后的每列原始数据包含两部分：(1)使用Jackson序列化的ColumnDescriptor，包含列数据的元信息，比如列数据的类型，是
否包含多值(multi-value)等； (2)剩余部分为编码后的二进制数据。

### metasmoosh

XXXXsmoosh文件是多个*.drd文件的并集，那么在加载内存后如何区分对应的数据？Druid使用meta.smoosh文件记录元信息，包括列名、分片编号与偏移量。以图1[数据结构示例]为例，meta.smoosh文件格式为cvs，包含两部分：

(1) 文件头

```
v1, 2147483647, 1
```

其中n代表Segment内部版本号，2147483647-2GB表示xxxxx.smoosh文件最大大小，1表示分片数量，即index.zip中XXXXX.smoosh文件数目。

(2) 文件体

Column | SmooshId | StartPos | EndPos
---|---|---|---
_time | 0 | 0 | 8000
Page | 0 | 110001 | 120000
_time | 0 | 8001 | 100000

第一列表示列式存储的各个列名，也就是各个维度；SmooshId表示所在的Smoosh文件的id，如文件头中所示，只有一个文件，所以都是0；StartPos表示该列在整个文件中的开始位置，EndPos表示结束位置，因为所有列的内容是序列化成二进制字节码进行保存的，所以知道了位置才能反序列化出各个列的内容。

