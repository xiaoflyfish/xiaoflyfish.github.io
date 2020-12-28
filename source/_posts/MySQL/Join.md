---
title: Join
categories: 
- MySQL
---

**join连接时哪个表是驱动表，哪个表是被驱动表：**

- 当使用`left join`时，左表是驱动表，右表是被驱动表
- 当使用`right join`时，右表是驱动表，左表是被驱动表
- 当使用`inner join`时，mysql会选择数据量比较小的表作为驱动表，大表作为被驱动表

**小表驱动大表优于大表驱动小表**

**join查询在有索引条件下**

- 驱动表有索引不会使用到索引
- 被驱动表建立索引会使用到索引

所以在以小表驱动大表的情况下，再给大表建立索引会大大提高执行速度

# 原理

mysql的join算法叫做`Nested-Loop Join`（嵌套循环连接）

而这个Nested-Loop Join有三种变种

**Simple Nested Loop**

驱动表中的每一条记录与被驱动表中的记录进行比较判断（就是个笛卡尔积）。对于两表联接来说，驱动表只会被访问一遍，但被驱动表却要被访问到好多遍

**Index Nested Loop**

这个是基于索引进行连接的算法

它要求被驱动表上有索引，可以通过索引来加速查询

**Block Nested Loop**

这个算法较`Simple Nested-Loop Join`的改进就在于可以减少被驱动表的扫描次数

因为它使用Join Buffer来减少内部循环读取表的次数

Join Buffer用以缓存联接需要的列

**关于Join Buffer**

1. Join Buffer会缓存所有参与查询的列而不是只有Join的列。
2. `join_buffer_size`的默认值是256K

在选择Join算法时，会有优先级：

```
Index Nested LoopJoin > Block Nested Loop Join > Simple Nested Loop Join
```

当不使用`Index Nested-Loop Join`的时候，默认使用`Block Nested-Loop Join`。

使用Block Nested-Loop Join算法需要开启优化器管理配置的`optimizer_switch`的设置`block_nested_loop`为on，默认为开启

