---
title: AQS
categories: 
- 并发编程
---

AQS定义了一套多线程访问共享资源的同步器框架

**整体架构图**

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201230165133.png" style="zoom:33%;" />

**JDK里利用AQS类的主要步骤：**

1. 在内部写一个 `Sync` 类，该 `Sync` 类继承 `AbstractQueuedSynchronizer`，即 AQS；

2. 在 `Sync` 类里，根据是否是独占，来重写对应的方法。如果是独占，则重写 `tryAcquire` 和 `tryRelease` 等方法；如果是非独占，则重写 `tryAcquireShared` 和 `tryReleaseShared` 等方法；

3. 实现获取/释放的相关方法，并在里面调用 `AQS` 对应的方法，如果是独占则调用 `acquire` 或 `release` 等方法，非独占则调用 `acquireShared` 或 `releaseShared` 或 `acquireSharedInterruptibly` 等

**基本属性**

```java
// 头结点，你直接把它当做 当前持有锁的线程 可能是最好理解的
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
// 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
private volatile int state;

// 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```

AQS内部维护属性`state`表示资源的可用状态

**AQS定义两种队列**

* 同步等待队列
* 条件等待队列

不同自定义同步器争用共享资源的方式也不同

自定义同步器在实现时只需要实现共享资源`state`的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），`AQS`已经在顶层实现好了

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201230164801.png" style="zoom:33%;" />

**同步等待队列**

AQS当中的同步等待队列也称CLH队列，CLH队列是一种基于双向链表数据结构的队列，是`FIFO`先入先出线程等待队列

AQS依赖它来管理等待中的线程，如果线程获取同步竞争资源失败时，会将线程阻塞，并加入到`CLH`同步队列；当竞争资源空闲时，基于`CLH`队列阻塞线程并分配资源

**条件等待队列**

Condition是一个多线程间协调通信的工具类，使得某个或者某些线程一起等待某个条件（`Condition`）,只有当该条件具备时，这些等待线程才会被唤醒，从而重新争夺锁

```java
// 排它模式下，尝试获得锁
public final void acquire(int arg) {
    // tryAcquire 方法是需要实现类去实现的，实现思路一般都是 cas 给 state 赋值来决定是否能获得锁
    if (!tryAcquire(arg) &&
        // addWaiter 入参代表是排他模式
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
// unlock 的基础方法
public final boolean release(int arg) {
    // tryRelease 交给实现类去实现，一般就是用当前同步器状态减去 arg，如果返回 true 说明成功释放锁。
    if (tryRelease(arg)) {
        Node h = head;
        // 头节点不为空，并且非初始化状态
        if (h != null && h.waitStatus != 0)
            // 从头开始唤醒等待锁的节点
            unparkSuccessor(h);
        return true;
    }
    return false;
```

