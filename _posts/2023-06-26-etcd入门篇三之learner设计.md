---
title: etcd入门篇二之learner设计
layout: post
header-img: "https://w.wallhaven.cc/full/ex/wallhaven-ex9gwo.png"
description: etcd系列第三篇博客
author: 陈家辉
tags:
- etcd
---

# 集群挑战

成员身份重置一直是最大的操作挑战之一，下面一起看看集群常见的挑战。

## 1 新成员导致Leader过载

一个新的etcd端点加入集群，由于没有任何数据，需要从leader获取更多的更新直到追上leader的日志。这就导致leader节点的网络更可能过载、阻塞或者失去和从节点的心跳。在这种情况下，flower可能会发起新的leader竞选。也就是说，拥有新成员的集群更容易受到领导人选举的影响。领导者选择和随后的更新传播到新成员都容易导致集群不可用的时期(参见*Figure 1*)。

![server-learner-figure-01](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-01.png)

## 2 网络分区

如果leader节点依旧保有绝大多数的节点链接，集群依然可以继续运行(参见 *Figure 2*)

![server-learner-figure-02](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-02.png)

### 2.1 leader孤立

当leader失去大多数的节点链接时，它将变为follower，这会影响集群的可用性(参见*Figure 3*)

![server-learner-figure-03](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-03.png)

### 2.2 集群分割为3+1

如果新增节点在leader的网络分区，leader依旧持有3票。不会发生重选举，不影响集群可用性。

![server-learner-figure-04](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-04.png)

### 2.3 集群分割为2+2

此时leader持有2票，不满足半数以上，将会发生重选举(参加*Figure 5*)。

![server-learner-figure-05](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-05.png)

### 2.4 丢失大多数

一个三节点集群，已经有一个follower网络分区，此时如果新增一个节点，此时quorum变为3，而新增节点并未就绪，leader丧失大多数，导致重选举。

![server-learner-figure-06](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-06.png)

因此在添加新节点时，可以先移除不健康节点，保证服务的可用性。

再向单节点集群添加新的节点时，会造成重选举。

![server-learner-figure-07](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-07.png)

## 3 集群错误配置

一个更糟糕的例子是加入的端点配置错误。成员重配置分为两步

1. `etcdctl member add`
2. 开始etcd服务对给出的url进行通信

如果此时url是错误的，并且出现了丢失大多数的情况，此时就无法恢复集群了。

![server-learner-figure-08](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-08.png)

多节点集群也会有这个问题

![server-learner-figure-09](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/server-learner-figure-09.png)

如上所述，一个简单的错误配置就可能使整个集群失效，进入不可操作的状态。在这种情况下，操作员需要使用 `etcd  --force-new-cluster` 标志手动重新创建集群。由于 etcd 已成为 Kubernetes 的关键任务服务，即使是最轻微的中断也可能对用户产生重大影响。我们怎样才能使这些操作更容易？其中，领导人选举对集群的可用性最为关键: 我们是否可以通过不改变法定人数来降低成员重组的破坏性？一个新节点是否可以空闲，只请求领导者的最小更新，直到它赶上？成员资格错误配置是否总是可逆的，并且能够以更安全的方式处理(错误的成员添加命令运行应该永远不会使集群失败) ？用户添加新成员时是否应该担心网络拓扑？无论节点的位置和正在进行的网络分区如何，成员可以添加 API 吗？

