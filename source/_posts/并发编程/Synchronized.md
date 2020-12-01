---
title: Synchronized
categories: 
- 并发编程
- 并发工具类
---

**一句话概括Synchronized作用**

能够保证在同一时刻最多只有一个线程执行该段代码，以达到保证并发安全的效果。

**synchronized和lock的区别**：

synchronized属于JVM层面，底层通过 **monitorenter** 和 **monitorexit** 两个指令实现；

lock是JUC提供的具体类，是API层面的东西；

synchronized不需要用户手动释放锁，当synchronized代码执行完毕之后会自动让线程释放持有的锁；

lock需要一般使用try-finally模式去手动释放锁，并且加锁-解锁数量需要一直，否则容易出现死锁或者程序不终止现象；

synchronized是不可中断的，除非抛出异常或者程序正常退出；

lock可中断：

* 设置超时方法tryLock(time, unit)；

* 使用lockInterruptibly，调用iterrupt方法可中断；

synchronized是非公平锁；

lock默认是非公平锁，但是可以通过构造函数传入boolean类型值更改是否为公平锁；

锁是否能绑定多个条件（condition）：

- synchronized没有condition的说法，要么唤醒所有线程，要么随机唤醒一个线程；
- lock可以使用condition实现分组唤醒需要唤醒的线程，实现**精准唤醒**；

# 两类用法

对象锁：包括方法锁（默认锁对象为this当前实例对象）和同步代码块锁（自己指定锁对象）
类锁：指synchronized修饰静态的方法或指定锁为Class对象。

# 性质

**可重入性质**

同一线程的外层函数获得锁之后，内层函数可以直接再次获取该锁。

一个线程拿一旦拿到一把锁，它可以对锁进行多次使用，那么就是可重入的，所以可重入锁也叫递归锁；如果一个线程拿到一把锁后，如果想要再次使用就必须先释放锁，然后和其它线程进行竞争，这就是不可重入。

**好处**

避免死锁：比如说有两个方法A、B都被synchronized修饰了，线程1执行到了方法A，获取了这把锁，要想执行方法B也需要获取该锁。假设synchronized不具备可重入性质，线程1执行方法A虽然获取了该锁，但是想要访问方法B不能直接使用该锁，此时既想获取锁又不想释放锁，这就造成永远等待即死锁。

**不可中断性**

一旦这个锁已经被别人获得了，如果我还想获得，我只能选择等待或者阻塞，直到别的线程釋放这个锁。如果别人永远不释放锁,那么我只能永远地等下去。

相比之下，Lock类可以拥有中断的能力，第一点，如果我觉得我等的时间太长了，有权中断现在已经获取到锁的线程的执行；第二点，如果我觉得我等待的时间太长了不想再等了，也可以退出

# 深入原理

synchronized关键字被编译成字节码后会被翻译成monitorenter 和 monitorexit 两条指令分别在同步块逻辑代码的起始位置与结束位置。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1844800b7456408ab8f50d8f7d52940e~tplv-k3u1fbpfcp-watermark.image)

每个同步对象都有一个自己的Monitor(监视器锁)，加锁过程如下图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dfc33d036694f46a11b70abdfba8e03~tplv-k3u1fbpfcp-watermark.image)

**Monitor监视器锁**

**monitorenter**：每个对象都是一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. **如果monitor的进入数为0**，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者；
2. **如果线程已经占有该monitor**，只是重新进入，则进入monitor的进入数加1；
3. **如果其他线程已经占用了monitor**，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权；

**monitorexit**：执行monitorexit的线程必须是objectref所对应的monitor的所有者。**指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者**。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

**monitorexit，指令出现了两次，第1次为同步正常退出释放锁；第2次为发生异步退出释放锁**；

**同步方法**

方法的同步并没有通过指令 **monitorenter** 和 **monitorexit** 来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了 **ACC_SYNCHRONIZED** 标示符。**JVM就是根据该标示符来实现方法的同步的**

当方法调用时，**调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置**，如果设置了，**执行线程将先获取monitor**，获取成功之后才能执行方法体，**方法执行完后再释放monitor**。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。

**可见性原理**

synchronized修饰的方法或者代码块在执行完毕后，该方法或者代码块中对共享变量的所作的任何修改都要在释放锁之前从线程内存写入到主内存中，因此就保证了线程内存的变量与主内存中变量的一致性。同样，在进入同步代码块或者方法中所得到的共享变量值也是直接从主内存中获取的。