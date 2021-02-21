---
title: Synchronized
categories: 
- 并发编程
- 锁
---

多线程编程中，有可能会出现多个线程同时访问同一个共享、可变资源的情况，这个资源我们称之其为临界资源；这种资源可能是：对象、变量、文件等

**如何解决线程并发安全问题**

实际上，所有的并发模式在解决线程安全问题时，采用的方案都是**序列化访问临界资源**。即在同一时刻，只能有一个线程访问临界资源，也称作**同步互斥访问**。

Java 中，提供了两种方式来实现同步互斥访问：synchronized和**Lock**

当多个线程执行一个方法时，该方法内部的局部变量并不是临界资源，因为这些局部变量是在每个线程的私有栈中，因此不具有共享性，不会导致线程安全问题。

synchroniz能够保证在同一时刻最多只有一个线程执行该段代码，以达到保证并发安全的效果

**synchronized和lock的区别**：

synchronized属于JVM层面，底层通过 `monitorenter` 和 monitorexit 两个指令实现；

lock是JUC提供的具体类，是API层面的东西；

synchronized不需要用户手动释放锁，当synchronized代码执行完毕之后会自动让线程释放持有的锁；

lock需要一般使用try-finally模式去手动释放锁，并且加锁-解锁数量需要一直，否则容易出现死锁或者程序不终止现象；

synchronized是不可中断的，除非抛出异常或者程序正常退出；

lock可中断：

* 设置超时方法tryLock(time, unit)；

* 使用`lockInterruptibly`，调用interrupt方法可中断；

synchronized是非公平锁；

lock默认是非公平锁，但是可以通过构造函数传入boolean类型值更改是否为公平锁；

锁是否能绑定多个条件：

- synchronized没有condition的说法，要么唤醒所有线程，要么随机唤醒一个线程；
- lock可以使用condition实现分组唤醒需要唤醒的线程，实现精准唤醒；

加解锁顺序不同

对于 Lock 而言如果有多把 Lock 锁，Lock 可以不完全按照加锁的反序解锁，比如我们可以先获取 Lock1 锁，再获取 Lock2 锁，解锁时则先解锁 Lock1，再解锁 Lock2，加解锁有一定的灵活度

synchronized 锁只能同时被一个线程拥有，但是 Lock 锁没有这个限制

例如在读写锁中的读锁，是可以同时被多个线程持有的，可是 `synchronized` 做不到

性能区别

在 Java 5 以及之前，`synchronized `的性能比较低，但是到了 Java 6 以后，发生了变化，因为 JDK 对 synchronized 进行了很多优化，比如自适应自旋、锁消除、锁粗化、轻量级锁、偏向锁等，所以后期的 Java 版本里的 synchronized 的性能并不比 Lock 差。

# 两类用法

对象锁：包括方法锁（默认锁对象为this当前实例对象）和同步代码块锁（自己指定锁对象）

类锁：指`synchronized`修饰静态的方法或指定锁为Class对象。

**多线程访问同步方法的几种情况**

两个线程同时访问一个对象的同步方法。

> 由于同步方法锁使用的是this对象锁，同一个对象的this锁只有一把，两个线程同一时间只能有一个线程持有该锁，所以该方法将会串行运行。

两个线程访问的是两个对象的同步方法。

> 由于两个对象的this锁互不影响，synchronized将不会起作用，所以该方法将会并行运行。

两个线程访问的是synchronized的静态方法。

> synchronized修饰的静态方法获取的是当前类模板对象的锁，该锁只有一把，无论访问多少个该类对象的方法，都将串行执行。

同时访问同步方法与非同步方法

> 非同步方法不受影响。

访问同一个对象的不同的普通同步方法。

> 由于this对象锁只有一个，不同线程访问多个普通同步方法将串行运行。

同时访问静态synchronized和非静态synchronized方法

> 静态synchronized方法的锁为class对象的锁，非静态synchronized方法锁为this的锁，它们不是同一个锁，所以它们将并行运行。

# 性质

**可重入性质**

同一线程的外层函数获得锁之后，内层函数可以直接再次获取该锁。

一个线程拿一旦拿到一把锁，它可以对锁进行多次使用，那么就是可重入的，所以可重入锁也叫递归锁；如果一个线程拿到一把锁后，如果想要再次使用就必须先释放锁，然后和其它线程进行竞争，这就是不可重入。

**好处**

避免死锁：比如说有两个方法A、B都被`synchronized`修饰了，线程1执行到了方法A，获取了这把锁，要想执行方法B也需要获取该锁。假设synchronized不具备可重入性质，线程1执行方法A虽然获取了该锁，但是想要访问方法B不能直接使用该锁，此时既想获取锁又不想释放锁，这就造成永远等待即死锁。

**不可中断性**

一旦这个锁已经被别人获得了，如果我还想获得，我只能选择等待或者阻塞，直到别的线程釋放这个锁。如果别人永远不释放锁,那么我只能永远地等下去。

相比之下，Lock类可以拥有中断的能力，第一点，如果我觉得我等的时间太长了，有权中断现在已经获取到锁的线程的执行；第二点，如果我觉得我等待的时间太长了不想再等了，也可以退出

# 深入原理

synchronized关键字被编译成字节码后会被翻译成monitorenter 和 monitorexit 两条指令分别在同步块逻辑代码的起始位置与结束位置。

每个同步对象都有一个自己的Monitor(监视器锁)，加锁过程如下图所示：

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dfc33d036694f46a11b70abdfba8e03~tplv-k3u1fbpfcp-watermark.image" style="zoom:50%;" />

**Monitor监视器锁**

monitorenter：每个对象都是一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者；
2. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1；
3. 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权；

monitorexit：执行monitorexit的线程必须是对象所对应的monitor的所有者。指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

**monitorexit，指令出现了两次，第1次为同步正常退出释放锁；第2次为发生异步退出释放锁**；

**同步方法**

方法的同步并没有通过指令 monitorenter 和 monitorexit 来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了` ACC_SYNCHRONIZED `标示符。JVM就是根据该标示符来实现方法的同步的

当方法调用时，调用指令将会检查方法的` ACC_SYNCHRONIZED `访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。

**可见性原理**

synchronized修饰的方法或者代码块在执行完毕后，该方法或者代码块中对共享变量的所作的任何修改都要在释放锁之前从线程内存写入到主内存中，因此就保证了线程内存的变量与主内存中变量的一致性。同样，在进入同步代码块或者方法中所得到的共享变量值也是直接从主内存中获取的。

## Monitor

可以把它理解为 **一个同步工具**，也可以描述为 **一种同步机制**，它通常被 **描述为一个对象**。与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，**因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁**。**也就是通常说Synchronized的对象锁，MarkWord锁标识位为10，其中指针指向的是Monitor对象的起始地址**。

在Java虚拟机（HotSpot）中，**Monitor是由ObjectMonitor实现的**，其主要数据结构如下（位于HotSpot虚拟机源码`ObjectMonitor.hpp`文件，C++实现的）：

```c++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; // 记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; // 处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

每个 Java 对象在 JVM 的对等对象的头中保存锁状态，指向 ObjectMonitor

ObjectMonitor 保存了当前持有锁的线程引用，EntryList 中保存目前等待获取锁的线程，WaitSet 保存 wait 的线程。此外还有一个计数器，每当线程获得 monitor 锁，计数器 +1，当线程重入此锁时，计数器还会 +1。当计数器不为0时，其它尝试获取 monitor 锁的线程将会被保存到EntryList中，并被阻塞。

当持有锁的线程释放了monitor 锁后，计数器 -1。当计数器归位为 0 时，所有 EntryList 中的线程会尝试去获取锁，但只会有一个线程会成功，没有成功的线程仍旧保存在 EntryList 中。

**由此可以看出 monitor 锁是非公平锁**

ObjectMonitor中有两个队列，**_WaitSet 和 _EntryList**，用来保存ObjectWaiter对象列表（ **每个等待锁的线程都会被封装成ObjectWaiter对象** ），**_owner指向持有ObjectMonitor对象的线程**，当多个线程同时访问一段同步代码时：

1. 首先会进入 `_EntryList` 集合，**当线程获取到对象的monitor后，进入 _Owner区域并把monitor中的owner变量设置为当前线程，同时monitor中的计数器count加1**；
2. 若线程调用 wait() 方法，**将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒**；
3. 若当前线程执行完毕，**也将释放monitor（锁）并复位count的值，以便其他线程进入获取monitor(锁)**；

同时，**Monitor对象存在于每个Java对象的对象头Mark Word中（存储的指针的指向），Synchronized锁便是通过这种方式获取锁的**，也是为什么Java中任意对象可以作为锁的原因，**同时notify/notifyAll/wait等方法会使用到Monitor锁对象，所以必须在同步代码块中使用**。

