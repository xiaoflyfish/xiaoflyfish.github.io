---
title: Final
categories: 
- Java基础
---

**final修饰变量**

关键字 final 修饰变量的作用是很明确的，那就是意味着这个变量一旦被赋值就不能被修改了，也就是说只能被赋值一次。如果我们尝试对一个已经赋值过 final 的变量再次赋值，就会报编译错误。

**final修饰参数**

关键字 final 还可以用于修饰方法中的参数。在方法的参数列表中是可以把参数声明为 final 的，这意味着我们没有办法在方法内部对这个参数进行修改

**final修饰方法**

选择用 final 修饰方法的原因之一是为了提高效率，因为在早期的 Java 版本中，会把 final 方法转为内嵌调用，可以消除方法调用的开销，以提高程序的运行效率。不过在后期的 Java 版本中，JVM 会对此自动进行优化，所以不需要我们程序员去使用 final 修饰方法来进行这些优化了，即便使用也不会带来性能上的提升。

目前我们使用 final 去修饰方法的唯一原因，就是想把这个方法锁定，意味着任何继承类都不能修改这个方法的含义，也就是说，被 final 修饰的方法不可以被重写，不能被 override

构造方法不允许被 final 修饰

**final的private方法**

```java
/**
 * private方法隐式指定为final
 */
public class PrivateFinalMethod {
    private final void privateEat() {
    }
}
class SubClass2 extends PrivateFinalMethod {
    private final void privateEat() {//编译通过，但这并不是真正的重写
    }
}
```

类中的所有 private 方法都是隐式的指定为自动被 final 修饰的，我们额外的给它加上 final 关键字并不能起到任何效果。由于我们这个方法是 private 类型的，所以对于子类而言，根本就获取不到父类的这个方法，就更别说重写了。在上面这个代码例子中，其实子类并没有真正意义上的去重写父类的 privateEat 方法，子类和父类的这两个 privateEat 方法彼此之间是独立的，只是方法名碰巧一样而已。

为了证明这点，我们尝试在子类的 privateEat 方法上加个 Override 注解，这个时候就会提示“Method does not override method from its superclass”，意思是“该方法没有重写父类的方法”，就证明了这不是一次真正的重写。

**final修饰类**

final 修饰的这个类不可被继承

假设我们给某个类加上了 final 关键字，这并不代表里面的成员变量自动被加上 final。事实上，这两者之间不存在相互影响的关系，也就是说，类是 final 的，不代表里面的属性就会自动加上 final。

**final修饰对象时，只是引用不可变**

当我们用 final 去修饰一个指向对象类型（而不是指向 8 种基本数据类型，例如 int 等）的变量时候，那么 final 起到的作用只是保证这个变量的引用不可变，而对象本身的内容依然是可以变化的

