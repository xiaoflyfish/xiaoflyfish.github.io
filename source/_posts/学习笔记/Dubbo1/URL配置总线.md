---
title: URL配置总线
categories: 
- 学习笔记
- Dubbo1
---

在互联网领域，每个信息资源都有统一的且在网上唯一的地址，该地址就叫 URL（Uniform Resource Locator，统一资源定位符），它是互联网的统一资源定位标志，也就是指网络地址。

URL 本质上就是一个特殊格式的字符串。一个标准的 URL 格式可以包含如下的几个部分：

```
protocol://username:password@host:port/path?key=value&key=value
```

- protocol：URL 的协议。我们常见的就是 HTTP 协议和 HTTPS 协议，当然，还有其他协议，如 FTP 协议、SMTP 协议等。

- username/password：用户名/密码。 HTTP Basic Authentication 中多会使用在 URL 的协议之后直接携带用户名和密码的方式。

- host/port：主机/端口。在实践中一般会使用域名，而不是使用具体的 host 和 port。

- path：请求的路径。

- parameters：参数键值对。一般在 GET 请求中会将参数放到 URL 中，POST 请求会将参数放到请求体中。


URL 是整个 Dubbo 中非常基础，也是非常核心的一个组件，阅读源码的过程中你会发现很多方法都是以 URL 作为参数的，在方法内部解析传入的 URL 得到有用的参数，所以有人将 URL 称为Dubbo 的配置总线。

例如，你会看到 URL 参与了扩展实现的确定；你还会看到 Provider 将自身的信息封装成 URL 注册到 ZooKeeper 中，从而暴露自己的服务， Consumer 也是通过 URL 来确定自己订阅了哪些 Provider 的。

**Dubbo 中的 URL**

Dubbo 中任意的一个实现都可以抽象为一个 URL，Dubbo 使用 URL 来统一描述了所有对象和配置信息，并贯穿在整个 Dubbo 框架之中。这里我们来看 Dubbo 中一个典型 URL 的示例，如下：

```
dubbo://172.17.32.91:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=32508&release=&side=provider&timestamp=1593253404714dubbo://172.17.32.91:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=32508&release=&side=provider&timestamp=1593253404714
```

这个 Demo Provider 注册到 ZooKeeper 上的 URL 信息，简单解析一下这个 URL 的各个部分：

- protocol：dubbo 协议。

- username/password：没有用户名和密码。

- host/port：172.17.32.91:20880。

- path：`org.apache.dubbo.demo.DemoService`。

- parameters：参数键值对，这里是问号后面的参数。

下面是 URL 的构造方法，你可以看到其核心字段与前文分析的 URL 基本一致：

```java
public URL(String protocol, 
            String username, 
            String password, 
            String host, 
            int port, 
            String path, 
            Map<String, String> parameters, 
            Map<String, Map<String, String>> methodParameters) { 
    if (StringUtils.isEmpty(username) 
            && StringUtils.isNotEmpty(password)) { 
        throw new IllegalArgumentException("Invalid url"); 
    } 
    this.protocol = protocol; 
    this.username = username; 
    this.password = password; 
    this.host = host; 
    this.port = Math.max(port, 0); 
    this.address = getAddress(this.host, this.port); 
    while (path != null && path.startsWith("/")) { 
        path = path.substring(1); 
    } 
    this.path = path; 
    if (parameters == null) { 
        parameters = new HashMap<>(); 
    } else { 
        parameters = new HashMap<>(parameters); 
    } 
    this.parameters = Collections.unmodifiableMap(parameters); 
    this.methodParameters = Collections.unmodifiableMap(methodParameters); 
}
```

另外，在 dubbo-common 包中还提供了 URL 的辅助类：

- URLBuilder， 辅助构造 URL；

- URLStrParser， 将字符串解析成 URL 对象。

**Dubbo 中的 URL 示例**

了解了 URL 的结构以及 Dubbo 使用 URL 的原因之后，我们再来看 Dubbo 中的三个真实示例，进一步感受 URL 的重要性。

> URL 在 SPI 中的应用

Dubbo SPI 中有一个依赖 URL 的重要场景——适配器方法，是被 @Adaptive 注解标注的， URL 一个很重要的作用就是与 @Adaptive 注解一起选择合适的扩展实现类。

例如，在 dubbo-registry-api 模块中我们可以看到 RegistryFactory 这个接口，其中的 getRegistry() 方法上有 `@Adaptive({"protocol"}) `注解，说明这是一个适配器方法，Dubbo 在运行时会为其动态生成相应的 `“$Adaptive” `类型，如下所示：

```java
public class RegistryFactory$Adaptive
              implements RegistryFactory { 
    public Registry getRegistry(org.apache.dubbo.common.URL arg0) { 
        if (arg0 == null) throw new IllegalArgumentException("..."); 
        org.apache.dubbo.common.URL url = arg0; 
        // 尝试获取URL的Protocol，如果Protocol为空，则使用默认值"dubbo" 
        String extName = (url.getProtocol() == null ? "dubbo" : 
             url.getProtocol()); 
        if (extName == null) 
            throw new IllegalStateException("..."); 
        // 根据扩展名选择相应的扩展实现，Dubbo SPI的核心原理在下一课时深入分析 
        RegistryFactory extension = (RegistryFactory) ExtensionLoader 
          .getExtensionLoader(RegistryFactory.class) 
                .getExtension(extName); 
        return extension.getRegistry(arg0); 
    } 
}
```

我们会看到，在生成的` RegistryFactory$Adaptive` 类中会自动实现 getRegistry() 方法，其中会根据 URL 的 Protocol 确定扩展名称，从而确定使用的具体扩展实现类。我们可以找到 RegistryProtocol 这个类，并在其 getRegistry() 方法中打一个断点

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210307211758.png" style="zoom:25%;" />

这里传入的 registryUrl 值为：

```
zookeeper://127.0.0.1:2181/org.apache.dubbo...
```

那么在 `RegistryFactory$Adaptive` 中得到的扩展名称为 zookeeper，此次使用的 Registry 扩展实现类就是ZookeeperRegistryFactory

> URL 在服务暴露中的应用

Provider 在启动时，会将自身暴露的服务注册到 ZooKeeper 上，具体是注册哪些信息到 ZooKeeper 上呢

我们来看 `ZookeeperRegistry.doRegister()` 方法，在其中打个断点，然后 Debug 启动 Provider

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210307211922.png" style="zoom:25%;" />

传入的 URL 中包含了 Provider 的地址（172.18.112.15:20880）、暴露的接口（`org.apache.dubbo.demo.DemoService`）等信息， toUrlPath() 方法会根据传入的 URL 参数确定在 ZooKeeper 上创建的节点路径，还会通过 URL 中的 dynamic 参数值确定创建的 ZNode 是临时节点还是持久节点。

> URL 在服务订阅中的应用

Consumer 启动后会向注册中心进行订阅操作，并监听自己关注的 Provider

那 Consumer 是如何告诉注册中心自己关注哪些 Provider 呢

我们来看 ZookeeperRegistry 这个实现类，它是由上面的 ZookeeperRegistryFactory 工厂类创建的 Registry 接口实现，其中的 doSubscribe() 方法是订阅操作的核心实现，在第 175 行打一个断点，并 Debug 启动 Demo 中 Consumer

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210307212048.png" style="zoom:25%;" />

我们看到传入的 URL 参数如下：

```
consumer://...?application=dubbo-demo-api-consumer&category=providers,configurators,routers&interface=org.apache.dubbo.demo.DemoService...
```

其中 Protocol 为 consumer ，表示是 Consumer 的订阅协议，其中的 category 参数表示要订阅的分类，这里要订阅 providers、configurators 以及 routers 三个分类；interface 参数表示订阅哪个服务接口，这里要订阅的是暴露 `org.apache.dubbo.demo.DemoService` 实现的 Provider。

通过 URL 中的上述参数，ZookeeperRegistry 会在 toCategoriesPath() 方法中将其整理成一个 ZooKeeper 路径，然后调用 zkClient 在其上添加监听。
