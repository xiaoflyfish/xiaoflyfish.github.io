---
title: ConcurrentHashMap
categories: 
- 并发编程
- 并发工具类
---

# 基本结构

ConcurrentHashMap 是一个存储 key/value 对的容器，并且是线程安全的。我们先看 ConcurrentHashMap 的存储结构，如下图：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1496206a83d44db9abbf39001338982~tplv-k3u1fbpfcp-watermark.image)

# put

ConcurrentHashMap 在 put 方法上的整体思路和 HashMap 相同，但在线程安全方面写了很多保障的代码，我们先来看下大体思路：

1. 如果数组为空，初始化，初始化完成之后，走 2；
2. 计算当前槽点有没有值，没有值的话，cas 创建，失败继续自旋（for 死循环），直到成功，槽点有值的话，走 3；
3. 如果槽点是转移节点(正在扩容)，就会一直自旋等待扩容完成之后再新增，不是转移节点走 4；
4. 槽点有值的，先锁定当前槽点，保证其余线程不能操作，如果是链表，新增值到链表的尾部，如果是红黑树，使用红黑树新增的方法新增；
5. 新增完成之后 check 需不需要扩容，需要的话去扩容。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4640b27832d4e2da4b521a447d082d0~tplv-k3u1fbpfcp-watermark.image)

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

为什么不直接用 key 的 hashCode，而是使用经 spreed 方法加工后的 hash 值？

**h ^ (h >>> 16)**

h >>> 16 的意思是把 h 的二进制数值向右移动 16 位。我们知道整形为 32 位，那么右移 16 位后，就是把高 16 位移到了低 16 位。而高 16 位清0了。

^为异或操作，二进制按位比较，如果相同则为 0，不同则为 1。这行代码的意思就是把高低16位做异或。如果两个hashCode值的低16位相同，但是高位不同，经过如此计算，低16位会变得不一样了。为什么要把低位变得不一样呢？这是由于哈希表数组长度n会是偏小的数值，那么进行 (n - 1) & hash 运算时，一直使用的是hash较低位的值。那么即使hash值不同，但如果低位相当，也会发生碰撞。而进行h ^ (h >>> 16)加工后的hash值，让hashCode高位的值也参与了哈希运算，因此减少了碰撞的概率。

**(h ^ (h >>> 16)) & HASH_BITS**

HASH_BITS 这个常量的值为 0x7fffffff，转化为二进制为 0111 1111 1111 1111 1111 1111 1111 1111。这个操作后会把最高位转为 0，其实就是消除了符号位，得到的都是正数。这是因为负的 hashCode 在ConcurrentHashMap 中有特殊的含义，因此我们需要得到一个正的 hashCode。

# 1.7和1.8的不同实现

## JDK1.7

jdk1.7中采用`Segment` + `HashEntry`的方式进行实现，结构如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1995bd377af8470bbbd5369aba83fca0~tplv-k3u1fbpfcp-watermark.image)

`ConcurrentHashMap`初始化时，计算出`Segment`数组的大小`ssize`和每个`Segment`中`HashEntry`数组的大小`cap`，并初始化`Segment`数组的第一个元素；其中`ssize`大小为2的幂次方，默认为16，`cap`大小也是2的幂次方，最小值为2，最终结果根据根据初始化容量`initialCapacity`进行计算

其中`Segment`在实现上继承了`ReentrantLock`，这样就自带了锁的功能。

**put实现**

当执行`put`方法插入数据时，根据key的hash值，在`Segment`数组中找到相应的位置，如果相应位置的`Segment`还未初始化，则通过CAS进行赋值，接着执行`Segment`对象的`put`方法通过加锁机制插入数据，实现如下：

场景：线程A和线程B同时执行相同`Segment`对象的`put`方法

1、线程A执行`tryLock()`方法成功获取锁，则把`HashEntry`对象插入到相应的位置；
2、线程B获取锁失败，则执行`scanAndLockForPut()`方法，在`scanAndLockForPut`方法中，会通过重复执行`tryLock()`方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行`tryLock()`方法的次数超过上限时，则执行`lock()`方法挂起线程B；
3、当线程A执行完插入操作时，会通过`unlock()`方法释放锁，接着唤醒线程B继续执行；

**size实现**

因为`ConcurrentHashMap`是可以并发插入数据的，所以在准确计算元素时存在一定的难度，一般的思路是统计每个`Segment`对象中的元素个数，然后进行累加，但是这种方式计算出来的结果并不一样的准确的，因为在计算后面几个`Segment`的元素个数时，已经计算过的`Segment`同时可能有数据的插入或则删除

在1.7的实现中，采用了如下方式：

先采用不加锁的方式，连续计算元素的个数，最多计算3次：
1、如果前后两次计算结果相同，则说明计算出来的元素个数是准确的；
2、如果前后两次计算结果都不同，则给每个`Segment`进行加锁，再计算一次元素的个数；

## JDK1.8

1.8中放弃了`Segment`臃肿的设计，取而代之的是采用`Node` + `CAS` + `Synchronized`来保证并发安全进行实现，结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16d31edaedf5496e84c6a706e1a695e9~tplv-k3u1fbpfcp-watermark.image)

**size实现**

1.8中使用一个`volatile`类型的变量`baseCount`记录元素的个数，当插入新数据或则删除数据时，会通过`addCount()`方法更新`baseCount`，实现如下：

1、初始化时`counterCells`为空，在并发量很高时，如果存在两个线程同时执行`CAS`修改`baseCount`值，则失败的线程会继续执行方法体中的逻辑，使用`CounterCell`记录元素个数的变化；

2、如果`CounterCell`数组`counterCells`为空，调用`fullAddCount()`方法进行初始化，并插入对应的记录数，通过`CAS`设置cellsBusy字段，只有设置成功的线程才能初始化`CounterCell`数组

3、如果通过`CAS`设置cellsBusy字段失败的话，则继续尝试通过`CAS`修改`baseCount`字段，如果修改`baseCount`字段成功的话，就退出循环，否则继续循环插入`CounterCell`对象；

所以在1.8中的`size`实现比1.7简单多，因为元素个数保存`baseCount`中，部分元素的变化个数保存在`CounterCell`数组中

通过累加`baseCount`和`CounterCell`数组中的数量，即可得到元素的总个数；