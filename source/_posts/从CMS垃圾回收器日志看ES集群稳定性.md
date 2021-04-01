---
title: 从CMS垃圾回收器日志看ES集群稳定性
date: 2021-03-02 23:00:17
tags:
- cms
- jvm
- gc
- elasticsearch
- elastic
- es
- 线上稳定性
categories:
- elasticsearch
---

# es堆内存配置

- -Xms30g -Xmx30g
- -XX:CMSInitiatingOccupancyFraction=75
- 查看具体的堆内存的分配大小
  - `/usr/local/java8/bin/jstat -gccapacity 17394  1000`
  - survivor: 629120.0，629M
  - eden: 5033216.0, 5G
  - old: 25165824.0, 25G

# cms正常的并发gc

CMS通过将大量工作分散到并发处理阶段来在减少STW时间，在这块做得非常优秀。

几个关键的时间日志：

- 0.138/0.138 secs：这个阶段的持续时间与时钟时间；
- user：GC线程在垃圾收集期间所使用的CPU总时间；
- sys：系统调用或者等待系统事件花费的时间；
- real：应用被暂停的时钟时间，由于GC线程是多线程的，导致了real小于(user+real)，如果是gc线程是单线程的话，real是接近于(user+real)时间。

下面个是一段ParNew和CMS结合的垃圾回收日志：

```
# 年轻代gc之后，对象分配失败
2021-03-02T07:44:58.434+0800: 807042.036: [GC (Allocation Failure) 807042.036: [ParNew
# survivor期望放下的对象大小是322M 按照上面es堆内存的配置，survivor的大小应该是600M+，这里为什么能发下的大小只有一般，Desired survivor size是Survivor区大小的50%(TargetSurvivorRatio的默认值)。
Desired survivor size 322109440 bytes, new threshold 1 (max 6)
# 年龄是1的对象的占用空间大小640M，可以发现，年龄为1的对象大小已经超过了survivor空间的大小，所以需要将对象升级到老年代
- age   1:  640934736 bytes,  640934736 total
# 年轻代：回收前->回收后(年轻代总的空间) 回收时间；整个堆：回收前->回收后(总空间) 回收时间 
: 5662336K->629120K(5662336K), 0.1170999 secs] 23497082K->19192581K(30828160K), 0.1175795 secs] [Times: user=2.19 sys=0.03, real=0.12 secs]
# 经过上面的parnew回收之后，年轻代共5G空间，占用629M，老年代共25G空间，占用20G，到达了cms的回收阈值75%
# stw 初始标记阶段
2021-03-02T07:45:02.448+0800: 807046.051: [GC (CMS Initial Mark) [1 CMS-initial-mark: 19230527K(25165824K)] 19860050K(30828160K), 0.0077555 secs] [Times: user=0.08 sys=0.01, real=0.01 secs]
# no stw 并发标记阶段
2021-03-02T07:45:02.457+0800: 807046.059: [CMS-concurrent-mark-start]
2021-03-02T07:45:04.725+0800: 807048.327: [CMS-concurrent-mark: 2.269/2.269 secs] [Times: user=25.46 sys=0.81, real=2.27 secs]
# no stw 并发preclean阶段
2021-03-02T07:45:04.725+0800: 807048.328: [CMS-concurrent-preclean-start]
2021-03-02T07:45:04.896+0800: 807048.498: [CMS-concurrent-preclean: 0.162/0.170 secs] [Times: user=0.44 sys=0.02, real=0.17 secs]
# no stw 并发的abortable preclean阶段
2021-03-02T07:45:04.896+0800: 807048.498: [CMS-concurrent-abortable-preclean-start]
 CMS: abort preclean due to time 2021-03-02T07:45:10.286+0800: 807053.888: [CMS-concurrent-abortable-preclean: 5.196/5.390 secs] [Times: user=23.80 sys=0.94, real=5.39 secs]
# stw 最终标记阶段
2021-03-02T07:45:50.010+0800: 807093.612: [GC (CMS Final Remark) [YG occupancy: 2388409 K (5662336 K)]807093.612: [Rescan (parallel) , 0.0570763 secs]807093.669: [weak refs processing, 0.0002759 secs]807093.669: [class unloading, 0.0418367 secs]807093.711: [scrub symbol table, 0.0110873 secs]807093.722: [scrub string table, 0.0018460 secs][1 CMS-remark: 19475169K(25165824K)] 21863578K(30828160K), 0.1166112 secs] [Times: user=1.35 sys=0.00, real=0.11 secs]
# no stw 并发清除阶段
2021-03-02T07:45:50.127+0800: 807093.729: [CMS-concurrent-sweep-start]
2021-03-02T07:45:53.535+0800: 807097.137: [CMS-concurrent-sweep: 3.270/3.408 secs] [Times: user=35.67 sys=1.43, real=3.41 secs]
# no stw 并发reset阶段
2021-03-02T07:45:53.535+0800: 807097.137: [CMS-concurrent-reset-start]
2021-03-02T07:45:53.601+0800: 807097.204: [CMS-concurrent-reset: 0.066/0.066 secs] [Times: user=0.35 sys=0.03, real=0.07 secs]
```

上面的垃圾回收日志是正常的一个回收过程，年轻代放不下之后，会将对象升级到老年代，老年代到达cms设置的阈值之后，会触发老年代的回收，上述回收过程正是cms的优势所在，一次gc过程，大部分时间是跟应用线程并发执行，对应用线程的影响比较小，对于一般的应用基本上也是感知不到的。

但是有一种情况下的gc会对应用造成很大的影响，cms本身有一个比较大的缺陷在于，无法处理浮动垃圾，单纯的标记清除算法，随着垃圾回收的进行，老年代的碎片也会越来越多，一次old gc的时候，如果新一波的young gc之后升级到老年代的对象，在目前老年代中仍然放不下，会产生concurrent mode failure，此时会造成单线程的fullgc，应用线程会被暂停，如果堆内存比较大，就像我们的es机器，堆内存30G，一次fullgc需要花费20多秒的时间，对线上的应用的影响会非常大。

# concurrent mode failure

CMS 收集器无法处理浮动垃圾（Floating Garbage），可能出现 “Concurrnet Mode Failure” 失败而导致另一次Full GC的产生，可能引发串行Full GC；下面是一段产生该异常的gc日志，gc造成es正常的线程停顿了26s的时间：

```
2021-02-23T07:01:47.432+0800: 199651.034: [GC (CMS Initial Mark) [1 CMS-initial-mark: 19184059K(25165824K)] 19829584K(30828160K), 0.0135926 secs] [Times: user=0.10 sys=0.00, real=0.01 secs]
2021-02-23T07:01:47.446+0800: 199651.048: Total time for which application threads were stopped: 0.0187626 seconds, Stopping threads took: 0.0026169 seconds
2021-02-23T07:01:47.446+0800: 199651.048: [CMS-concurrent-mark-start]
2021-02-23T07:01:49.548+0800: 199653.151: [CMS-concurrent-mark: 2.102/2.102 secs] [Times: user=24.28 sys=0.90, real=2.11 secs]
2021-02-23T07:01:49.548+0800: 199653.151: [CMS-concurrent-preclean-start]
2021-02-23T07:01:49.728+0800: 199653.330: [CMS-concurrent-preclean: 0.169/0.179 secs] [Times: user=0.35 sys=0.02, real=0.17 secs]
2021-02-23T07:01:49.728+0800: 199653.330: [CMS-concurrent-abortable-preclean-start]
2021-02-23T07:01:51.246+0800: 199654.848: Total time for which application threads were stopped: 0.0029138 seconds, Stopping threads took: 0.0004014 seconds
2021-02-23T07:01:51.247+0800: 199654.849: Total time for which application threads were stopped: 0.0012148 seconds, Stopping threads took: 0.0000792 seconds
2021-02-23T07:01:52.643+0800: 199656.245: Total time for which application threads were stopped: 0.0033441 seconds, Stopping threads took: 0.0003442 seconds
2021-02-23T07:01:52.645+0800: 199656.247: Total time for which application threads were stopped: 0.0019024 seconds, Stopping threads took: 0.0002444 seconds
2021-02-23T07:01:54.746+0800: 199658.348: [GC (Allocation Failure) 199658.348: [ParNew
Desired survivor size 322109440 bytes, new threshold 1 (max 6)
- age   1:  476010080 bytes,  476010080 total
: 5650054K->558251K(5662336K), 0.1145601 secs] 24834113K->20039782K(30828160K), 0.1148260 secs] [Times: user=1.48 sys=0.03, real=0.12 secs]
2021-02-23T07:01:54.861+0800: 199658.463: Total time for which application threads were stopped: 0.1188860 seconds, Stopping threads took: 0.0014423 seconds
2021-02-23T07:01:55.211+0800: 199658.813: Total time for which application threads were stopped: 0.0041470 seconds, Stopping threads took: 0.0018983 seconds
 CMS: abort preclean due to time 2021-02-23T07:01:55.372+0800: 199658.974: [CMS-concurrent-abortable-preclean: 5.430/5.644 secs] [Times: user=36.94 sys=2.87, real=5.65 secs]
2021-02-23T07:01:55.375+0800: 199658.977: [GC (CMS Final Remark) [YG occupancy: 1938421 K (5662336 K)]199658.977: [Rescan (parallel) , 0.0611395 secs]199659.038: [weak refs processing, 0.0003288 secs]199659.039: [class unloading, 0.0382319 secs]199659.077: [scrub symbol t
able, 0.0107094 secs]199659.088: [scrub string table, 0.0015974 secs][1 CMS-remark: 19481531K(25165824K)] 21419952K(30828160K), 0.1160335 secs] [Times: user=1.42 sys=0.12, real=0.12 secs]
2021-02-23T07:01:55.491+0800: 199659.093: Total time for which application threads were stopped: 0.1192748 seconds, Stopping threads took: 0.0005368 seconds
2021-02-23T07:01:55.491+0800: 199659.094: [CMS-concurrent-sweep-start]
2021-02-23T07:01:55.640+0800: 199659.242: Total time for which application threads were stopped: 0.0029760 seconds, Stopping threads took: 0.0005614 seconds
2021-02-23T07:01:55.642+0800: 199659.244: Total time for which application threads were stopped: 0.0015641 seconds, Stopping threads took: 0.0002054 seconds
2021-02-23T07:01:56.638+0800: 199660.241: Total time for which application threads were stopped: 0.0036077 seconds, Stopping threads took: 0.0012128 seconds
2021-02-23T07:01:56.675+0800: 199660.278: [GC (Allocation Failure) 199660.278: [ParNew (promotion failed): 5590537K->5661406K(5662336K), 1.3361918 secs] 24852053K->25503147K(30828160K), 1.3365327 secs] [Times: user=3.56 sys=0.02, real=1.34 secs]
2021-02-23T07:01:58.012+0800: 199661.614: [Full GC (Allocation Failure) 199661.614: [CMS2021-02-23T07:02:01.539+0800: 199665.142: [CMS-concurrent-sweep: 4.686/6.048 secs] [Times: user=27.51 sys=2.27, real=6.04 secs]
 (concurrent mode failure): 19841741K->12822928K(25165824K), 26.3160634 secs] 25503147K->12822928K(30828160K), [Metaspace: 85227K->85227K(1130496K)], 26.3188523 secs] [Times: user=26.33 sys=0.00, real=26.31 secs]
2021-02-23T07:02:24.331+0800: 199687.934: Total time for which application threads were stopped: 27.6682172 seconds, Stopping threads took: 0.0098307 seconds
```

# gc日志的设置

这个gc日志不是es特有的设置，而是jvm本身的设置

```
-Xloggc:/opt/meituan/es_logs/elasticsearch.gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=15
-XX:GCLogFileSize=20M
```

之前遇到fullgc的问题，为了避免对线上应用的影响，第一时间重启了es进程，由于es进程配置的gc日志只保存在一个文件中，重启之后日志被重写，我们无法去分析当时的一个gc日志情况；这里开启了日志滚动的开关，生成的日志的形式如下：

```
elasticsearch.gc.log
elasticsearch.gc.log.0
elasticsearch.gc.log.1
elasticsearch.gc.log.2.current
```

- 上面表示，当前正在写的文件的编号是2，假如文件的个数是3个，那么编号2的写完了之后，会继续写后缀为0的文件，就是这种滚动写日志文件的形式；
- 每次程序启动的时候，都会从编号0开始，重新创建后缀0的文件
  - 假设目前程序生成了文件0、1、2，重启之后，会重新写文件0，因此文件2中的内容还保存，也就是重启之前的gc的内容还可以去查看，分析重启之前的gc现场
  - 存在一个问题，假设日志文件比较少，日志还没有滚动，只有文件0，这时候，重启了程序，文件0会重新创建，这时候之前的日志内容就没了

# es线上稳定性的思考

事前、事中、事后

- 事前：监控、慢查询梳理
- 事中：踢除有问题节点，保证线上查询正常；重启节点，需要注意的是保存好事故现场
  - 直接杀掉有问题的节点，缺点无法保留问题现场
- 事后：分析问题出现的原因，解决问题
  - 优化索引结构，优化慢查询
  - 开发小工具，保证问题出现的时候，线上服务影响程度最小化

当然这不是一个一次性的工作，需要针对问题不断去优化事前、事中、事后的处理逻辑

- 比如说：监控是依赖es node api的，node api需要获取每个节点的状态，当一个节点full gc之后，整个api不返回结果了，监控无效了
  - 通过线程池api，提前发现问题
- 获取节点的工具化页面，在full gc之后也无法获取到，通过cellar缓存解决
- gc log有问题，只保存在一个文件中，每次重启文件被重写，无法分析当时问题现场的gc日志

### 参考文章

- [一次GC Tuning小记](https://neway6655.github.io/java,%20gc%20tuning/2016/09/24/gc-tuning.html)
- [JVM 之 ParNew 和 CMS 日志分析](https://matt33.com/2018/07/28/jvm-cms/)
- [JVM 配置GC日志](https://lihuimintu.github.io/2019/02/19/gcLog/)
