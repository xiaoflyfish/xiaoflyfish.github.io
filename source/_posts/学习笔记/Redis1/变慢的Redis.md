**查看 Redis 的响应延迟:**

- 大部分时候，Redis 延迟很低，但是在某些时刻，有些 Redis 实例会出现很高的响应延迟，甚至能达到几秒到十几秒，不过持续时间不长
- 当你发现 Redis 命令的执行时间突然就增长到了几秒，基本就可以认定 Redis 变慢了。

基于当前环境下的 Redis 基线性能:

- 基线性能: 一个系统在低压力、无干扰下的基本性能，这个性能只由当前的软硬件配置决定

怎么确定基线性能:

实际上，从 2.8.7 版本开始，redis-cli 命令提供了`–intrinsic-latency` 选项，可以用来监测和统计测试期间内的最大延迟，这个延迟可以作为 Redis 的基线性能。其中，测试时长可以用`–intrinsic-latency` 选项的参数来指定。

> 例子:

该命令会打印 120 秒内监测到的最大延迟。可以看到，这里的最大延迟是 119 微秒，也就是基线性能为 119 微秒。一般情况下，运行 120 秒就足够监测到最大延迟了，所以，我们可以把参数设置为 120。

```
./redis-cli --intrinsic-latency 120
Max latency so far: 17 microseconds.
Max latency so far: 44 microseconds.
Max latency so far: 94 microseconds.
Max latency so far: 110 microseconds.
Max latency so far: 119 microseconds.

36481658 total runs (avg latency: 3.2893 microseconds / 3289.32 nanoseconds per run).
Worst run took 36x longer than the average latency.
```

一般来说，你要把运行时延迟和基线性能进行对比，**如果你观察到的 Redis 运行时延迟是其基线性能的 2 倍及以上，就可以认定 Redis 变慢了。**

如果你想了解网络对 Redis 性能的影响，一个简单的方法是用 iPerf 这样的工具，测量从 Redis 客户端到服务器端的网络延迟。如果这个延迟有几十毫秒甚至是几百毫秒，就说明，Redis 运行的网络环境中很可能有大流量的其他应用程序在运行，导致网络拥塞了。这个时候，你就需要协调网络运维，调整网络的流量分配了

**慢查询命令**

Redis 的不同命令的复杂度:

- [redis commands](https://redis.io/commands/)

如果的确有大量的慢查询命令，有两种处理方式：

- 用其他高效命令代替
- 当你需要执行排序、交集、并集操作时，可以在客户端完成，而不要用 SORT、SUNION、SINTER 这些命令，以免拖慢 Redis 实例。

还有一个比较容易忽略的慢查询命令，就是 **KEYS**。

它用于返回和输入模式匹配的所有 key，例如，以下命令返回所有包含“name”字符串的 keys。

**过期 key 操作**

过期 key 的自动删除机制，它是 Redis 用来回收内存空间的常用机制，应用广泛，本身就**会引起 Redis 操作阻塞，导致性能变慢**，所以，你必须要知道该机制对性能的影响。

默认情况下，Redis 每 100 毫秒会删除一些过期 key，具体的算法如下：

- 采样 `ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP` 个数的 key，并将其中过期的 key 全部删除；
- 如果超过 25% 的 key 过期了，则重复删除的过程，直到过期 key 的比例降至 25% 以下。

> 算法的第二条是怎么被触发的呢

其中一个重要来源，就是频繁使用带有相同时间参数的 EXPIREAT 命令设置过期 key，这就会导致，在同一秒内有大量的 key 同时过期

正常情况下，一秒内基本有 200 个过期 key 会被删除，并不会对 Redis 造成太大影响，但是如果触发第二条，Redis就会一直删除以释放空间。

删除操作是阻塞的(Redis 4.0 后可以用异步线程机制来减少阻塞影响)，一旦该条件触发，Redis 的线程就会一直执行删除，这样一来，就没办法正常服务其他的键值操作，就会进一步引起其他键值操作的延迟增加，Redis 就会变慢。

可以在过期时间参数上，加上一定大小范围内的随机数，这样，既保证 key 在一个邻近时间范围内被删除，又避免了同时过期造成的压力