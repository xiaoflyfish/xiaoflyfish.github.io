---
title: Raft算法
categories: 
- 分布式算法
- Paxos算法
---

# Quorum

Quorum就是限定了一次需要读取至少`N+1-w`的副本数据，听起来有些抽象，举个例子，我们维护了10个副本，一次成功更新了三个，那么至少需要读取八个副本的数据，可以保证我们读到了最新的数据

N = 存储数据副本的数量

W = 更新成功所需的副本

R = 一次数据对象读取要访问的副本的数量

**Quorum的应用**

Quorum 机制无法保证强一致性，也就是无法实现任何时刻任何用户或节点都可以读到最近一次成功提交的副本数据。

Quorum 是分布式系统中常用的一种机制，用来保证数据冗余和最终一致性的投票算法，在 `Paxos`、Raft 和 ZooKeeper 的 `Zab` 等算法中，都可以看到 `Quorum` 机制的应用

**和Quorum机制对应的是WARO**

也就是Write All Read one，是一种简单的副本控制协议，当 `Client `请求向某副本写数据时（更新数据），只有当所有的副本都更新成功之后，这次写操作才算成功，否则视为失败。

WARO 优先保证读服务，因为所有的副本更新成功，才能视为更新成功，从而保证了所有的副本一致，这样的话，只需要读任何一个副本上的数据即可。

写服务的可用性较低，因为只要有一个副本更新失败，此次写操作就视为失败了。假设有 N 个副本，N－1 个都宕机了，剩下的那个副本仍能提供读服务；但是只要有一个副本宕机了，写服务就不会成功。

WARO 牺牲了更新服务的可用性，最大程度地增强了读服务的可用性，而` Quorum` 就是在更新服务和读服务之间进行的一个折衷。

# Paxos

**Paxos的节点角色**

在 Paxos 协议中，有三类节点角色，分别是 Proposer、`Acceptor` 和 Learner，另外还有一个 `Client`，作为产生议题者

**在工作实践中，一个节点可以同时充当这三类角色**

> Proposer提案者

Proposer 可以有多个，在流程开始时，Proposer 提出议案value

> Acceptor批准者

在集群中，Acceptor 有 N 个，Acceptor 之间完全对等独立，`Proposer` 提出的 value 必须获得超过半数`（N/2+1）`的 Acceptor 批准后才能通过。

> Learner学习者

Learner 不参与选举，而是学习被批准的 value，在`Paxos`中，Learner主要参与相关的状态机同步流程。

这里Leaner的流程就参考了Quorum 议会机制，某个 value 需要获得 `W=N/2 + 1` 的 Acceptor 批准，Learner 需要至少读取 `N/2+1` 个 Accpetor，最多读取 N 个 Acceptor 的结果后，才能学习到一个通过的 value。

> Client 产生议题者

Client 角色，作为产生议题者，实际不参与选举过程，比如发起修改请求的来源等。

**Proposer与Acceptor之间的交互**

Paxos 中， Proposer 和 Acceptor 是算法核心角色，Paxos 描述的就是在一个由多个 Proposer 和多个 Acceptor 构成的系统中，如何让多个 Acceptor 针对 Proposer 提出的多种提案达成一致的过程，而 Learner 只是学习最终被批准的提案。