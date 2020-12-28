---
title: ArrayList
categories: 
- Java基础
- 集合类
---

# 整体架构

ArrayList 整体架构比较简单，就是一个数组结构，比较简单

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201213142046.png)

图中展示是长度为 10 的数组，从 1 开始计数，index 表示数组的下标，从 0 开始计数，elementData 表示数组本身，还有以下三个基本概念：

- `DEFAULT_CAPACITY` 表示数组的初始大小，默认是 10；
- size 表示当前数组的大小，类型 int，没有使用 volatile 修饰，非线程安全的；
- modCount 统计当前数组被修改的版本次数，数组结构有变动，就会 +1

# 扩容

扩容是通过这行代码来实现的：`Arrays.copyOf(elementData, newCapacity);`，这行代码描述的本质是数组之间的拷贝，扩容是会先新建一个符合我们预期容量的新数组，然后把老数组的数据拷贝过去，我们通过 `System.arraycopy` 方法进行拷贝，此方法是 native 的方法

**小结**

- ArrayList是List接口的一个可变大小的数组的实现
- ArrayList的内部是使用一个Object对象数组来存储元素的
- 初始化ArrayList的时候，可以指定初始化容量的大小，如果不指定，就会使用默认大小，为10
- 当添加一个新元素的时候，首先会检查容量是否足够添加这个元素，如果够就直接添加，如果不够就进行扩容，扩容为原数组容量的1.5倍
- 当删除一个元素的时候，会将数组右边的元素全部左移



