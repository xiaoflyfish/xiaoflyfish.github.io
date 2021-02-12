---
title: BitMap
categories: 
- Redis
- 数据类型
---

bitmaps为一个以位为单位的数组，数组的每个单元只能存储0和1

在redis 2.2.0版本之后，新增了一个位图数据，其实它不是一种数据结构

实际上它就是一个一个字符串结构（涉及单个bitmap**可存储最大值问题**），只不过value是一个二进制数据，每一位只能是0或者1

redis单独对bitmap提供了一套命令。可以对任意一位进行设置和读取

http://redisdoc.com/bitmap/index.html

# 登录统计

**登录标记**

用户登录时，使用setbit命令和用户id（假设id=123456）标记当日（2020-10-05）用户已经登录，具体命令如下：

```
# 时间复杂度O(1)
setbit login:20201005 123456 1
```

**每日用户登录数量统计**

```
# 时间复杂度O(N)
bitcount login:20201005
```

**活跃用户（连续三日登录）统计**

如果我们想要获取近三日活跃用户数量的话，可以使用bitop命令

bitmap的bitop命令支持对bitmap进行`AND(与)`，`(OR)或`，`XOR(亦或)`，`NOT(非)`四种相关操作，我们对近三日的bitmap做`AND`操作即可，操作之后会形成一个新的bitmap，我们可以取名为`login:20201005:b3`

```
# 时间复杂度O(N)
bitop and login:20201005:b3 login:20201005 login:20201004 login:20201003
```

然后我们可以对`login:20201005:b3`使用bitcount或者getbit命令，用于统计活跃用户数量，或者查看某个用户是否为活跃用户

**内存占用**

我们新建一个bitmap，用于测试最大值4294967296-1，内存相关占用：

```
127.0.0.1:6379> setbit login:20201005 4294967296 1
(error) ERR bit offset is not an integer or out of range
127.0.0.1:6379> setbit login:20201005 4294967295 1
(integer) 0
```

我们可以发现直接设置4294967296（超过最大值）会出现报错

