---
title: Ribbon
categories: 
- 学习笔记
- SpringCloud1
---

# 负载均衡

基于 Spring Cloud Netflix Ribbon，通过注解就能简单实现在面向服务的接口调用中，自动集成负载均衡功能，使用方式主要包括以下两种：

**使用 @LoadBalanced 注解。**

@LoadBalanced 注解用于修饰发起 HTTP 请求的 RestTemplate 工具类，并在该工具类中自动嵌入客户端负载均衡功能。开发人员不需要针对负载均衡做任何特殊的开发或配置。

**使用 @RibbonClient 注解。**

Ribbon 还允许你使用 @RibbonClient 注解来完全控制客户端负载均衡行为。这在需要定制化负载均衡算法等某些特定场景下非常有用，我们可以使用这个功能实现更细粒度的负载均衡配置。

事实上，无论使用哪种方法，我们首先需要明确如何通过 Eureka 提供的 DiscoveryClient 工具类查找注册在 Eureka 中的服务，这是 Ribbon 实现客户端负载均衡的基础。

**使用 DiscoveryClient 获取服务实例信息**

基于这一点，假如现在没有 Ribbon 这样的负载均衡工具，我们也可以通过代码在运行时实时获取注册中心中的服务列表，并通过服务定义并结合各种负载均衡策略动态发起服务调用。

接下来，让我们来演示如何根据服务名称获取 Eureka 中的服务实例信息。通过 DiscoveryClient 可以很容易实现这一点。

首先，我们获取当前注册到 Eureka 中的服务名称全量列表，如下所示：

```java
List<String> serviceNames = discoveryClient.getServices();
```

基于这个服务名称列表可以获取所有自己感兴趣的服务，并进一步获取这些服务的实例信息：

```java
List<ServiceInstance> serviceInstances = discoveryClient.getInstances(serviceName);
```

ServiceInstance 对象代表服务实例，包含了很多有用的信息，定义如下：

```java
public interface ServiceInstance {
    //服务实例的唯一性 Id
    String getServiceId();
    //主机
    String getHost();
	//端口
    int getPort();
	//URI
    URI getUri();
	//元数据
    Map<String, String> getMetadata();
	…
}
```

显然，一旦获取了一个 ServiceInstance 列表，我们就可以基于常见的随机、轮询等算法来实现客户端负载均衡，也可以基于服务的 URI 信息等实现各种定制化的路由机制。一旦确定负载均衡的最终目标服务，就可以使用 HTTP 工具类来根据服务的地址信息发起远程调用。

在 Spring 的世界中，访问 HTTP 端点最常见的方法就是使用 RestTemplate 工具类

```java
@Autowired
RestTemplate restTemplate;
 
@Autowired
private DiscoveryClient discoveryClient;
 
public User getUserByUserName(String userName) {
        List<ServiceInstance> instances = discoveryClient.getInstances("userservice");
 
        if (instances.size()==0) 
         return null;

        String userserviceUri = String.format("%s/users/%s",instances.get(0).getUri()
	.toString(),userName);

        ResponseEntity<User> user =
            restTemplate.exchange(userserviceUri, HttpMethod.GET, null, User.class, userName);

        return result.getBody();
}
```

可以看到，这里通过 RestTemplate 工具类就可以使用 ServiceInstance 中的 URL 轻松实现 HTTP 请求。在上面的示例代码中，我们通过` instances.get(0) `方法获取的是服务列表中的第一个服务，然后使用 RestTemplate 的 exchange() 方法封装整个 HTTP 请求调用过程并获取结果

**通过 @Loadbalanced 注解调用服务**

```java
@SpringBootApplication
@EnableEurekaClient
public class InterventionApplication {
 
    @LoadBalanced
    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
 
    public static void main(String[] args) {
        SpringApplication.run(InterventionApplication.class, args);
    }
}
```

**通过 @RibbonClient 注解自定义负载均衡策略**

在基于 @LoadBalanced 注解执行负载均衡时，采用的是 Ribbon 内置的负载均衡机制。默认情况下，Ribbon 使用的是轮询策略，我们无法控制具体生效的是哪种负载均衡算法。

但在有些场景下，我们就需要对负载均衡这一过程进行更加精细化的控制，这时候就可以用到 @RibbonClient 注解。Spring Cloud Netflix Ribbon 提供 @RibbonClient 注解的目的在于通过该注解声明自定义配置，从而来完全控制客户端负载均衡行为。@RibbonClient 注解的定义如下：

```java
public @interface RibbonClient {
    //同下面的 name 属性
    String value() default "";
    //指定服务名称
    String name() default "";
	//指定负载均衡配置类
    Class<?>[] configuration() default {};
}
```

通常，我们需要指定这里的目标服务名称以及负载均衡配置类。所以，为了使用 @RibbonClient 注解，我们需要创建一个独立的配置类，用来指定具体的负载均衡规则。以下代码演示的就是一个自定义的配置类 SpringHealthLoadBalanceConfig：

```java
@Configuration
public class SpringHealthLoadBalanceConfig{
 
    @Autowired
    IClientConfig config;
 
    @Bean
    @ConditionalOnMissingBean
    public IRule springHealthRule(IClientConfig config) {

        return new RandomRule();
    }
}
```

显然该配置类的作用是使用 RandomRule 替换 Ribbon 中的默认负载均衡策略 RoundRobin。我们可以根据需要返回任何自定义的 IRule 接口的实现策略

有了这个 SpringHealthLoadBalanceConfig 之后，我们就可以在调用特定服务时使用该配置类，从而对客户端负载均衡实现细粒度的控制

```java
@SpringBootApplication
@EnableEurekaClient
@RibbonClient(name = "userservice", configuration = SpringHealthLoadBalanceConfig.class)
public class InterventionApplication{
 
	@Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
 
    public static void main(String[] args) {
        SpringApplication.run(InterventionApplication.class, args);
    }
}
```

可以注意到，我们在 @RibbonClient 中设置了目标服务名称为 userservice，配置类为 SpringHealthLoadBalanceConfig。现在每次访问 user-service 时将使用 RandomRule 这一随机负载均衡策略。

对比 @LoadBalanced 注解和 @RibbonClient 注解，如果使用的是普通的负载均衡场景，那么通常只需要 @LoadBalanced 注解就能完成客户端负载均衡。而如果我们要对 Ribbon 运行时行为进行定制化处理时，就可以使用 @RibbonClient 注解

# 实现原理

**Netflix Ribbon 中的核心类**

Netflix Ribbon 的核心接口 ILoadBalancer ，接口位于` com.netflix.loadbalancer `包下，定义如下：

```java
public interface ILoadBalancer {
  //添加后端服务
  public void addServers(List<Server> newServers);
 
  //选择一个后端服务
  public Server chooseServer(Object key); 
 
	//标记一个服务不可用
  public void markServerDown(Server server);
 
	//获取当前可用的服务列表
  public List<Server> getReachableServers();
	 
	//获取所有后端服务列表
   public List<Server> getAllServers();
}
```

其中 AbstractLoadBalancer 是个抽象类，只定义了两个抽象方法，并不构成一种模板方法的结构。所以我们直接来看 ILoadBalancer 接口，该接口最基本的实现类是 BaseLoadBalancer，可以说负载均衡的核心功能都可以在这个类中得以实现。

我们先来梳理 BaseLoadBalancer 包含的作为一个负载均衡器应该具备的一些核心组件，比较重要的有以下三个。

**IRule**

IRule 接口是对负载均衡策略的一种抽象，可以通过实现这个接口来提供各种适用的负载均衡算法

```java
public interface IRule{
    public Server choose(Object key);
    public void setLoadBalancer(ILoadBalancer lb);
    public ILoadBalancer getLoadBalancer();
}
```

显然 choose 方法是该接口的核心方法

**IPing**

IPing 接口判断目标服务是否存活，定义如下：

```java
public interface IPing {
    public boolean isAlive(Server server);
}
```

可以看到 IPing 接口中只有一个 isAlive() 方法，通过对服务发出"Ping"操作来获取服务响应，从而判断该服务是否可用。

**LoadBalancerStats**

LoadBalancerStats 类记录负载均衡的实时运行信息，用来作为负载均衡策略的运行时输入。

注意，在 BaseLoadBalancer 内部维护着 allServerList 和 upServerList 这两个线程的安全列表，所以对于 ILoadBalancer 接口定义的 addServers、getReachableServers、getAllServers 这几个方法而言，主要就是对这些列表的维护和管理工作。以 addServers 方法为例，它的实现如下所示：

```java
public void addServers(List<Server> newServers) {
        if (newServers != null && newServers.size() > 0) {
            try {
                ArrayList<Server> newList = new ArrayList<Server>();
                newList.addAll(allServerList);
                newList.addAll(newServers);
                setServersList(newList);
            } catch (Exception e) {
                logger.error("LoadBalancer [{}]: Exception while adding Servers", name, e);
            }
        }
}
```

显然，这里的处理过程就是将原有的服务实例列表 allServerList 和新传入的服务实例列表 newServers 都合并到一个 newList 中，然后再调用 setServersList 方法用这个新的列表覆盖旧的列表。

针对负载均衡，我们重点应该关注的是 ILoadBalancer 接口中 chooseServer 方法的实现，不难想象该方法肯定通过前面介绍的 IRule 接口集成了具体负载均衡策略的实现。在 BaseLoadBalancer 中的 chooseServer 方法如下所示：

```java
public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                return rule.choose(key);
            } catch (Exception e) {
                 return null;
            }
        }
}
```

果然，这里使用了 IRule 接口的 choose 方法。

**Netflix Ribbon 中的负载均衡策略**

一般而言，负载均衡算法可以分成两大类，即静态负载均衡算法和动态负载均衡算法。静态负载均衡算法比较容易理解和实现，典型的包括随机（Random）、轮询（Round Robin）和加权轮询（Weighted Round Robin）算法等。

所有涉及权重的静态算法都可以转变为动态算法，因为权重可以在运行过程中动态更新。例如动态轮询算法中权重值基于对各个服务器的持续监控并不断更新。另外，基于服务器的实时性能分析分配连接是常见的动态策略。典型动态算法包括源 IP 哈希算法、最少连接数算法、服务调用时延算法等。

Netflix Ribbon 中的负载均衡实现策略非常丰富，既提供了 RandomRule、RoundRobinRule 等无状态的静态策略，又实现了 AvailabilityFilteringRule、WeightedResponseTimeRule 等多种基于服务器运行状况进行实时路由的动态策略。

还看到了 RetryRule 这种重试策略，该策略会对选定的负载均衡策略执行重试机制。严格意义上讲重试是一种服务容错而不是负载均衡机制，但 Ribbon 也内置了这方面的功能。

静态的几种策略相对都比较简单，而像 RetryRule 实际上不算是严格意义上的负载均衡策略，这里重点关注 Ribbon 所实现的几种不同的动态策略。

> BestAvailableRule 策略

选择一个并发请求量最小的服务器，逐个考察服务器然后选择其中活跃请求数最小的服务器。

> WeightedResponseTimeRule 策略

该策略与请求的响应时间有关，显然，如果响应时间越长，就代表这个服务的响应能力越有限，那么分配给该服务的权重就应该越小。而响应时间的计算就依赖于前面介绍的 ILoadBalancer 接口中的 LoadBalancerStats。

WeightedResponseTimeRule 会定时从 LoadBalancerStats 读取平均响应时间，为每个服务更新权重。权重的计算也比较简单，即每次请求的响应时间减去每个服务自己平均的响应时间就是该服务的权重。

> AvailabilityFilteringRule 策略

通过检查 LoadBalancerStats 中记录的各个服务器的运行状态，过滤掉那些处于一直连接失败或处于高并发状态的后端服务器。

**Spring Cloud Netflix Ribbon**

Spring Cloud Netflix Ribbon 相当于 Netflix Ribbon 的客户端。而对于 Spring Cloud Netflix Ribbon 而言，我们的应用服务相当于它的客户端

这次，我们打算从应用服务层的 @LoadBalanced 注解入手，切入 Spring Cloud Netflix Ribbon，然后再从 Spring Cloud Netflix Ribbon 串联到 Netflix Ribbon，从而形成整个负载均衡闭环管理。

**@LoadBalanced 注解**

为什么通过 @LoadBalanced 注解创建的 RestTemplate 就能自动具备客户端负载均衡的能力？

事实上，在 Spring Cloud Netflix Ribbon 中存在一个自动配置类——LoadBalancerAutoConfiguration 类。而在该类中，维护着一个被 @LoadBalanced 修饰的 RestTemplate 对象的列表。在初始化的过程中，对于所有被 @LoadBalanced 注解修饰的 RestTemplate，调用 RestTemplateCustomizer 的 customize 方法进行定制化，该定制化的过程就是对目标 RestTemplate 增加拦截器 LoadBalancerInterceptor，如下所示：

```java
@Configuration
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
static class LoadBalancerInterceptorConfig {
        @Bean
        public LoadBalancerInterceptor ribbonInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) {
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }
 
        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) {
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
        }
}
```

这个 LoadBalancerInterceptor 用于实时拦截，可以看到它的构造函数中传入了一个对象 LoadBalancerClient，而在它的拦截方法本质上就是使用 LoadBalanceClient 来执行真正的负载均衡。LoadBalancerInterceptor 类代码如下所示：

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
 
    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;
 
    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
        this.loadBalancer = loadBalancer;
        this.requestFactory = requestFactory;
    }
 
    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
    }
 
    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
    }
}
```

可以看到这里的拦截方法 intercept 直接调用了 LoadBalancerClient 的 execute 方法完成对请求的负载均衡执行。

**LoadBalanceClient 接口**

LoadBalancerClient 是一个非常重要的接口，定义如下：

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
 
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
 
    <T> T execute(String serviceId, ServiceInstance serviceInstance,
           LoadBalancerRequest<T> request) throws IOException;
 
    URI reconstructURI(ServiceInstance instance, URI original);
}
```

这里有两个 execute 重载方法，用于根据负载均衡器所确定的服务实例来执行服务调用。而 reconstructURI 方法则用于构建服务 URI，使用负载均衡所选择的 ServiceInstance 信息重新构造访问 URI，也就是用服务实例的 host 和 port 再加上服务的端点路径来构造一个真正可供访问的服务。

LoadBalancerClient 继承自 ServiceInstanceChooser 接口，该接口定义如下：

```java
public interface ServiceInstanceChooser {
 
    ServiceInstance choose(String serviceId);
}
```

从负载均衡角度讲，我们应该重点关注实际上是这个 choose 方法的实现，而提供具体实现的是实现了 LoadBalancerClient 接口的 RibbonLoadBalancerClient，而 RibbonLoadBalancerClient 位于 spring-cloud-netflix-ribbon 工程中。这样我们的代码流程就从应用程序转入到了 Spring Cloud Netflix Ribbon 中。

在 LoadBalancerClient 接口的实现类 RibbonLoadBalancerClient 中，choose 方法最终调用了如下所示的 getServer 方法：

```java
protected Server getServer(ILoadBalancer loadBalancer) {
        if (loadBalancer == null) {
            return null;
        }
        return loadBalancer.chooseServer("default"); 
}
```

这里的 loadBalancer 对象就是前面介绍的 Netflix Ribbon 中的 ILoadBalancer 接口的实现类。这样，我们就把 Spring Cloud Netflix Ribbon 与 Netflix Ribbon 的整体协作流程串联起来