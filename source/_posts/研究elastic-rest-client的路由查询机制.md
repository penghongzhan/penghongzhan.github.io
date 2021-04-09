---
title: 研究elastic rest client的路由查询机制
date: 2021-04-09 18:18:23
tags:
- elasticsearch
- elastic
- client
- sniff
categories:
- elasticsearch
---

# 黑名单机制

- 每次查询轮询每个节点，当某个节点查询失败之后，会被加入黑名单中
- 某个节点查询成功之后，会将该节点从黑名单中移除
- 每次重新嗅探之后，会清空黑名单

也就是说，黑名单是针对那些能嗅探到，但是访问失败的节点

```java
// 查询的时候，会将restclient中的节点，踢除黑名单中的节点，踢除的时候还有个判断，某个黑名单中的节点，在黑名单中不足一分钟
// 也就是说，没过一分钟，都会尝试一下查询黑名单中的节点，如果能查到，从黑名单中踢除，如果查询不到，还在黑名单中
Set<HttpHost> filteredHosts = new HashSet<>(hostTuple.hosts);
for (Map.Entry<HttpHost, DeadHostState> entry : blacklist.entrySet()) {
    if (System.nanoTime() - entry.getValue().getDeadUntilNanos() < 0) {
        filteredHosts.remove(entry.getKey());
    }
}
```

# 自动嗅探机制

每隔一段时间去查询一次目前es中可用的节点，然后重新设置restclient的节点

使用的嗅探api：`curles 'http://esprod:8080/_nodes/http?timeout=1000ms'`

这个api有个牛逼的地方，超时时间参数

- 请求会发送到url中配置的节点
- 该节点会去获取集群中其他节点的状态，应该会去请求其他节点
- 如果请求其他节点的时间超过了1s或者直接失败了，会直接返回获取该节点失败
- 相比于cat api获取节点的状态，该api不会一直阻塞住，cat api要么全部获取成功，要么全部获取失败，不会返回哪个节点成功，哪个节点失败

# 自动化节点路由

我们之前采用手动的方式，去删除有问题的节点，因为出问题的时候，cat api根本无法查询到集群中可用的节点，所以无法自动化去踢除，只能收到告警之后，手动的去删除某个节点。

利用上面sniff的嗅探机制，我们可以通过上面的_nodes api，获取集群中没有问题的节点，这时候，查询的reference中可以直接将这个节点踢除；

而且我们可以利用这个机制，去设置es的告警，只要_nodes api获取某个节点的信息失败，那么可以直接告警。

## 需要注意的问题

同机房的机器，只能踢除其中一台机器，如果两台机器同时出现问题，不能直接从reference中把两台机器都踢除了，因为这样会直接导致查询失败。所以这个自动化修改reference的方案需要慎重！
