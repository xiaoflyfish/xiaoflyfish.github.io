---
title: SortedSet
categories: 
- Redis
- 数据类型
---

有序集合每个字符串元素都关联到一个双精度64位的浮点型数字字符串score，里面的元素总是通过score进行着排序，因此它是可以检索的一系列元素

默认升序排列，即通过命令ZRANGE实现；如果要按照降序排列，需要通过命令ZREVRANGE实现

当score即得分一样时，按照字典顺序对member进行排序，字典排序用的是二进制，它比较的是字符串的字节数组，所以实际上是比较ASCII码

## 底层结构

**Ziplist**

有序集合对象的编码可以是ziplist或者skiplist，同时满足以下条件时使用ziplist编码：

- 元素数量小于128个
- 所有member的长度都小于64字节

ziplist编码的有序集合使用紧挨在一起的压缩列表节点来保存，第一个节点保存member，第二个保存score

ziplist内的集合元素按score从小到大排序，score较小的排在表头位置

**Skiplist**

skiplist编码的有序集合底层包含一个字典和一个跳跃表 ，跳跃表按score从小到大保存所有集合元素，而字典则保存着从member到score的映射，这样就可以用O(1)的复杂度来查找member对应的score值

向有序集合中添加score和element 时间复杂度 O(logN)

获取element的分数，时间复杂度O(1)

自增element的分数，时间复杂度O(1)

返回有序集合中元素的个数，时间复杂度O(1)