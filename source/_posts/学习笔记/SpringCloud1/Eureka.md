---
title: Eureka
categories: 
- 学习笔记
- SpringCloud1
---

# 服务器

## 构建注册中心

```xml
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
	public static void main(String[] args) {
	 
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

Eureka 也为开发人员提供了一系列的配置项。这些配置项可以分成三大类，一类用于控制 Eureka 服务器端行为，以 `eureka.server` 开头；一类则是从客户端角度出发考虑配置需求，以` eureka.client `开头；而最后一类则关注于注册到 Eureka 的服务实例本身，以` eureka.instance `开头。请注意，Eureka 除了充当服务器端组件之外，实际上也可以作为客户端注册到 Eureka 本身，这时候它使用的就是客户端配置项。

```yml
server:
  port: 8761
 
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8761
```

registerWithEureka 用于指定是否把当前的客户端实例注册到 Eureka 服务器中，而 fetchRegistry 则用于指定是否从 Eureka 服务器上拉取已注册的服务信息

这两个配置项默认都是 true，但这里都将其设置为 false。因为在微服务体系中，包括 Eureka 服务在内的所有服务对于注册中心来说都可以算作客户端，而 Eureka 服务显然不同于业务服务，我们不希望 Eureka 服务对自身进行注册。而 serviceUrl 配置项用于服务地址，这个配置项在构建 Eureka 服务器集群是很有用

## 构建Eureka集群

如果我们把 Eureka 也视为一个服务，也就是说 Eureka服务自身也能注册到其他 Eureka 服务上，从而实现相互注册，并构成一个集群。在 Eureka中，这种实现高可用的部署方式被称为 Peer Awareness 模式。

现在我们准备两个 Eureka 服务实例 eureka1 和 eureka2。在 Spring Boot 中，我们分别提供` application-eureka1.yml` 和` application-eureka2.yml `这两个配置文件来设置相关的配置项

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: eureka1
  client
    serviceUrl
	   defaultZone: http:// eureka2:8762/eureka/
```

```yaml
server:
  port: 8762

eureka:
  instance:
    hostname: eureka2
  client
    serviceUrl
	   defaultZone: http://eureka1:8761/eureka/
```

## 实现原理

对于一个注册中心而言，我们首先需要关注它的数据存储方法。在 Eureka 中，我们发现 InstanceRegistry 接口及其实现类（位于` com.netflix.eureka.registry `包中）承接了这部分职能

Spring Cloud 中同样存在一个 InstanceRegistry（位于 `org.springframework.cloud.netflix.eureka.server` 包中），它实际上是基于 Netflix 中 InstanceRegistry 实现的一种包装

InstanceRegistry 接口的实现类 AbstractInstanceRegistry 中发现了 Eureka 用于保存注册信息的数据结构

```java
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```

可以看到这是一个双层的 HashMap，采用的是 JDK 中线程安全的 ConcurrentHashMap。其中第一层的 ConcurrentHashMap 的 Key 为` spring.application.name`，也就是服务名，Value 为一个 ConcurrentHashMap；而第二层的 ConcurrentHashMap 的 Key 为 instanceId，也就是服务的唯一实例 ID，Value 为 Lease 对象。

Eureka 采用 Lease（租约）这个词来表示对服务注册信息的抽象，Lease 对象保存了服务实例信息以及一些实例服务注册相关的时间，如注册时间 registrationTimestamp、最新的续约时间 lastUpdateTimestamp 等

而对于 InstanceRegistry 本身，它也继承了 Eureka 中两个非常重要的接口，即LeaseManager 接口和 LookupService 接口。其中 LeaseManager 接口定义如下：

```java
public interface LeaseManager<T> {
    void register(T r, int leaseDuration, boolean isReplication);
    boolean cancel(String appName, String id, boolean isReplication);
    boolean renew(String appName, String id, boolean isReplication);
    void evict();
}
```

显然 LeaseManager 做的事情就是 Eureka 注册中心模型中的服务注册、服务续约、服务取消和服务剔除等核心操作，关注于对服务注册过程的管理。而 LookupService 接口定义如下，关注于对应用程序与服务实例的管理：

```java
public interface LookupService<T> {
    Application getApplication(String appName);
    Applications getApplications();
    List<InstanceInfo> getInstancesById(String id);
    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);
}
```

在内部实现上，实际上对于注册中心服务器而言，服务注册、续约、取消和剔除等不同操作所执行的工作流程基本一致，即都是对服务存储的操作，并把这一操作同步到其他 Eureka 节点。我们这里选择用于服务注册操作的 register 方法进行展开，register 方法非常长，对源码进行裁剪，得出如下所示的核销处理流程：

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try { 
        //从已存储的 registry 获取一个服务定义
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        REGISTER.increment(isReplication);
        if (gMap == null) {
            //初始化一个 Map<String, Lease<InstanceInfo>> ，并放入 registry 中
        }
 
        //根据当前注册的 ID 找到对应的 Lease
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
 
        if (existingLease != null && (existingLease.getHolder() != null)) {
            //如果 Lease 能找到，根据当前节点的最新更新时间和注册节点的最新更新时间比较
            //如果前者的时间晚于后者的时间，那么注册实例就以已存在的实例为准
        } else {
              //如果找不到，代表是一个新注册，则更新其每分钟期望的续约数量及其阈值
        }
 
        //创建一个新 Lease 并放入 Map 中
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        gMap.put(registrant.getId(), lease);

        //处理服务的 InstanceStatus
        registrant.setActionType(ActionType.ADDED);
 
        //更新服务最新更新时间
        registrant.setLastUpdatedTimestamp();
 
        //刷选缓存
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
    } 
}
```

**Eureka服务缓存源码解析**

Eureka 服务器端组件的另一个核心功能是提供服务列表。为了提高性能，Eureka 服务器会缓存一份所有已注册的服务列表，并通过一定的定时机制对缓存数据进行更新。

我们知道为了获取注册到 Eureka 服务器上具体某一个服务实例的详细信息，可以访问如下地址：

```
http://<eureka-server-ip>:8761/eureka/apps/<APPID>
```

该地址代表的就是一个普通的 HTTP 请求，URL 中的 APPID 就是服务名称

该地址代表的就是一个普通的 HTTP GET 请求。Eureka 中所有对服务器端的访问都是通过RESTful 风格的资源（Resource） 进行获取，ApplicationResource 类（位于`com.netflix.eureka.resources` 包中）提供了根据应用获取注册信息的入口。我们来看该类的 getApplication 方法，核心代码如下所示：

```java
Key cacheKey = new Key(
       Key.EntityType.Application,
       appName,
       keyType,
       CurrentRequestVersion.get(),
       EurekaAccept.fromString(eurekaAccept)
);
 
String payLoad = responseCache.get(cacheKey);
 
if (payLoad != null) {
      logger.debug("Found: {}", appName);
      return Response.ok(payLoad).build();
} else {
      logger.debug("Not Found: {}", appName);
      return Response.status(Status.NOT_FOUND).build();
}
```

可以看到这里是构建了一个 cacheKey，并直接调用了` responseCache.get(cacheKey) `方法来返回一个字符串并构建响应。从命名上看，不难想象这里使用了缓存机制。我们来看 ResponseCache 的定义，如下所示，其中最核心的就是这里的 get 方法：

```java
public interface ResponseCache {
    void invalidate(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress);
 
    AtomicLong getVersionDelta();
    AtomicLong getVersionDeltaWithRegions();
    String get(Key key);
    byte[] getGZIP(Key key);
}
```

从类层关系上看，ResponseCache 只有一个实现类 ResponseCacheImpl，我们来看它的 get 方法，发现该方法使用了如下处理策略：

```java
Value getValue(final Key key, boolean useReadOnlyCache) {
        Value payload = null;
        try {
            if (useReadOnlyCache) {
                final Value currentPayload = readOnlyCacheMap.get(key);
                if (currentPayload != null) {
                    payload = currentPayload;
                } else {
                    payload = readWriteCacheMap.get(key);
                    readOnlyCacheMap.put(key, payload);
                }
            } else {
                payload = readWriteCacheMap.get(key);
            }
        } catch (Throwable t) {
            logger.error("Cannot get value for key : {}", key, t);
        }
        return payload;
}
```

可以看到上述代码中有两个缓存，一个是 readOnlyCacheMap，一个是 readWriteCacheMap。其中 readOnlyCacheMap 就是一个 JDK 中的 ConcurrentMap，而 readWriteCacheMap 使用的则是 Google Guava Cache 库中的 LoadingCache 类型。在创建 LoadingCache过程中，缓存数据的来源是调用 generatePayload 方法来生成。而在这个 generatePayload 方法中，就会调用前面介绍的 AbstractInstanceRegistry 中的 getApplications 方法获取应用信息并放到缓存中。这样我们就实现了把注册信息与缓存信息进行关联。

这里有一个设计和实现上的技巧。把缓存设计为一个只读的 readOnlyCacheMap 以及一个可读写的 readWriteCacheMap，可以更好地分离职责。但因为两个缓存中保存的实际上是同一份数据，所以，我们在不断更新 readWriteCacheMap 时，也需要确保 readOnlyCacheMap 中的数据得到同步。为此 ResponseCacheImpl 提供了一个定时任务 CacheUpdateTask，如下所示：

```java
private TimerTask getCacheUpdateTask() {
        return new TimerTask() {
            @Override
            public void run() {
                for (Key key : readOnlyCacheMap.keySet()) {
                    try {
                        CurrentRequestVersion.set(key.getVersion());
                        Value cacheValue = readWriteCacheMap.get(key);
                        Value currentCacheValue = readOnlyCacheMap.get(key);
                        if (cacheValue != currentCacheValue) {
                            readOnlyCacheMap.put(key, cacheValue);
                        }
                    } catch (Throwable th) {
                    }
                }
            }
        };
}
```

显然，这个定时任务主要是从 readWriteCacheMap 更新数据到 readOnlyCacheMap。

**Eureka 高可用源码解析**

我们在 InstanceRegistry 的类层结构中也已经看到了它的一个扩展接口 PeerAwareInstanceRegistry 以及该接口的实现类 PeerAwareInstanceRegistryImpl。

在 PeerAwareInstanceRegistryImpl 中同样存在一个 register 方法，如下所示：

```java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        super.register(info, leaseDuration, isReplication);
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

我们在这里看到了一个非常重要的replicateToPeers 方法，该方法作就是用来实现服务器节点之间的状态同步。replicateToPeers 方法的核心代码如下所示：

```java
for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
    //如何该 URL 代表主机自身，则不用进行注册
    if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
         continue;
    }
    replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
}
```

为了理解这个操作，我们首先需要理解 Eureka 中的集群模式，这部分代码位于` com.netflix.eureka.cluster` 包中，其中包含了代表节点的 PeerEurekaNode 和 PeerEurekaNodes 类，以及用于节点之间数据传递的 HttpReplicationClient 接口。而 replicateInstanceActionsToPeers 方法中则根据不同的 Action 来调用 PeerEurekaNode 的不同方法。例如，如果是 StatusUpdate Action，则会调动 PeerEurekaNode的statusUpdate 方法，而该方法又会执行如下代码

```java
replicationClient.statusUpdate(appName, id, newStatus, info);
```

这句代码完成了 PeerEurekaNode 之间的通信，而 replicationClient 是 HttpReplicationClient 接口的实例，该接口定义如下：

```java
public interface HttpReplicationClient extends EurekaHttpClient {
    EurekaHttpResponse<Void> statusUpdate(String asgName, ASGStatus newStatus);
 
    EurekaHttpResponse<ReplicationListResponse> submitBatchUpdates(ReplicationList replicationList);
}
```

HttpReplicationClient 接口继承自 EurekaHttpClient 接口，而 EurekaHttpClient 接口属于 Eureka 客户端组件

在这里，我们只需要明白 Eureka 提供了 JerseyReplicationClient（位于` com.netflix.eureka.transport `包下）这一基于 Jersey 框架实现的HttpReplicationClient。以 statusUpdate 方法为例，它的实现过程如下：

```java
@Override
public EurekaHttpResponse<Void> statusUpdate(String asgName, ASGStatus newStatus) {
        ClientResponse response = null;
        try {
            String urlPath = "asg/" + asgName + "/status";
            response = jerseyApacheClient.resource(serviceUrl)
                    .path(urlPath)
                    .queryParam("value", newStatus.name())
                    .header(PeerEurekaNode.HEADER_REPLICATION, "true")
                    .put(ClientResponse.class);
            return EurekaHttpResponse.status(response.getStatus());
        } finally {
            if (response != null) {
                response.close();
            }
        }
}
```

这是典型的基于 Resource 的 RESTful 风格的调用方法，用到了 ApacheHttpClient4 工具类

# 客户端

## 实现服务注册

```xml
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaClient
public class UserApplication {
	public static void main(String[] args) {
	 
        SpringApplication.run(UserApplication.class, args);
    }
}
```

```yaml
spring:
  application:
	name: userservice 
server:
  port: 8081
	 
eureka:
  client:
    serviceUrl:
	  defaultZone: http://localhost:8761/eureka/
```

## 基本原理

在 Netflix Eureka 中，专门提供了一个客户端包，并抽象了一个客户端接口 EurekaClient。EurekaClient 接口继承自 LookupService 接口

这个 LookupService 接口实际上也是InstanceRegistry 接口的父接口。EurekaClient 在 LookupService 接口的基础上提供了一系列扩展方法，这些扩展方法并不是重点，我们还是更应该关注于它的类层机构

可以看到 EurekaClient 接口有个实现类 DiscoveryClient（位于 `com.netflix.discovery `包中），该类包含了服务提供者和服务消费者的核心处理逻辑，同时提供了我们在介绍 Eureka 服务器端基本原理时所介绍的 register、renew 等方法。DiscoveryClient 类的实现非常复杂，我们重点关注它构造方法中的这行代码：

```java
initScheduledTasks();
```

通过分析该方法中的代码，我们看到系统在这里初始化了一批调度任务，具体包含缓存刷新 cacheRefresh、心跳 heartbeat、服务实例复制 InstanceInfoReplicator 等，其中缓存刷新面向服务消费者，而心跳和服务实例复制面向服务提供者。

**服务提供者操作源码解析**

服务提供者关注服务注册、服务续约和服务下线等功能，它可以使用 Eureka 服务器提供的 RESTful API 完成上述操作。

在 DiscoveryClient 类中，服务注册操作由register 方法完成，如下所示。为了简单起见，我们对代码进行了裁剪，省略了日志相关等非核心代码：

```java
boolean register() throws Throwable {
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            throw e;
        }

        return httpResponse.getStatusCode() == 204;
}
```

上述 register 方法会在 InstanceInfoReplicator 类的 run 方法中进行执行。从操作流程上讲，上述代码的逻辑非常简单，即服务提供者先将自己注册到 Eureka 服务器中，然后根据返回的结果确定操作是否成功。显然，这里的重点代码是`eurekaTransport.registrationClient.register()`，DiscoveryClient 通过这行代码发起了远程请求。

首先我们来看 EurekaTransport 类，这是 DiscoveryClient 类中的一个内部类，定义了 registrationClient 变量用于实现服务注册。registrationClient 的类型是 EurekaHttpClient 接口，该接口的定义如下：

```java
public interface EurekaHttpClient {
    EurekaHttpResponse<Void> register(InstanceInfo info);
    EurekaHttpResponse<Void> cancel(String appName, String id);
    EurekaHttpResponse<InstanceInfo> sendHeartBeat(String appName, String id, InstanceInfo info, InstanceStatus overriddenStatus);
    EurekaHttpResponse<Void> statusUpdate(String appName, String id, InstanceStatus newStatus, InstanceInfo info);
    EurekaHttpResponse<Void> deleteStatusOverride(String appName, String id, InstanceInfo info);
    EurekaHttpResponse<Applications> getApplications(String... regions);
    EurekaHttpResponse<Applications> getDelta(String... regions);
    EurekaHttpResponse<Applications> getVip(String vipAddress, String... regions);
    EurekaHttpResponse<Applications> getSecureVip(String secureVipAddress, String... regions);
    EurekaHttpResponse<Application> getApplication(String appName);
    EurekaHttpResponse<InstanceInfo> getInstance(String appName, String id);
    EurekaHttpResponse<InstanceInfo> getInstance(String id);
    void shutdown();
}
```

可以看到这个 EurekaHttpClient 接口定义了 Eureka 服务器的一些底层 REST API，包括 register、cancel、sendHeartBeat、statusUpdate、getApplications 等。

在 Eureka 中，关于如何实现客户端与服务器端的远程通信，从工作原理上讲只是一个 RESTful 风格的 HTTP 请求，但在具体设计和实现上可以说是非常考究，因此类层结构上也比较复杂。我们先来看 EurekaHttpClient 接口的一个实现类 EurekaHttpClientDecorator，从命名上看它是一个装饰器（Decorator），如下所示：

```java
public abstract class EurekaHttpClientDecorator implements EurekaHttpClient {
 
    public enum RequestType {
        Register,
        Cancel,
        SendHeartBeat,
        StatusUpdate,
        DeleteStatusOverride,
        GetApplications,
        …
    }
 
    public interface RequestExecutor<R> {
        EurekaHttpResponse<R> execute(EurekaHttpClient delegate);
 
        RequestType getRequestType();
    }
 
    protected abstract <R> EurekaHttpResponse<R> execute(RequestExecutor<R> requestExecutor);
 
    @Override
    public EurekaHttpResponse<Void> register(final InstanceInfo info) {
        return execute(new RequestExecutor<Void>() {
            @Override
            public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
                return delegate.register(info);
            }
 
            @Override
            public RequestType getRequestType() {
                return RequestType.Register;
            }
        });
	    }
	 
	//省略其他方法实现
}
```

可以看到 EurekaHttpClientDecorator 通过定义一个抽象方法 execute(RequestExecutor requestExecutor) 来包装 EurekaHttpClient，这种包装是代理机制的一种表现形式。


然后我们再来看如何构建一个 EurekaHttpClient，Eureka 也专门提供了 EurekaHttpClientFactory 类来负责构建具体的 EurekaHttpClient。显然，这是工厂模式的一种典型应用。EurekaHttpClientFactory 接口定义如下：

```java
public interface EurekaHttpClientFactory {
    EurekaHttpClient newClient();
    void shutdown();
}
```

Eureka 中存在一批 EurekaHttpClientFactory 的实现类，包括 RetryableEurekaHttpClient 和 MetricsCollectingEurekaHttpClient 等，这些类都位于` com.netflix.discovery.shared.transport.decorator `包下。同时，在 `com.netflix.discovery.shared.transport` 包下，还存在一个 EurekaHttpClients 工具类，能够创建通过 RedirectingEurekaHttpClient、RetryableEurekaHttpClient、SessionedEurekaHttpClient 包装之后的 EurekaHttpClient。如下所示：

```java
new EurekaHttpClientFactory() {
            @Override
            public EurekaHttpClient newClient() {
                return new SessionedEurekaHttpClient(
                        name,
                        RetryableEurekaHttpClient.createFactory(
                                name,
                                transportConfig,
                                clusterResolver,
                                RedirectingEurekaHttpClient.createFactory(transportClientFactory),
                                ServerStatusEvaluators.legacyEvaluator()),
                        transportConfig.getSessionedClientReconnectIntervalSeconds() * 1000
                );
            }
};
```

这是 EurekaHttpClient 创建过程中的一条分支，即通过包装器对请求过程进行层层封装和代理。而在执行远程请求时，Eureka 同样提供了另一套体系来完成真正的远程调用，原始的 EurekaHttpClient 通过 TransportClientFactory 进行创建。TransportClientFactory 接口定义如下：

```java
public interface TransportClientFactory {
    EurekaHttpClient newClient(EurekaEndpoint serviceUrl);
    void shutdown();
}
```

TransportClientFactory 同样存在一批实现类，其中有些是实名类，有些是匿名类。以实名的实现类 JerseyEurekaHttpClientFactory 为例，它位于` com.netflix.discovery.shared.transport.jersey `包下，通过 EurekaJerseyClient 获取 Jersey 客户端，而 EurekaJerseyClient 又会使用 ApacheHttpClient4 对象，从而完成 REST 调用。

作为总结，这里也给你分享一个 Eureka 在设计和实现上的技巧，也就是所谓的**高阶（High Level）API和低阶（Low Level）API**

针对高阶 API，主要是通过装饰器模式进行一系列包装，从而创建目标 EurekaHttpClient。而关于低阶 API 的话，主要是 HTTP 远程调用的实现，Netflix 提供的是基于 Jersey 的版本，而 Spring Cloud 则提供了基于 RestTemplate 的版本

**服务消费者操作源码解析**

Eureka 客户端通过定时任务完成缓存刷新操作，我们已经在前面的内容中提到 DiscoveryClient 中的 initScheduledTasks 方法用于初始化各种调度任务，对于缓存刷选而言，调度器的初始化过程如下所示：

```java
if (clientConfig.shouldFetchRegistry()) {
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
}
```

显然，这里启动了一个调度任务并通过 CacheRefreshThread 线程完成具体操作。CacheRefreshThread 线程定义如下：

```java
class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
}
```

对于服务消费者而言，最重要的操作就是获取服务注册信息。在这里的 refreshRegistry 方法中，我们发现在进行一系列的校验之后，最终调用了 fetchRegistry 方法以完成注册信息的更新，该方法代码如下。为了简单起见，我们对代码进行了部分裁剪，只保留主流程：

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        try {
            // 获取应用
            Applications applications = getApplications();
            if (…) //如果满足全量拉取条件
            {
              // 全量拉取服务实例数据
                getAndStoreFullRegistry();
            } else {
              // 增量拉取服务实例数据
                getAndUpdateDelta(applications);
            } 
           // 重新计算和设置一致性hashcode
	applications.setAppsHashCode(applications.getReconcileHashCode());

        } 
 
        // 刷新本地缓存
        onCacheRefreshed();
        // 更新远程服务实例运行状态
        updateInstanceRemoteStatus();
 
        return true;
}
```

这里的几个带注释的方法都非常有用，因为 getAndStoreFullRegistry 的逻辑相对比较简单，我们将重点介绍 getAndUpdateDelta 方法，以便学习在 Eureka 中如何实现增量数据更新的设计技巧。裁剪之后的 getAndUpdateDelta 方法代码如下所示：

```java
private void getAndUpdateDelta(Applications applications) throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();
 
        Applications delta = null;
        //通过 eurekaTransport.queryClient 获取增量信息
        EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            delta = httpResponse.getEntity();
        }
 
        if (delta == null) {
              //如果增量信息为空，就直接发起一次全量更新
            getAndStoreFullRegistry();
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {//通过CAS来确保请求的线程安全性
            String reconcileHashCode = "";
            if (fetchRegistryUpdateLock.tryLock()) {
                try {
                 //比对从服务器端返回的增量数据和本地数据，合并两者的差异数据
                    updateDelta(delta);
 
	//用合并了增量数据之后的本地数据来生成一致性 hashcode
                    reconcileHashCode = getReconcileHashCode(applications);
                } finally {
                    fetchRegistryUpdateLock.unlock();
                }
            } else {
            }
 
	//比较本地数据中的 hashcode 和来自服务器端的 hashcode
            if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
                 //如果 hashcode 不一致，就触发远程调用进行全量更新
                reconcileAndLogDifference(delta, reconcileHashCode);
            }
        } else {
        }
}
```

回顾 Eureka 服务器端基本原理，我们知道 Eureka 服务器端会保存一个服务注册列表的缓存。Eureka 官方文档中提到这个数据保留时间是三分钟，而 Eureka 客户端的定时调度机制会每隔 30 秒刷选本地缓存。原则上，只要 Eureka 客户端不停地获取服务器端的更新数据，就能保证自己的数据和 Eureka 服务器端的保持一致。但如果客户端在 3 分钟之内没有获取更新数据，就会导致自身与服务器端的数据不一致，这是这种更新机制所必须要考虑的问题，也是我们自己在设计类似场景时的一个注意点。

针对上述问题，Eureka 采用了**一致性 HashCode** 方法来进行解决。Eureka 服务器端每次返回的增量数据中都会带有一个一致性 HashCode，这个 HashCode 会与 Eureka 客户端用本地服务列表数据算出的一致性 HashCode 进行比对，如果两者不一致就证明增量更新出了问题，这时候就需要执行一次全量更新。

在 Eureka 中，计算一致性 HashCode 的方法如下所示，可以看到这一方法基于服务注册实例信息完成编码计算过程，最终返回一个 String 类型的计算结果：

```java
public static String getReconcileHashCode(Map<String, AtomicInteger> instanceCountMap) {
        StringBuilder reconcileHashCode = new StringBuilder(75);
        for (Map.Entry<String, AtomicInteger> mapEntry : instanceCountMap.entrySet()) {
            reconcileHashCode.append(mapEntry.getKey()).append(STATUS_DELIMITER).append(mapEntry.getValue().get())
                    .append(STATUS_DELIMITER);
        }
        return reconcileHashCode.toString();
}
```

作为总结，Eureka 客户端缓存定时更新的流程如下图所示，可以看到它与服务注册的流程基本一致，也就是说在 Eureka 中，服务提供者和服务消费者作为 Eureka 服务器的客户端采用了同一套体系完成与服务器端的交互

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210217142503.png" style="zoom:33%;" />