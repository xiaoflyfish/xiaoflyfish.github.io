---
title: ReentrantLock
categories: 
- 并发编程
- 并发工具类
---

# 类结构

ReentrantLock 类本身是不继承 AQS 的，实现了 Lock 接口，如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {}
```

Lock 接口定义了各种加锁，释放锁的方法，接口有如下几个：

```java
// 获得锁方法，获取不到锁的线程会到同步队列中阻塞排队
 
void lock();
 
// 获取可中断的锁
 
void lockInterruptibly() throws InterruptedException;
 
// 尝试获得锁，如果锁空闲，立马返回 true，否则返回 false
 
boolean tryLock();
 
// 带有超时等待时间的锁，如果超时时间到了，仍然没有获得锁，返回 false
 
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
 
// 释放锁
 
void unlock();
 
// 得到新的 Condition
 
Condition newCondition();
```

ReentrantLock 就负责实现这些接口，我们使用时，直接面对的也是这些方法，这些方法的底层实现都是交给 Sync 内部类去实现的，Sync 类的定义如下：

```java
abstract static class Sync extends AbstractQueuedSynchronizer {}
```

Sync 继承了 AbstractQueuedSynchronizer ，所以 Sync 就具有了锁的框架，根据 AQS 的框架，Sync 只需要实现 AQS 预留的几个方法即可，但 Sync 也只是实现了部分方法，还有一些交给子类 NonfairSync 和 FairSync 去实现了，NonfairSync 是非公平锁，FairSync 是公平锁，定义如下：

```java
// 同步器 Sync 的两个子类锁
 
static final class FairSync extends Sync {}
 
static final class NonfairSync extends Sync {}
```

几个类整体的结构如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c631d3cae0d48e9941720bb3aedbd7d~tplv-k3u1fbpfcp-watermark.image)

# AQS

AQS 内部有一个 volatile 类型的 state 属性，实际上多线程对锁的竞争体现在对 state 值写入的竞争。一旦 state 从 0 变为 1，代表有线程已经竞争到锁，那么其它线程则进入等待队列。等待队列是一个链表结构的 FIFO 队列，这能够确保公平锁的实现。同一线程多次获取锁时，如果之前该线程已经持有锁，那么对 state 再次加 1。释放锁时，则会对 state-1。直到减为 0，才意味着此线程真正释放了锁。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8506d9f28264f1289eebd379db309be~tplv-k3u1fbpfcp-watermark.image)

# 底层原理

**加锁和释放锁的底层原理**

AQS对象内部有一个核心的变量叫做state，是int类型的，代表了加锁的状态。初始状态下，这个state的值是0。

另外，这个AQS内部还有一个关键变量，用来记录当前加锁的是哪个线程，初始化状态下，这个变量是null

接着线程1跑过来调用ReentrantLock的lock()方法尝试进行加锁，这个加锁的过程，直接就是用CAS操作将state值从0变为1

如果之前没人加过锁，那么state的值肯定是0，此时线程1就可以加锁成功。

一旦线程1加锁成功了之后，就可以设置当前加锁线程是自己

释放锁的过程非常的简单，就是将AQS内的state变量的值递减1，如果state值为0，则彻底释放锁，会将“加锁线程”变量也设置为null！

# 其他问题

**公平锁和非公平锁区别**

公平锁比非公平锁只多一行代码 !hasQueuedPredecessors()，它用来查看队列中是否有比它等待时间更久的线程，如果没有，就尝试一下是否能获取到锁，如果获取成功，则标记为已经被占用