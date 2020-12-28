---
title: String
categories: 
- Java基础
---

**String是不可变的**

```java
public final class String
    implements Java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
	//...
}
```

String 类是被 final 修饰的，所以这个 String 类是不会被继承的，因此没有任何人可以通过扩展或者覆盖行为来破坏 String 类的不变性。

这里面有个非常重要的属性，即 private final 的 char 数组，数组名字叫 value。它存储着字符串的每一位字符，同时 value 数组是被 final 修饰的，也就是说，这个 value 一旦被赋值，引用就不能修改了；并且在 String 的源码中可以发现，除了构造函数之外，并没有任何其他方法会修改 value 数组里面的内容，而且 value 的权限是 private，外部的类也访问不到，所以最终使得 value 是不可变的。

**不可变的好处**

1.可以使用字符串常量池

2.用作HashMap的key

通常建议把不可变对象作为 HashMap的 key，比如 String 就很合适作为 HashMap 的 key

3.缓存HashCode

4.线程安全

因为具备不变性的对象一定是线程安全的，我们不需要对其采取任何额外的措施，就可以天然保证线程安全