---
title: String
categories: 
- Java基础
---

以主流的 JDK 版本 1.8 来说，String 内部实际存储结构为 `char` 数组，源码如下：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // 用于存储字符串的值
    private final char value[];
    // 缓存字符串的 hash code
    private int hash; // Default to 0
    // ......其他内容
}
```

equals()比较两个字符串是否相等

```java
public boolean equals(Object anObject) {
    // 对象引用相同直接返回 true
    if (this == anObject) {
        return true;
    }
    // 判断需要对比的值是否为 String 类型，如果不是则直接返回 false
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            // 把两个字符串都转换为 char 数组对比
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            // 循环比对两个字符串的每一个字符
            while (n-- != 0) {
                // 如果其中有一个字符不相等就 true false，否则继续对比
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

String 类型重写了 Object 中的 `equals()` 方法，`equals()` 方法需要传递一个 Object 类型的参数值

**== 和 equals 的区别**

== 对于基本数据类型来说，是用于比较 “值”是否相等的；而对于引用类型来说，是用于比较引用地址是否相同的。

Object 中也有 `equals()` 方法，源码如下：

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

**final 修饰的好处**

从 String 类的源码我们可以看出 String 是被 final 修饰的不可继承类

Java 语言之父 `James Gosling` 的回答是，他会更倾向于使用 `final`，因为它能够缓存结果，当你在传参时不需要考虑谁会修改它的值；如果是可变类的话，则有可能需要重新拷贝出来一个新值进行传参，这样在性能上就会有一定的损失。

James Gosling 还说迫使 String 类设计成不可变的另一个原因是**安全**，当你在调用其他方法时，比如调用一些系统级操作指令之前，可能会有一系列校验，如果是可变类的话，可能在你校验过后，它的内部的值又被改变了，这样有可能会引起严重的系统崩溃问题，这是迫使 `String` 类设计成不可变类的一个重要原因。

总结来说，使用 final 修饰的第一个好处是**安全**；第二个好处是**高效**，以 JVM 中的字符串常量池来举例

只有字符串是不可变时，我们才能实现字符串常量池，字符串常量池可以为我们缓存字符串，提高程序的运行效率

这里面有个非常重要的属性，即 `private final` 的 char 数组，数组名字叫 value。它存储着字符串的每一位字符，同时 value 数组是被 final 修饰的，也就是说，这个 value 一旦被赋值，引用就不能修改了；并且在 String 的源码中可以发现，除了构造函数之外，并没有任何其他方法会修改 value 数组里面的内容，而且 value 的权限是 private，外部的类也访问不到，所以最终使得 value 是不可变的。

**String 和 JVM**

String 常见的创建方式有两种，`new String()` 的方式和直接赋值的方式，直接赋值的方式会先去字符串常量池中查找是否已经有此值，如果有则把引用地址直接指向此值，否则会先在常量池中创建，然后再把引用指向此值；而 `new String() `的方式一定会先在堆上创建一个字符串对象，然后再去常量池中查询此字符串的值是否已经存在，如果不存在会先在常量池中创建此字符串，然后把引用的值指向此字符串，如下代码所示：

```java
String s1 = new String("Java");
String s2 = s1.intern();
String s3 = "Java";
System.out.println(s1 == s2); // false
System.out.println(s2 == s3); // true
```

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210206222453.png" style="zoom:50%;" />

> JDK 1.7 之后把永生代换成的元空间，把字符串常量池从方法区移到了 Java 堆上。

除此之外编译器还会对 String 字符串做一些优化，例如以下代码：

```java
String s1 = "Ja" + "va";
String s2 = "Java";
System.out.println(s1 == s2);
```

虽然 s1 拼接了多个字符串，但对比的结果却是 true，我们使用反编译工具，看到的结果如下：

```c++
Compiled from "StringExample.java"
public class com.lagou.interview.StringExample {
  public com.lagou.interview.StringExample();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
    LineNumberTable:
      line 3: 0

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String Java
       2: astore_1
       3: ldc           #2                  // String Java
       5: astore_2
       6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: aload_1
      10: aload_2
      11: if_acmpne     18
      14: iconst_1
      15: goto          19
      18: iconst_0
      19: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
      22: return
    LineNumberTable:
      line 5: 0
      line 6: 3
      line 7: 6
      line 8: 22
}
```

从编译代码 `#2` 可以看出，代码 "Ja"+"va" 被直接编译成了 "Java" ，因此 `s1==s2` 的结果才是 true，这就是编译器对字符串优化的结果