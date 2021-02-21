---
title: Volatile
categories: 
- 并发编程
---

volatile是Java虚拟机提供的轻量级的同步机制

volatile关键字有如下两个作用

- 保证被volatile修饰的共享变量对所有线程总数可见的，也就是当一个线程修改了一个被volatile修饰共享变量的值，新值总是可以被其他线程立即得知。
- 禁止指令重排序优化。

**内存可见性**

- 第一： 使用volatile关键字会强制将修改的值立即写入主存；
- 第二： 使用volatile关键字的话，当线程2进行修改时， 会导致线程1的工作内存中缓存变量的缓存行无效；
- 第三： 由于线程1的工作内存中缓存变量的缓存行无效，所以线程1再次读取变量的值时会去主存读取。

**禁止重排序**

1.当程序执行到volatile变量的读操作或者写操作时， 在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见

2.在进行指令优化时，不能把volatile变量前面的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

为了实现volatile的内存语义， 加入 volatile 关键字时， 编译器在生成字节码时，会在指令序列中插入内存屏障， 会多出一个 lock 前缀指令。

**内存屏障，有2个作用：**

1.先于这个内存屏障的指令必须先执行， 后于这个内存屏障的指令必须后执行。

2.使得内存可见性。 所以， 如果你的字段是 volatile， 在读指令前插入读屏障， 可以让高速缓存中的数据失效， 重新从主内存加载数据。 在写指令之后插入写屏障， 能让写入缓存的最新数据写回到主内存。

**硬件层的内存屏障**

Intel硬件提供了一系列的内存屏障，主要有：

> lfence，是一种Load Barrier 读屏障
>
> sfence, 是一种Store Barrier 写屏障
>
> mfence, 是一种全能型的屏障，具备ifence和sfence的能力
>
> Lock前缀，Lock不是一种内存屏障，但是它能完成类似内存屏障的功能。Lock会对CPU总线和高速缓存加锁，可以理解为CPU指令级的一种锁

不同硬件实现内存屏障的方式不同，Java内存模型屏蔽了这种底层硬件平台的差异，由JVM来为不同的平台生成相应的机器码。 JVM中提供了四类内存屏障指令：

| 屏障类型   | 指令示例                   | 说明                                                         |
| ---------- | -------------------------- | ------------------------------------------------------------ |
| LoadLoad   | Load1; LoadLoad; Load2     | 保证load1的读取操作在load2及后续读取操作之前执行             |
| StoreStore | Store1; StoreStore; Store2 | 在store2及其后的写操作执行前，保证store1的写操作已刷新到主内存 |
| LoadStore  | Load1; LoadStore; Store2   | 在stroe2及其后的写操作执行前，保证load1的读操作已读取结束    |
| StoreLoad  | Store1; StoreLoad; Load2   | 保证store1的写操作已刷新到主内存之后，load2及其后的读操作才能执行 |

Memory Barrier的另外一个作用是强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本

JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略。

> 在每个volatile写操作的前面插入一个StoreStore屏障。
>
> 在每个volatile写操作的后面插入一个StoreLoad屏障。
>
> 在每个volatile读操作的后面插入一个LoadLoad屏障。
>
> 在每个volatile读操作的后面插入一个LoadStore屏障。

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210221171307.png" style="zoom:33%;" />

**无法保证原子性**

```java
//示例
public class VolatileVisibility {
    public static volatile int i =0;
    public static void increase(){
        i++;
    }
}
```

在并发场景下，i变量的任何改变都会立马反应到其他线程中，但是如此存在多条线程同时调用increase()方法的话，就会出现线程安全问题，毕竟i++，操作并不具备原子性，该操作是先读取值，然后写回一个新值，相当于原来的值加上1，分两步完成

如果第二个线程在第一个线程读取旧值和写回新值期间读取i的域值，那么第二个线程就会与第一个线程一起看到同一个值，并执行相同值的加1操作，这也就造成了线程安全失败，因此对于increase方法必须使用synchronized修饰，以便保证线程安全，需要注意的是一旦使用synchronized修饰方法后，由于synchronized本身也具备与volatile相同的特性，即可见性，因此在这样种情况下就完全可以省去volatile修饰变量

**只能保证读取/写入的原子性，不保证组合操作的原子性**

加了 volatile 后，所有变量的读/写操作（包含 long 和 double）。这也就意味着 long 和 double 加了 volatile 关键字之后，对它们的读写操作同样具备原子性；

