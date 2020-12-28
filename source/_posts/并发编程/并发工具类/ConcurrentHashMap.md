---
title: ConcurrentHashMap
categories: 
- 并发编程
- 并发工具类
---

# 基本结构

ConcurrentHashMap 是一个存储 key/value 对的容器，并且是线程安全的

ConcurrentHashMap 的存储结构，如下图：

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1496206a83d44db9abbf39001338982~tplv-k3u1fbpfcp-watermark.image" style="zoom:50%;" />

# put

ConcurrentHashMap 在 put 方法上的整体思路和 HashMap 相同，但在线程安全方面写了很多保障的代码，我们先来看下大体思路：

1. 如果数组为空，初始化，初始化完成之后，走 2；
2. 计算当前槽点有没有值，没有值的话，cas 创建，失败继续自旋（for 死循环），直到成功，槽点有值的话，走 3；
3. 如果槽点是转移节点(正在扩容)，就会一直自旋等待扩容完成之后再新增，不是转移节点走 4；
4. 槽点有值的，先锁定当前槽点，保证其余线程不能操作，如果是链表，新增值到链表的尾部，如果是红黑树，使用红黑树新增的方法新增；
5. 新增完成之后 check 需不需要扩容，需要的话去扩容。

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4640b27832d4e2da4b521a447d082d0~tplv-k3u1fbpfcp-watermark.image" style="zoom:30%;" />

# 扩容

ConcurrentHashMap 扩容的方法叫做 transfer，从 put 方法的 addCount 方法进去，就能找到 transfer 方法，transfer 方法的主要思路是：

1. 首先需要把老数组的值全部拷贝到扩容之后的新数组上，先从数组的队尾开始拷贝；
2. 拷贝数组的槽点时，先把原数组槽点锁住，保证原数组槽点不能操作，成功拷贝到新数组时，把原数组槽点赋值为转移节点；
3. 这时如果有新数据正好需要 put 到此槽点时，发现槽点为转移节点，就会一直等待，所以在扩容完成之前，该槽点对应的数据是不会发生变化的；
4. 从数组的尾部拷贝到头部，每拷贝成功一次，就把原数组中的节点设置成转移节点；
5. 直到所有数组数据都拷贝到新数组时，直接把新数组整个赋值给数组容器，拷贝完成。

# get

ConcurrentHashMap 读的话，就比较简单，先获取数组的下标，然后通过判断数组下标的 key 是否和我们的 key 相等，相等的话直接返回，如果下标的槽点是链表或红黑树的话，分别调用相应的查找数据的方法，整体思路和 HashMap 很像

# Hash算法

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

传入的参数h为 key 对象的 hashCode，spreed 方法对 hashCode 进行了加工

**h ^ (h >>> 16)**

`h >>> 16` 的意思是把 h 的二进制数值向右移动 16 位。我们知道整形为 32 位，那么右移 16 位后，就是把高 16 位移到了低 16 位。而高 16 位清0了。

`^`为异或操作，二进制按位比较，如果相同则为 0，不同则为 1。这行代码的意思就是把高低16位做异或。如果两个hashCode值的低16位相同，但是高位不同，经过如此计算，低16位会变得不一样了。

为什么要把低位变得不一样呢？

这是由于哈希表数组长度n会是偏小的数值，那么进行` (n - 1) & hash` 运算时，一直使用的是hash较低位的值。那么即使hash值不同，但如果低位相当，也会发生碰撞。而进行`h ^ (h >>> 16)`加工后的hash值，让hashCode高位的值也参与了哈希运算，因此减少了碰撞的概率。

**(h ^ (h >>> 16)) & HASH_BITS**

`HASH_BITS` 这个常量的值为 0x7fffffff，转化为二进制为 0111 1111 1111 1111 1111 1111 1111 1111。这个操作后会把最高位转为 0，其实就是消除了符号位，得到的都是正数。这是因为负的 hashCode 在ConcurrentHashMap 中有特殊的含义，因此我们需要得到一个正的 hashCode。

# 1.7和1.8不同

**数据结构**

Java 7 采用 Segment 分段锁来实现，而 Java 8 中的 ConcurrentHashMap 使用数组 + 链表 + 红黑树

**并发度**

Java 7 中，每个 Segment 独立加锁，最大并发个数就是 Segment 的个数，默认是 16。

但是到了 Java 8 中，锁粒度更细，理想情况下 table 数组元素的个数（也就是数组长度）就是其支持并发的最大个数，并发度比之前有提高。

**保证并发安全的原理**

Java 7 采用 Segment 分段锁来保证安全，而 Segment 是继承自 ReentrantLock。

Java 8 中放弃了 Segment 的设计，采用 `Node + CAS + synchronized `保证线程安全。

**遇到 Hash 碰撞**

Java 7 在 Hash 冲突时，会使用拉链法，也就是链表的形式。

Java 8 先使用拉链法，在链表长度超过一定阈值时，将链表转换为红黑树，来提高查找效率。

**查询时间复杂度**

Java 7 遍历链表的时间复杂度是 `O(n)`，n 为链表长度。

Java 8 如果变成遍历红黑树，那么时间复杂度降低为 `O(log(n))`，n 为树的节点个数。

# 线程安全

1.使用volatile保证当Node中的值变化时对于其他线程是可见的

2.使用table数组的头结点作为synchronized的锁来保证写操作的安全

3.当头结点为null时，使用CAS操作来保证数据能正确的写入