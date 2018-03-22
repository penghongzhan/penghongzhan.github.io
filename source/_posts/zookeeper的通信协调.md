---
title: zookeeper的通信协调
date: 2018-03-18 15:47:22
categories:
- zookeeper
tags:
- zookeeper
- session
---

---
# observer服务器
---

observer不需要将事务持久化到磁盘，一旦observer被重启，需要从leader重新同步整个名字空间。

依据Observer的特点。我们能够使用Observer做跨数据中心部署。假设把Leader和Follower分散到多个数据中心的话，由于数据中心之间的网络的延迟。势必会导致集群性能的大幅度下降。使用Observer的话，将Observer跨机房部署。而Leader和Follower部署在单独的数据中心，这样更新操作会在同一个数据中心来处理，并将数据发送的其它数据中心（包括Observer的），然后Client就能够在其它数据中心查询数据了。可是使用了Observer并不是就能全然消除数据中心之间的延迟，由于Observer还得接收Leader的同步结果合Observer有更新请求也必须转发到Leader，所以在网络延迟非常大的情况下还是会有影响的，它的优势就为了本地读请求的高速响应。

---
# 服务器之间的通信协调
---

消息类型：数据同步型，服务器初始化型，请求处理型，会话管理型。

## 数据同步型

leader和learner之间同步的数据类型。

zookeeper的集群角色有两种：leader和learner，learner对应的服务器有两种角色，包括follower和observer，所以服务器的角色有三种，leader, follower, observer。learner需要不断跟leader同步状态，包括同步最新的ZXID和epoch。

## 服务器初始化型

一般发生在集群或者新的机器初始化的时候，leader和learner之间同步状态的时候发送的类型。

## 请求处理型

发生在集群接收client请求的过程中。集群中的服务器都可以接收客户端的请求，但是事务请求需要转发给leader服务器进行执行。

follower服务器接受到客户端的事务请求之后，会通过request请求发送给leader服务器，leader 会根据事务请求创建投票，以proposal信号发送给集群中得 follower服务器。follower服务器会进行事务日志的记录，记录成功之后会向leader服务器响应ack信号，一旦过半以上的服务器响应之后，leader会向follower服务器发送commit信号，表示可以进行事务日志的提交。对于follower服务器来说，由于之前发送过来的proposal信号中携带了完整的事务请求，follower本地也记录了事务日志，所以leader发送给follower的commit信号只是一个简单的触发信号，不再携带事务内容，但是对于observer服务器来说，由于不存在事务投票的过程，所以observer本身没有记录事务内容，所以leader服务器通过另一个信号，叫inform信号，携带事务内容给observer，进行事务的同步。

## zookeeper的事务请求头说明：

- clientID：用来唯一标识一个客户端，应该是zookeeper生成的sessionID。
- cxid：客户端操作序列号。
- zxid：该事务请求对应的事务zxid。
- time：服务器开始处理该事务请求的时间。
- typ：事务请求的类型，例如创建、删除等

事务请求头封装完成之后，会发送完整的事务请求到follower服务器进行事务操作，包括sync, proposal, commit。

- sync：针对事务请求进行事务日志的记录，完成之后像leader服务器发送ack信号。
- proposal：每一个事务请求都需要集群中过半的服务器投票通过才能更新到zk的内存数据库中。zk将事务封装成proposal请求发送给follower服务器，经过上一步的sync操作之后，leader服务器会收到follower返回的ack信号，一旦过半的follower返回成功，会进行后续的commit操作。
- commit：对事务日志记录的事务进行提交，更新到内存数据库中。

## 会话管理型

该消息是zk跟客户端保持会话用到的消息，客户端过来的请求不一定会发送到leader服务器，很多会发送到follower服务器，如果不是事务操作，follower可以直接响应，所以有一些客户端的会话是通过follower服务器上的learner来管理的，leader要想管理所有的会话，就需要通过会话管理消息，实际上是一种ping操作，来获取learner维护的客户端会话。从而通过leader服务器统一对连接到zookeeper的客户端进行会话激活。

---
# zookeeper与客户端的会话管理
---

zk启动的时候，会初始化一个起始的sessionid，后续客户端连到zk创建会话的时候，会基于这个其实的sessionid递增。

## sessionid

**起始的sessionid生成方案：**

sessionid的初始值会根据当前server的id和创建时的时间戳来生成。

```
sid_ = (System.currentTimeMillis() << 24 ) >>> 8;
sid = sid_ | (serverid << 56);
```

## ticktime

为了高效低耗的实现会话的超时检查和清理，zookeeper为每个会话标记下一次会话超时时间点，zookeeper会定时在这个时间点检查会话是否超时。

假设在时间点T1客户端跟zk创建了会话，同时会携带一个client定义的超时时间T2，那么简单的讲，可以在T1+T2时间点上检查该会话是否超时，但是通过这种策略无法实现分桶的方案。因此zookeeper的下次检查的时间为：

```
((T1 + T2) / ticktime + 1) * ticktime
```

所以下次的检查时间一定是ticktime的倍数，也就是基于这样的倍数，倍数的时间点上会有一个桶的概念，相同时间点的会话会放在这个桶中。

定时任务会定期检查某个时间点的桶中的会话，如果已经超时，那么从桶中移除。

## 会话的续期

客户端每次发送请求给zk的时候，例如创建、移除等操作，都会自动给会话续期，针对会话续期，zk的操作就是将该会话从上一个桶移到新的桶中。除了请求zk的时候会自动续期会话，客户端本身会在他的会话超时之前向zk发送ping保持会话的有效性，俗称为“心跳检测”。

## 会话的清理

根据分桶的策略，客户端的会话都被放在指定的桶中，如果临近的桶中的会话在清理桶的定时任务执行之前，没有进行会话续期，那么定时清理任务就会将遗留在该桶中的会话清理掉。因为根据上面会话续期的说明中已经知道，续期的会话会从当前桶移动到新的桶中，所以遗留在桶中的会话都是没有续期的，需要被移除。

