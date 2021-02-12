---
title: MVCC
categories: 
- MySQL
---

[MVCC由浅入深学习](https://juejin.im/post/6873120170774036488)

下面几个语句都是当前读，都会读取最新的快照数据，都会加锁（除了第一个加共享锁，其他都是互斥锁）：

```sql
select * from table where ? lock in share mode; 
select * from table where ? for update; 
insert; 
update; 
delete;
```

在执行这几个操作时会读取最新的记录，即使是别的事务提交的数据也可以查询到。比如要update一条记录，但是在另一个事务中已经delete掉这条数据并且commit了，如果update就会产生冲突，所以在update的时候需要知道最新的数据。读取的是最新的数据，并且**需要加锁（排它锁或者共享锁）**

