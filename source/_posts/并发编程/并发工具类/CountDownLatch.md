---
title: CountDownLatch
categories: 
- 并发编程
- 并发工具类
---

CountDownLatch会等主线程等待另外一组线程都执行完成之后，再继续执行。

**使用CountDownLatch**

我们来模拟打篮球的例子，主线程假如是控球后卫

```java
public class Client {
    private static  final CountDownLatch countDownLatch = new CountDownLatch(5);
    public static void main(String[] args) throws InterruptedException {

        System.out.println("控球后卫到位！等待所有位置球员到位！");
        countDownLatch.countDown();

        new Thread(()->{
            System.out.println("得分后卫到位！");
            countDownLatch.countDown();
        }).start();

        new Thread(()->{
            System.out.println("中锋到位！");
            countDownLatch.countDown();
        }).start();

        new Thread(()->{
            System.out.println("大前锋到位！");
            countDownLatch.countDown();
        }).start();

        new Thread(()->{
            System.out.println("小前锋到位！");
            countDownLatch.countDown();
        }).start();

        countDownLatch.await();

        System.out.print("全部到位，开始进攻！");
    }
}
```