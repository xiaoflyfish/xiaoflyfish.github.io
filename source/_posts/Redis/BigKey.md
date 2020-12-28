---
title: BigKey
categories: 
- Redis
---

bigkey是指key对应的value所占的内存空间比较大

如果按照数据结构来细分的话，一般分为字符串类型和非字符串类型：

- 字符串类型：体现在单个value值很大，一般认为超过10KB就是bigkey
- 非字符串类型：哈希、列表、集合、有序集合，体现在元素个数过多

**bigkey的危害体现在三个方面：**

- 内存空间不均匀：例如在Redis Cluster中，bigkey会造成节点的内存空间使用不均匀
- 超时阻塞：由于Redis单线程的特性，操作bigkey比较耗时，也就意味着阻塞Redis可能性增大
- 网络拥塞：每次获取bigkey产生的网络流量较大，假设一个bigkey为1MB，每秒访问量为1000，那么每秒产生1000MB的流量，对于普通的千兆网卡（按照字节算是128MB/s）的服务器来说简直是灭顶之灾

**bigkey的发现**

```
redis-cli --bigkeys
```

`redis-cli--bigkeys`可以命令统计bigkey的分布

**debug object**

判断一个key是否为bigkey，只需要执行`debug object key`查看serializedlength属性即可，它表示key对应的value序列化之后的字节数

**如何优雅地删除bigkey**

对于string类型使用del命令一般不会产生阻塞

hash、list、set、sorted set类型的删除可以使用sscan、hscan、zscan命令