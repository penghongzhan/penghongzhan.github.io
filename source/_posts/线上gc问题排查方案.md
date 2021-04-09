---
title: 线上gc问题排查方案
date: 2021-03-31 20:52:17
tags:
- jvm
- gc
- 线上问题
- fullgc
categories:
- gc
---

> 背景：线上es节点出现fullgc，导致整个查询不可用；因为es节点的堆内存非常大，通过dump的方式拿到的文件有几十个G，本身的dump过程也是非常慢的，所以需要采用一些及时的命令去分析出现fullgc的时候的堆内存中对象的情况，比如说jvm的命令、arthas等。

jvm分析命令：

- jmap -histo:live 3409 > a.txt
- jmap -heap 3409