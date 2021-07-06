---
title: jvm相关学习笔记
date: 2021-06-09 19:42:14
categories:
- jvm
tags:
- java
- jvm
- gc
---

# 手动触发full gc

使用jmap -histo[:live] pid查看堆内存中的对象数目、大小统计直方图。如果带上live则只统计活对象，jvm进行Full GC 后，进行统计。

jmap -dump:format=b,file=文件名 [pid]
导出Dump文件，可以用相关工具(jhat/jvisualvm/mat)进行分析。-dump:live会触发Full GC后转存Dump。
需要注意的是，上面的导出快照命令，在1G左右JVM内存的情况下，要大概等待1分钟左右的时间，且执行时会使JVM暂停执行，因此不要在正式运行系统的高峰期或关键时刻使用。

> -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5

CMS收集器提供了一个-XX:+UseCMSCompactAtFullCollection开关参数（默认是开启的，此参数从JDK 9开始废弃），用于在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程，由于这个内存整理必须移动存活对象，（在Shenandoah和ZGC出现前）是无法并发的。这样空间碎片问题是解决了，但停顿时间又会变长，因此虚拟机设计者们还提供了另外一个参数-XX:CMSFullGCsBeforeCompaction（此参数从JDK 9开始废弃），这个参数的作用是要求CMS收集器在执行过若干次（数量由参数值决定）不整理空间的Full GC之后，下一次进入Full GC前会先进行碎片整理（默认值为0，表示每次进入Full GC时都进行碎片整理）。
