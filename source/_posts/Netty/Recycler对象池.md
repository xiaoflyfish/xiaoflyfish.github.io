---
title: Recycler对象池
categories: 
- Netty
---

对象池与内存池的都是为了提高 `Netty `的并发处理能力，我们知道 Java 中频繁地创建和销毁对象的开销是很大的，所以很多人会将一些通用对象缓存起来，当需要某个对象时，优先从对象池中获取对象实例。通过重用对象，不仅避免频繁地创建和销毁所带来的性能损耗，而且对 JVM GC 是友好的，这就是对象池的作用。

Recycler 是 `Netty` 提供的自定义实现的轻量级对象回收站，借助 Recycler 可以完成对象的获取和回收

# 内部组件

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201224090803.png" style="zoom:30%;" />

第一个核心组件是 `Stack`，Stack 是整个对象池的顶层数据结构，描述了整个对象池的构造，用于存储当前本线程回收的对象。在多线程的场景下，Netty 为了避免锁竞争问题，每个线程都会持有各自的对象池，内部通过 `FastThreadLocal` 来实现每个线程的私有化

第二个要介绍的组件是 `WeakOrderQueue`，WeakOrderQueue 用于存储其他线程回收到当前线程所分配的对象，并且在合适的时机，Stack 会从异线程的 `WeakOrderQueue` 中收割对象

第三个组件是` Link`，每个 WeakOrderQueue 中都包含一个 Link 链表，回收对象都会被存在 Link 链表中的节点上

第四个组件是 `DefaultHandle`，DefaultHandle 实例中保存了实际回收的对象，Stack 和 WeakOrderQueue 都使用 DefaultHandle 存储回收的对象

**从Recycler中获取对象**

```java
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    Stack<T> stack = threadLocal.get(); // 获取当前线程缓存的 Stack
    DefaultHandle<T> handle = stack.pop(); // 从 Stack 中弹出一个 DefaultHandle 对象
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle); // 创建的对象并保存到 DefaultHandle
    }
    return (T) handle.value;
}
```

这个`Recycler#get() `方法的逻辑非常清晰，首先通过 FastThreadLocal 获取当前线程的唯一栈缓存 Stack，然后尝试从栈顶弹出 DefaultHandle 对象实例，如果 Stack 中没有可用的 `DefaultHandle` 对象实例，那么会调用 `newObject` 生成一个新的对象，完成 handle 与用户对象和 Stack 的绑定