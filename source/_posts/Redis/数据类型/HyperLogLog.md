---
title: HyperLogLog
categories: 
- Redis
- 数据类型
---

HyperLogLog并不是一种数据结构，而是一种算法，可以利用极小的内存空间完成独立总数的统计。

这个结构可以非常省内存的去统计各种计数，比如注册IP数、每日访问IP数。当然，存在误差！Redis官方给出的数字是0.81%的失误率。

> PFADD 添加一个元素，如果重复，只算作一个
> PFCOUNT 返回元素数量的近似值
> PFMERGE 将多个 HyperLogLog 合并为一个 HyperLogLog

**注意**：PFADD命令不支持key中含有":"，可以使用"_"进行分割

```
127.0.0.1:6379> PFADD login:20201005 123456
(error) WRONGTYPE Key is not a valid HyperLogLog string value.
127.0.0.1:6379> PFADD login_20201005 123456
(integer) 1
```

通过测试工程往HyperLogLog里PFADD了一亿个元素，通过rdb tools工具统计了这个key的信息

只需要14392 Bytes！也就是14KB的空间

bitmap存储一亿需要12M，而HyperLogLog只需要14K的空间

查了文档，发现HyperLogLog是一种概率性数据结构，在标准误差0.81%的前提下，能够统计`2^64`个数据

所以 HyperLogLog 适合在比如统计日活月活此类的对精度要不不高的场景

