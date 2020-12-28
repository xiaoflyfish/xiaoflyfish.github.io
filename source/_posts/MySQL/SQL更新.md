---
title: SQL更新
categories: 
- MySQL
---

首先假设我们有一条SQL语句是这样的：

```sql
update users set name='xxx' where id=10
```

那么这条SQL语句是如何执行的

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201225093103.png" style="zoom:33%;" />

InnoDB存储引擎中有一个非常重要的放在内存里的组件，就是缓冲池（`Buffer Pool`），这里面会缓存很多的数据， 以便于以后在查询的时候，万一你要是内存缓冲池里有数据，就可以不用去查磁盘了

**数据丢失**

根据一定的策略把redo日志从`redo log buffer`里刷入到磁盘文件里去。

这个策略是通过`innodb_flush_log_at_trx_commit`来配置的

当这个参数的值为0的时候，那么你提交事务的时候，不会把redo log buffer里的数据刷入磁盘文件的

当这个参数的值为1的时候，你提交事务的时候，就必须把redo log从内存刷入到磁盘文件里去

参数的值是2，提交事务的时候，把redo日志写入磁盘文件对应的os cache缓存里去，而不是直接进入磁盘文件，可 能1秒后才会把`os cache`里的数据写入到磁盘文件里去

