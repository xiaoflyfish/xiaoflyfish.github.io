---
title: Hystrix
categories: 
- 学习笔记
- SpringCloud1
---

# 服务容错

假设，微服务系统中还存在依赖于服务 B 的服务 C。这样，基于同样的原因，服务 B 的不可用同样会导致服务 C 的不可用。类似的，系统中可能还存在服务 D 等其他服务依赖服务 C......以此类推，最终在以服务 A 为起点的整个调用链路上的所有服务都会变得不可用。这种扩散效应就是所谓的服务雪崩效应

服务雪崩效应本质上是一种服务依赖失败。服务依赖失败较之服务自身失败而言，影响更大，也更加难以发现和处理。

**服务容错的模式**

消费者容错的常见实现模式包括集群容错、服务隔离、服务熔断和服务回退

> 集群容错

某一个服务应该构建多个实例，这样当一个服务实例出现问题时可以重试其他实例。一个集群中的服务本身就是冗余的。而针对不同的重试方式就诞生了一批集群容错策略，常见的包括 Failover（失效转移）、Failback（失败通知）、Failsafe（失败安全）和 Failfast（快速失败）等。

这里以最常见、最实用的集群容错策略 Failover 为例展开讨论。Failover 即失效转移，当发生服务调用异常时，请求会重新在集群中查找下一个可用的服务提供者实例

为了防止无限重试，如果采用 Failover 机制，通常会对失败重试最大次数进行限制。

> 服务隔离

所谓隔离，就是指对资源进行有效的管理，从而避免因为资源不可用、发生失败等情况导致系统中的其他资源也变得不可用。

在日常开发过程中，我们主要的处理对象还是线程级别的隔离。

要实现线程隔离，简单而主流的做法是使用线程池。针对不同的业务场景，我们可以设计不同的线程池。因为不同的线程池之间线程是不共享的，所以某个线程池因为业务异常导致资源消耗时，不会将这种资源消耗扩散到其他线程池，从而保证其他服务持续可用。

> 服务熔断

在微服务架构中，也存在类似的“熔断器”：当系统中出现某一个异常情况时，能够直接熔断整个服务的请求处理过程。这样可以避免一直等到请求处理完毕或超时，从而避免浪费。

从设计理念上讲，服务熔断也是快速失败的一种具体表现。当服务消费者向服务提供者发起远程调用时，服务熔断器会监控该次调用，如果调用的响应时间过长，服务熔断器就会中断本次调用并直接返回。

请注意服务熔断器判断本次调用是否应该快速失败是有状态的，也就是说服务熔断器会把所有的调用结果都记录下来，如果发生异常的调用次数达到一定的阈值，那么服务熔断机制才会被触发，快速失败就会生效；反之，将按照正常的流程执行远程调用

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210217212854.png" style="zoom:33%;" />

**执行熔断的程度：**

- Closed： 对于熔断器而言，Closed 状态代表熔断器不进行任何的熔断处理。尽管这个时候人们感觉不到熔断器的存在，但它在背后会对调用失败次数进行积累，到达一定阈值或比例时则自动启动熔断机制。

- Open： 一旦对服务的调用失败次数达到一定阈值时，熔断器就会打开，这时候对服务的调用将直接返回一个预定的错误，而不执行真正的网络调用。同时，熔断器内置了一个时间间隔，当处理请求达到这个时间间隔时会进入半熔断状态。

- Half-Open： 在半开状态下，熔断器会对通过它的部分请求进行处理，如果对这些请求的成功处理数量达到一定比例则认为服务已恢复正常，就会关闭熔断器，反之就会打开熔断器

> 服务回退

当远程调用发生异常时，服务回退并不是直接抛出异常，而是产生一个另外的处理机制来应对该异常。这相当于执行了另一条路径上的代码或返回一个默认处理结果。

在现实环境中，服务回退的实现方式可以很简单，原则上只需要保证异常被捕获并返回一个处理结果即可。但在有些场景下，回退的策略则可以非常复杂，我们可能会从其他服务或数据中获取相应的处理结果，需要具体问题具体分析

**Spring Cloud 中的服务容错解决方案**

我们已经知道 Spring Cloud 中专门用于提供服务容错功能的 Spring Cloud Circuit Breaker 框架。从命名上看，Spring Cloud Circuit Breaker 是对熔断器的一种抽象，支持不同的熔断器实现方案。在 Spring Cloud Circuit Breaker 中，内置了四种熔断器

其中 Netflix Hystrix 显然来自 Netflix OSS；Resilience4j 是受 Hystrix 项目启发所诞生的一款新型的容错库；Sentinel 从定位上讲是一款包含了熔断降级功能的高可用流量防护组件；而最后的 Spring Retry 是 Spring 自研的重试和熔断框架

# 引入Hystrix

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

Spring Cloud 推出了一个全新的注解，@SpringCloudApplication 注解。该注解用来集成服务治理和服务熔断方面的核心功能

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```

@SpringCloudApplication 是一个组合注解，整合了 @SpringBootApplication、@EnableDiscoveryClient 和 @EnableCircuitBreaker 这三个微服务所需的核心注解

现在的 Bootstrap 类的定义如下所示：

```java
@SpringCloudApplication
public class InterventionApplication
```

在 Hystrix 中，最核心的莫过于HystrixCommand 类。HystrixCommand 是一个抽象类，只包含了一个抽象方法，即如下所示的 run 方法：

```java
protected abstract R run() throws Exception;
```

显然，这个方法是让开发人员实现服务容错所需要处理的业务逻辑。在微服务架构中，我们通常在这个 run 方法中添加对远程服务的访问代码。

同时我们在 HystrixCommand 类中还发现了另一个很有用的方法 getFallback。这个方法用于在 HystrixCommand 子类中设置服务回退函数的具体实现，如下所示：

```java
protected R getFallback() {
 
    throw new UnsupportedOperationException("No fallback available.");
}
```

Hystrix 是一个非常经典而完善的服务容错开发框架，同时支持了服务隔离、服务熔断和服务回退机制

**使用 Hystrix 实现服务隔离**

基于前面对 HystrixCommand 抽象类的理解，我们就可以提供一个该类的子类来实现服务隔离。针对服务隔离，Hystrix 组件在提供了线程池隔离机制的同时，还实现了信号量隔离。

这里，我们基于最常用的线程池隔离来进行介绍。典型的 HystrixCommand 子类代码风格如下所示：

```java
public class GetUserCommand extends HystrixCommand<UserMapper> {

    //远程调用 user-service 的客户端工具类
    private UserServiceClient userServiceClient;
 
    protected GetUserCommand(String name) {
        super(Setter.withGroupKey(
            //设置命令组
            HystrixCommandGroupKey.Factory.asKey("springHealthGroup"))
                //设置命令键
                .andCommandKey(HystrixCommandKey.Factory.asKey("interventionKey"))
                //设置线程池键
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(name))
                //设置命令属性
                .andCommandPropertiesDefaults(
                    HystrixCommandProperties.Setter()
                        .withExecutionTimeoutInMilliseconds(5000))
                //设置线程池属性
                .andThreadPoolPropertiesDefaults(
                    HystrixThreadPoolProperties.Setter()
                        .withMaxQueueSize(10)
                        .withCoreSize(2))
        );
    }
 
    @Override
    protected UserMapper run() throws Exception {
        return userServiceClient.getUserByUserName("springhealth_user1");
    }
 
    @Override
    protected UserMapper getFallback() {
        return new UserMapper(1L,"user1","springhealth_user1");
    }
}
```

在 Hystrix 中，从控制粒度上讲，开发人员可以从服务分组和服务本身这两个维度出发，对线程隔离机制进行配置。也就是说我们既可以把一批服务都划分到一个线程池中，也可以把单个服务划分到一个线程池中。上述代码中的 HystrixCommandGroupKey 和 HystrixCommandKey 分别用来配置服务分组名称和服务名称，然后 HystrixThreadPoolKey 用来配置线程池的名称。

当我们根据需要设置了分组、服务以及线程池名称后，接下来就需要指定与线程池相关的各个属性。这些属性都包含在 HystrixThreadPoolProperties 中。例如，在上述代码中，我们使用 maxQueueSize 配置线程池队列的最大值，使用 coreSize 配置核心线程池的最大值等。同时，我们也注意到可以使用 withExecutionTimeoutInMilliseconds 配置项来指定请求的超时时间。

虽然，上述代码有助于我们更好的理解 Hystrix 中线程池隔离实现机制，但在日常开发过程中，一般不建议你通过创建一个 HystrixCommand 子类的方式来实现服务隔离，而是推荐你使用更为简单的 @HystrixCommand 注解。@HystrixCommand 是 Hystrix 为简化开发过程而专门提供的一个注解，定义如下：

```java
public @interface HystrixCommand {

    String groupKey() default "";
    String commandKey() default "";
    String threadPoolKey() default "";
    String fallbackMethod() default "";
    HystrixProperty[] commandProperties() default {};
    HystrixProperty[] threadPoolProperties() default {};
    Class<? extends Throwable>[] ignoreExceptions() default {};
    ObservableExecutionMode observableExecutionMode() default ObservableExecutionMode.EAGER;
    HystrixException[] raiseHystrixExceptions() default {};
    String defaultFallback() default "";
}
```

在上述定义中，我们看到了用于设置分组、服务与线程池名称相关的 groupKey、commandKey 和 threadPoolKey方法，以及与线程池属性相关的 threadPoolProperties 对象。让我们回到案例，并使用 @HystrixCommand 注解进行重构，效果如下：

```java
@HystrixCommand
private UserMapper getUser(String userName) {
 
    return userClient.getUserByUserName(userName);
}
```

可以看到这里只使用了 @HystrixCommand 这个注解就完成 HystrixCommand 的创建。当然，我们也可以进一步使用 @HystrixProperty 注解来设置所需的各项属性，如下所示：

```java
@HystrixCommand(threadPoolKey = "springHealthGroup",
    threadPoolProperties =
     {
         @HystrixProperty(name="coreSize",value="2"),
         @HystrixProperty(name="maxQueueSize",value="10")
     }
)
private UserMapper getUser(String userName) {
 
    return userClient.getUserByUserName(userName);
}
```

同样可以看到，我们为该 HystrixCommand 设置了 threadPoolKey，也提供了 threadPoolProperties 来设置 coreSize 和 maxQueueSize

**使用 Hystrix 实现服务熔断**

在 HystrixCommand 中我们也可以对熔断的超时时间、失败率等各项阈值进行设置。例如我们可以在 getDevice() 方法上添加如下配置项以改变 Hystrix 的默认行为：

```java
@HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
})
private DeviceMapper getDevice(String deviceCode)
```

Hystrix 还提供了一系列的配置项来细化对熔断器的控制。常见的配置项如下所示：

```java
@HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "12000"),
            //一个滑动窗口内最小的请求数
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            //错误比率阈值
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "75"),
            //触发熔断的时间值
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "7000"),
            //一个滑动窗口的时间长度
            @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "15000"),
            //一个滑动窗口被划分的数量
            @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "5") })
```

**使用 Hystrix 实现服务回退**

Hystrix 在服务调用失败时都可以执行服务回退逻辑。在开发过程上，我们只需要提供一个 Fallback 方法实现并进行配置即可。

```java
private UserMapper getUserFallback(String userName) {
 
        UserMapper fallbackUser = new UserMapper(0L,"no_user","not_existed_user");

        return fallbackUser;
}
```

有了这个 Fallback 方法，剩下来要做的就是在 @HystrixCommand 注解中设置“fallbackMethod”配置项。重构后的 getUser 方法如下所示：

```java
@HystrixCommand(threadPoolKey = "springHealthGroup",
    threadPoolProperties =
         {
             @HystrixProperty(name="coreSize",value="2"),
             @HystrixProperty(name="maxQueueSize",value="10")
         },
         fallbackMethod = "getUserFallback"
)
private UserMapper getUser(String userName) {
 
        return userClient.getUserByUserName(userName);
}
```

