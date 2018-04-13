---
title: Flink的checkpoint学习笔记
date: 2018-04-13 13:33:26
tags:
- flink
- checkpoint
categories:
- flink
---

学习[官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.4/internals/stream_checkpointing.html)

数据流中消费到barrier n（第n个快照对应的barrier）的时候，flink会做一次checkpoin，生成一个snapshot。当出现故障之后，集群进入恢复阶段，会从最近的snapshot恢复出最新的快照。从当前快照的执行进度继续执行。

**快照中需要记录消费的source的位置信息Sn（第n次快照对应的source的位置）**

例如消费Kafka的数据，消费了3条记录到flink内存中，还未进行sink，此时flink崩溃了，内存中3条数据会丢失，kafka的offset也增加了3。flink恢复了之后，从snapshot重新开始执行，继续消费kafka，此时不能从kafka现在的offset开始消费，需要退回3条记录之前的offset开始消费。

此时需要flink消费kafka的程序能够记录kafka的offset，进行snapshot的时候，需要记录当前消费的kafka的offset，故障恢复之后，需要从snapshot记录的offset位置重新消费kafka。operator在处理barrier n的时候，会将barrier n中记录的source的位置信息Sn发送给协调者（Flink的JobManager）进行记录。

Flink的快照机制受分布式快照的标准Chandy-Lamport算法的启发，并专门为Flink的执行模型量身定制。由于数据会分布到并发的流中执行，所以在某个时刻可能出现对应不同snapshot的barrier n，这就是针对并发需要应对的场景。

---
# barrier在流中的传输
---

![barrier在流中的传输](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/stream_aligning.svg)

现在有一个operator，输入是两个stream，输出是一个stream，输入的两个stream中都有各自的barrier n，两个barrier n会随着流进入operator，operator接收到两个barrier n之后，才会将barrier n发送到输出stream。现在输出只有一个stream，如果有多个stream，operator在向后发送的时候，会给每个输出stream都发送barrier n。

假设现在消费一个kafka的分区，该分区的并发度为3，消费到offset为100的时候触发了一次snapshot，此时会向三个stream中写入相同的barrier n，barrier n对应的Sn为100。

---
# 包含barrier的数据流的处理规则
---

- 对于某一个stream来说，operator一旦接收到该stream的barrier n，该stream的barrier n后续的记录将不会被处理，直到接收到所有stream中的Barrie n；
- 最后一个barrier n到来之前，其他的stream的barrier n之后的记录会放在缓冲区中，不会被处理；
- 一旦接收到了所有stream的barrier n，operator会进行该次快照，快照完成之后operator会向后发送barrier n；
- barrier n发送出去之后，继续消费stream中的记录，会优先处理之前放在缓冲区中的记录。

---
# 快照中的状态
---

operator进行快照的时候，会将其中的状态也保存到快照中，状态的形式有：用户自定义状态和系统状态。

operator接收到最后一个barrier n的时候，表明已经接收到barrier n对应的所有数据，这些数据的状态已经更新完成，在发射barrier n之前，会将这批数据已经更新完的状态保存到snapshot中。由于状态信息可能很大，所以可以配置到HDFS等可靠的存储中，默认是放在jobManager的内存中。

![快照中的状态](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/checkpointing.svg)

如果放在外部存储中，实际的内存中存放的快照信息包含：

- 数据源：快照创建时数据流中的位置偏移
- operator：快照在外部存储中的位置

状态存放到存储中之后，operator快照完成，将barrier向后发送。一旦后续的sink操作完成后，会向协调者发送确认信号，表明该批次的快照数据已经处理完成。

---
# 快照带来的问题
---

这种快照方案需要阻塞等待所有的barrier n，所以可能会带来数据流的延时。对于实时性要求毫秒以内的应用，Flink支持关闭barrier对齐，也就是说operator接收到第一个barrier n之后就会开始进行快照，barrier n对应的快照中会存在n+1中的数据，在进行快照恢复的时候，这些数据会被重复处理。

正常阻塞barrier n的快照过程，需要将快照持久化到外部存储之上，这个过程通常是比较耗时的，为了加快快照速度，可以将该过程设置成异步的。原来是等到快照持久化完成之后才会向后发送barrier，现在生成一个快照状态之后，启动异步任务进行持久化（等到持久化完成之后会通知coordinate——JobManager），并立即向后发送barrier，所以等到sink操作确认状态之后（可能快照还没有存储完成），需要最终等到持久化完成才表示该快照完成。

快照还提供了可以被取消的操作，用于释放流和其他资源。
