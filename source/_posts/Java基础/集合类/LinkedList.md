---
title: LinkedList
categories: 
- Java基础
- 集合类
---

LinkedList 适用于集合元素先入先出和先入后出的场景

# 整体架构

LinkedList 底层数据结构是一个双向链表

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201213125820.png)

链表每个节点我们叫做 Node，Node 有 prev 属性，代表前一个节点的位置，next 属性，代表后一个节点的位置；

first 是双向链表的头节点，它的前一个节点是 null。

last 是双向链表的尾节点，它的后一个节点是 null；

当链表中没有数据时，first 和 last 是同一个节点，前后指向都是 null；

因为是个双向链表，只要机器内存足够强大，是没有大小限制的

LinkedList类实现了List接口和Deque接口，是一种链表类型的数据结构，支持高效的插入和删除操作，同时也实现了Deque接口，使得LinkedList类也具有队列的特性。

# 节点查询

链表查询某一个节点是比较慢的，需要挨个循环查找才行

LinkedList 并没有采用从头循环到尾的做法，而是采取了简单二分法，首先看看 index 是在链表的前半部分，还是后半部分。如果是前半部分，就从头开始寻找，反之亦然。通过这种方式，使循环的次数至少降低了一半，提高了查找的性能

