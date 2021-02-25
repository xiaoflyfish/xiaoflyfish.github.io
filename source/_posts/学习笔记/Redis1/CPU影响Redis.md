---
title: CPU影响Redis
categories: 
- 学习笔记
- Redis1
---

**主流的 CPU 架构**

一个 CPU 处理器中一般有多个运行核心，我们把一个运行核心称为一个物理核，每个物理核都可以运行应用程序。每个物理核都拥有私有的一级缓存（Level 1 cache，简称 L1 cache），包括一级指令缓存和一级数据缓存，以及私有的二级缓存（Level 2 cache，简称 L2 cache）。

- 物理核访问它们的延迟不超过 10 纳秒
- 大小: KB 级别
- 访问内存: 百纳秒级别

不同的物理核还会共享一个共同的三级缓存（Level 3 cache，简称为 L3 cache）

L3 缓存能够使用的存储资源比较多，所以一般比较大，能达到几 MB 到几十 MB，这就能让应用程序缓存更多的数据。当 L1、L2 缓存中没有数据缓存时，可以访问 L3，尽可能避免访问内存

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225151901.png" style="zoom:33%;" />

在主流的服务器上，一个 CPU 处理器会有 10 到 20 多个物理核。同时，为了提升服务器的处理能力，服务器上通常还会有多个 CPU 处理器（也称为多 CPU Socket），每个处理器有自己的物理核（包括 L1、L2 缓存），L3 缓存，以及连接的内存，同时，不同处理器间通过总线连接

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225152221.png" style="zoom:33%;" />

在多 CPU 架构下，一个应用程序访问所在 **Socket 的本地内存和访问远端内存的延迟并不一致**，所以，我们也把这个架构称为非统一内存访问架构（Non-Uniform Memory Access，NUMA 架构）。

**CPU的NUMA架构对Redis性能的影响**

在实际应用Redis时，有一种做法：为了提升Redis的网络性能，把操作系统的网络中断处理程序和CPU核绑定。

在CPU的NUMA架构下，当网络中断处理程序、Redis实例分别和CPU核绑定后，就会有一个潜在的风险：**如果网络中断处理程序和Redis实例各自所绑的CPU核不在同一个CPU Socket上，那么，Redis实例读取网络数据时，就需要跨CPU Socket访问内存，这个过程会花费较多时间**

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225153625.png" style="zoom:13%;" />

为了避免Redis跨CPU Socket访问网络数据，我们最好把网络中断程序和Redis实例绑在同一个CPU Socket上，这样一来，Redis实例就可以直接从本地内存读取网络数据了

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210225153803.png" style="zoom:13%;" />

需要注意的是，在 CPU 的 NUMA 架构下，对 CPU 核的编号规则，并不是先把一个 CPU Socket 中的所有逻辑核编完，再对下一个 CPU Socket 中的逻辑核编码，而是先给每个 CPU Socket 中每个物理核的第一个逻辑核依次编号，再给每个 CPU Socket 中的物理核的第二个逻辑核依次编号

CPU的NUMA架构下进行绑定要注意CPU核的编号规则，可以执行**lscpu命令**来查看核的编号

```
lscpu
 
Architecture: x86_64
...
NUMA node0 CPU(s): 0-5,12-17
NUMA node1 CPU(s): 6-11,18-23
...
```

可以看到，NUMA node0 的 CPU 核编号是 0 到 5、12 到 17。其中，0 到 5 是 node0 上的 6 个物理核中的第一个逻辑核的编号，12 到 17 是相应物理核中的第二个逻辑核编号

NUMA node1 的 CPU 核编号规则和 node0 一样

> 先编1号核, 再编2号核

**CPU 多核对 Redis 性能的影响**

在多核 CPU 的场景下，一旦应用程序需要在一个新的 CPU 核上运行，那么，运行时信息就需要重新加载到新的 CPU 核上。而且，新的 CPU 核的 L1、L2 缓存也需要重新加载数据和指令，这会导致程序的运行时间增加。

> 99% 尾延迟:

我们把所有请求的处理延迟从小到大排个序，99% 的请求延迟小于的值就是 99% 尾延迟。比如说，我们有 1000 个请求，假设按请求延迟从小到大排序后，第 991 个请求的延迟实测值是 1ms，而前 990 个请求的延迟都小于 1ms，所以，这里的 99% 尾延迟就是 1ms。

> context switch:

context switch 是指线程的上下文切换，这里的上下文就是线程的运行时信息。在 CPU 多核的环境中，一个线程先在一个 CPU 核上运行，之后又切换到另一个 CPU 核上运行，这时就会发生 context switch。

> 使用 taskset 命令把一个程序绑定在一个核上运行:

执行下面的命令，就把 Redis 实例绑在了 0 号核上，其中，“-c”选项用于设置要绑定的核编号

```
taskset -c 0 ./redis-server
```

**绑核的风险和解决方案**

方案一：一个Redis实例对应绑一个物理核

在给Redis实例绑核时，我们不要把一个实例和一个逻辑核绑定，而要和一个物理核绑定，也就是说，把一个物理核的2个逻辑核都用上。

方案二：优化Redis源码

通过修改Redis源码，把子进程和后台线程绑到不同的CPU核上