---
title: ThreadLocal
categories: 
- Java基础
---

ThreadLocal 提供了一种方式，让在多线程环境下，每个线程都可以拥有自己独特的数据，并且可以在整个线程执行过程中，从上而下的传递

**使用场景**

1、存储需要在线程隔离的数据。比如线程执行的上下文信息，每个线程是不同的，但是对于同一个线程来说会共享同一份数据。Spring MVC的 RequestContextHolder 的实现就是使用了ThreadLocal；

2、跨层传递参数。层次划分在软件设计中十分常见。层次划分后，体现在代码层面就是每层负责不同职责，一个完整的业务操作，会由一系列不同层的类的方法调用串起来完成。有些时候第一层获得的一个变量值可能在第三层、甚至更深层的方法中才会被使用。如果我们不借助ThreadLocal，就只能一层层地通过方法参数进行传递

简单说就是：

每个线程需要有自己单独的实例

实例需要在一个线程中多个方法中共享，但不希望被多线程共享

**原理**

当执行set方法时，其值是保存在当前线程的`threadLocals`变量中，当执行get方法中，是从当前线程的`threadLocals`变量获取

`threadLocals`是一个`ThreadLocalMap`数据结构，

在ThreadLoalMap中，也是初始化一个大小16的Entry数组，Entry对象用来保存每一个key-value键值对，只不过这里的key永远都是ThreadLocal对象，通过ThreadLocal对象的set方法，结果把ThreadLocal对象自己当做key，放进了ThreadLoalMap中

ThreadLoalMap的Entry是继承WeakReference，和HashMap很大的区别是，Entry中没有next字段，所以就不存在链表的情况了

**内存泄露**

threadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal不存在外部强引用时，Key势必会被GC回收，这样就会导致ThreadLocalMap中key为null， 而value还存在着强引用，只有thead线程退出以后，value的强引用链条才会断掉

但如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链，永远无法回收，造成内存泄漏

所以每次使用完ThreadLocal都调用它的remove()方法清除数据

将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，也就能保证任何时候都能通过ThreadLocal的弱引用访问到Entry的value值，进而清除掉