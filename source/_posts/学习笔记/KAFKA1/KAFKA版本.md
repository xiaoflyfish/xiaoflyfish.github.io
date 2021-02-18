---
title: KAFKA版本
categories: 
- 学习笔记
- KAFKA1
---

# 几种Kafka

**Apache Kafka**

Apache Kafka 是最“正宗”的 Kafka，也应该是你最熟悉的发行版了。自 Kafka 开源伊始，它便在 Apache 基金会孵化并最终毕业成为顶级项目，它也被称为社区版 Kafka。更重要的是，它是后面其他所有发行版的基础。也就是说，后面提到的发行版要么是原封不动地继承了 Apache Kafka，要么是在此之上扩展了新功能，总之 Apache Kafka 是我们学习和使用 Kafka 的基础。

**Confluent Kafka**

Confluent 公司，它主要从事商业化 Kafka 工具开发，并在此基础上发布了 Confluent Kafka。Confluent Kafka 提供了一些 Apache Kafka 没有的高级特性，比如跨数据中心备份、Schema 注册中心以及集群监控工具等

**Cloudera/Hortonworks Kafka**

Cloudera 提供的 CDH 和 Hortonworks 提供的 HDP 是非常著名的大数据平台，里面集成了目前主流的大数据框架，能够帮助用户实现从分布式存储、集群调度、流处理到机器学习、实时数据库等全方位的数据处理。很多创业公司在搭建数据平台时首选就是这两个产品。不管是 CDH 还是 HDP 里面都集成了 Apache Kafka，因此我把这两款产品中的 Kafka 称为 CDH Kafka 和 HDP Kafka。

# 版本号

Kafka 目前总共演进了 7 个大版本，分别是 0.7、0.8、0.9、0.10、0.11、1.0 和 2.0

Kafka 从 0.7 时代演进到 0.8 之后正式引入了**副本机制**

0.8.2.0 版本社区引入了**新版本 Producer API**，即需要指定 Broker 地址的 Producer

0.9 大版本增加了基础的安全认证 / 权限功能，同时使用 Java 重写了新版本消费者 API，另外还引入了 Kafka Connect 组件用于实现高性能的数据抽取

0.10.0.0 是里程碑式的大版本，因为该版本**引入了 Kafka Streams**

0.11.0.0 版本，引入了两个重量级的功能变更：一个是提供幂等性 Producer API 以及事务（Transaction） API；另一个是对 Kafka 消息格式做了重构

1.0 和 2.0 版本，这两个大版本主要还是 Kafka Streams 的各种改进，在消息引擎方面并未引入太多的重大功能特性

