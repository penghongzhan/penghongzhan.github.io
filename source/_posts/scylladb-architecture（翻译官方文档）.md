---
title: scylla architecture（翻译官方文档）
date: 2018-05-09 22:46:24
tags:
- scylladb
categories:
- scylladb
---

[原文链接](https://www.scylladb.com/product/technology/)

---
# 架构对比
---

Scylla是NoSQL数据存储设计的新方法，针对现代化硬件进行了优化

**传统堆栈**

- 锁竞争
- 缓存竞争
- NUMA（非统一内存访问）不友好

**seastar的共享堆栈**

- 没有竞争
- 线性扩展
- NUMA友好

**架构对比：**![架构对比](https://www.scylladb.com/wp-content/uploads/network-diagram.png)

传统的nosql数据存储包括一个建立在linux操作系统之上的JVM，利用页面缓存，并使用复杂的内存分配策略来“欺骗”JVM垃圾回收器，以避免内存回收过程中的`stop-the-world pauses`。由于低处理器利用率，这样的设计会遭受突然延迟卡顿，昂贵的锁竞争和低吞吐量。

Scylla设计基于现代化的无共享方法。Scylla运行多个引擎，每个核心对应一个引擎，每个引擎具有自己的内存，CPU和多队列NIC。我们可以在一台商业服务器上轻松实现一百万个CQL操作。另外，Scylla针对插入、删除和读取的操作，实现一致的低延迟，延时控制在一毫秒以内。

---
# 效果
---

性能改进不仅可以减少硬件资源，还可以简化架构，调试和DevOps人工的开发流程。Scylla为您提供了构建适合应用程序的数据模型的能力，而不是扭曲数据模型以适应您的NoSQL系统。

复杂性减慢了开发时间。 Scylla的高效设计意味着架构师可以消除额外的表，外部缓存和复杂的操作系统级配置。

---
# 实现
---

现代化的工作负载运行的硬件不同于编程范例所依赖的硬件以及当前软件基础架构所设计的硬件。

scylla使用seastar架构实现了多核心硬件上的性能提升。

## 核心数量增长；时钟速度保持不变

伴随着最新的时钟速度的出现，CPU制造商无法再依赖加快时钟速度来提升CPU的性能。相反，现代芯片通过增加核心数来提升性能。随着这一变化，真实世界的应用程序性能在单个内核上的效率和吞吐量变得越来越低，而更多的则是关于软件如何在多个内核之间进行协调。

在新硬件上，标准工作负载的性能取决于核心之间的锁和协调，而不是单个核心的性能。软件架构师面临着两个没有吸引力的选择：

- 粗粒度锁：这将面临应用程序线程争夺数据的控制权，并等待而不是产生有用的工作
- 细粒度锁：除了难以编程和调试之外，由于锁定原语本身的原因，即使在没有争用发生时也会发现很大的开销。

## 同时，I/O速度继续增加

现代系统上的网络和存储设备的速度也在不断增长。但是，尽管处理器获得了更多的内核，但在任何一个内核上处理数据包的能力都没有增长。

一个的2GHz处理器在10GBps网络上以线速处理1024字节数据包，每个数据包只有1670个时钟周期。（source: [Intel DPDK Overview](http://www.intel.com/content/dam/www/public/us/en/documents/presentation/dpdk-packet-processing-ia-overview-presentation.pdf)）

当每个数据包有许多时钟周期可用时，有效和安全的软件设计方法已不再可持续。

---
# seastar模式：无共享
---

因为跨内核共享信息需要高成本的锁，所以Scylla使用无共享模式，将所有请求分散到独立的内核中。

Scylla每个内核运行应用程序的一个线程，并且依赖于显式消息传递，而不是线程之间的共享内存。此设计可避免缓慢，不可缩放的锁定原语和缓存反弹。

任何跨核心的资源共享都必须被明确处理。例如，当两个请求是同一个会话的一部分，并且每个CPU都分别获得了依赖于相同会话状态的请求时，一个CPU必须明确地将请求转发给另一个。每个CPU可能处理任何的响应（不太懂什么意思）。Seastar框架提供了限制跨内核通信需求的功能，但是当通信不可避免时，它会提供高性能的非阻塞通信原语以确保性能不会降低。

无锁的核心间通信线程化的应用程序需要固有的昂贵的锁操作，而Seastar模型完全避免了跨CPU通信的锁。

从程序员的角度来看，Seastar使用预期，承诺和延续（f/p/c）。在使用epoll和libevent等用户空间库进行常规事件驱动编程使编写复杂应用程序变得非常困难的情况下，f/p/c使编写复杂的异步代码变得更加容易。

例如，发送者核心C0和接收者核心C1之间的以下交互可以在不需要锁的情况下进行。

```
# 看不懂
C0: sender -> wait for queue entry (usually immediate) -> enqueue request, allocate promise.
C1: dequeue request; execute it -> move result to request object -> enqueue request on response queue
C0: dequeue request; extract response, use it to fulfill promise; destroy request
Each actual queue, one for requests and a return queue for fulfilled requests, is a simple queue of pointers.
```

系统上每对CPU核心有一个请求队列和一个返回队列。由于核心不与自身配对，16核心系统将有240个请求队列和240个返回队列。

其结果是一个独特的NoSQL数据存储，可以以最小的开销扩展到每个节点的大量内核。