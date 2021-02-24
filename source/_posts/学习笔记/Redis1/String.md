---
title: String
categories: 
- 学习笔记
- Redis1
---

**为什么String 类型内存开销大**

String 类型结构：

- 当你保存 64 位有符号整数时，String 类型会把它保存为一个 8 字节的 Long 类型整数，这种保存方式通常也叫作 int 编码方式。
- 当你保存的数据中包含字符时，String 类型就会用简单动态字符串（SDS）结构体来保存

对于 String 类型来说，除了 SDS 的额外开销，还有一个来自于 RedisObject 结构体的开销

因为 Redis 的数据类型有很多，而且，不同数据类型都有些相同的元数据要记录（比如最后一次访问的时间、被引用的次数等），所以，Redis 会用一个 RedisObject 结构体来统一记录这些元数据，同时指向实际数据。

一个 RedisObject 包含了 8 字节的元数据和一个 8 字节指针，这个指针再进一步指向具体数据类型的实际数据所在，例如指向 String 类型的 SDS 结构所在的内存地址

一方面，当保存的是 Long 类型整数时，RedisObject 中的指针就直接赋值为整数数据了，这样就不用额外的指针再指向整数了，节省了指针的空间开销。

另一方面，当保存的是字符串数据，并且字符串小于等于 44 字节时，RedisObject 中的元数据、指针和 SDS 是一块连续的内存区域，这样就可以避免内存碎片。这种布局方式也被称为 embstr 编码方式。

当然，当字符串大于 44 字节时，SDS 的数据量就开始变多了，Redis 就不再把 SDS 和 RedisObject 布局在一起了，而是会给 SDS 分配独立的空间，并用指针指向 SDS 结构。这种布局方式被称为 raw 编码模式

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225001309.png" style="zoom:13%;" />

Redis 会使用一个全局哈希表保存所有键值对，哈希表的每一项是一个 **dictEntry** 的结构体，用来指向一个键值对

dictEntry 结构中有三个 8 字节的指针，分别指向 key、value 以及下一个 dictEntry，三个指针共 24 字节

**Redis使用的内存分配库jemalloc** 

jemalloc 在分配内存时，会根据我们申请的字节数 N，找一个比 N 大，但是最接近 N 的 2 的幂次数作为分配的空间，这样可以减少频繁分配的次数。

举个例子。如果你申请 6 字节空间，jemalloc 实际会分配 8 字节空间；如果你申请 24 字节空间，jemalloc 则会分配 32 字节

**用什么数据结构可以节省内存**

Redis 有一种底层数据结构，叫压缩列表（ziplist），这是一种非常节省内存的结构。

压缩列表的构成。表头有三个字段 zlbytes、zltail 和 zllen，分别表示列表长度、列表尾的偏移量，以及列表中的 entry 个数。压缩列表尾还有一个 zlend，表示列表结束

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225001837.png" style="zoom:15%;" />

压缩列表之所以能节省内存，就在于它是用一系列连续的 entry 保存数据。每个 entry 的元数据包括下面几部分。

- **prev_len**，表示前一个 entry 的长度。`prev_len` 有两种取值情况：1 字节或 5 字节。取值 1 字节时，表示上一个 entry 的长度小于 254 字节。虽然 1 字节的值能表示的数值范围是 0 到 255，但是压缩列表中 zlend 的取值默认是 255，因此，就默认用 255 表示整个压缩列表的结束，其他表示长度的地方就不能再用 255 这个值了。所以，当上一个 entry 长度小于 254 字节时，`prev_len` 取值为 1 字节，否则，就取值为 5 字节。
- **len**：表示自身长度，4 字节；
- **encoding**：表示编码方式，1 字节；
- **content**：保存实际数据

[Redis容量预估](http://www.redis.cn/redis_memory/)