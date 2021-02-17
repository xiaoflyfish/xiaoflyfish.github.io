---
title: Zuul
categories: 
- 学习笔记
- SpringCloud1
---

在 Spring Cloud 中，针对 API 网关的实现提供了两种解决方案。

一种是集成 Netflix 中的 Zuul 网关，一种是自研 Spring Cloud Gateway

# 构建API网关

```xml
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulServerApplication {
	 
	public static void main(String[] args) {
        SpringApplication.run(ZuulServerApplication.class, args);
     }
}
```

```yaml
server:
  port: 5555
	 
eureka:
  instance:
    preferIpAddress: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
	   defaultZone: http://localhost:8761/eureka/
```

另一方面，API 网关在其服务的消费者和提供者之间提供了一层反向代理，充当着前置负载均衡器的角色。

所以，API 网关的定位决定了 Zuul 需要依赖 Ribbon。而 Zuul 可以与 Ribbon 完成无缝集成

**如何使用 Zuul 实现服务路由**

对于 API 网关而言，最重要的功能就是服务路由，即通过 Zuul 访问的请求会路由并转发到对应的后端服务中。通过 Zuul 进行服务访问的 URL 通用格式如下所示：

```
http://zuulservice:5555/service
```

其中 zuulservice 代表 Zuul 服务器的地址。而这里的 service 所对应的后端服务的确定就需要依赖位于 Zuul 中的服务路由信息

**基于服务发现映射服务路由**

Zuul 可以基于注册中心的服务发现机制实现自动化服务路由功能。所以使用 Zuul 实现服务路由最常见的、也最推荐的做法就是利用这种自动化的路由映射关系来确定路由信息。

从开发角度讲，系统自动映射也最简单。我们不需要做任何事情，因为在 Eureka 中已经保存了各种服务定义信息。而服务定义信息中包含了各个服务的名称，所以 Zuul 就可以把这些服务的名称与目标服务进行自动匹配。匹配的规则就是直接将目标服务映射到服务名称上。

例如，我们在 user-service 的配置文件中通过以下配置项指定了该服务的名称为 userservice：

```yaml
spring:
   application:
	   name: userservice
```

这样我们就可以通过http://zuulservice:5555/userservice访问到该服务，注意该 URL 中的目标服务就是 userservice，与服务定义中的名称保持一致。

在 Zuul 启动的过程中，会从 Eureka 中获取当前所有已注册的服务信息，然后自动生成服务名称与目标服务之间的映射关系

现在，让我们来演示一下。我们先后启动 eureka-server、user-service 和 zuul-service，然后访问以下地址：

```
http://localhost:5555/actuator/routes
```

这时候可以看到如下所示的键值对信息，这些键值对信息实际上就是服务路由映射信息：

```json
{
	  "/userservice/**":"userservice"
}
```

http://localhost:5555/actuator/routes：是 Zuul 提供的服务路由端点，展示了目前在 Zuul 中配置的服务路由信息。其中`"/userservice/**"中后半部分的“**”`代表所有访问该路径以及子路径的请求都将被自动路由到注册在 Eureka 中的名为 userservice 的某一个服务实例中。

当采用这种系统自动映射方式时，如果我们注册一个新服务或下线某个已有服务，那么这份映射列表也会做相应的调整。整个过程对于开发人员完全透明可见。

**基于动态配置映射服务路由**

基于服务发现机制的系统自动映射非常方便，但也有明显的局限性。在日常开发过程中，我们往往对服务映射关系有更多的定制化需求，比方说不使用默认的服务名称来命名目标服务，或者在各个请求路径之前加一个统一的前缀（Prefix）等。Zuul 充分考虑到了这些需求，开发和运维人员可以通过配置实现服务路由的灵活映射。

首先我们可以在 zuul-server 工程下的` application.yml `配置文件中为 user-service 配置特定的服务名称与请求地址之间的映射关系，如下所示：

```yaml
zuul:
   routes:
     userservice: /user/**
```

注意，这里我们使用 /user 来为 user-service 指定请求根地址。现在我们访问 http://zuulservice:5555/user/ 时，就相当于将请求发送给了 Eureka 中的 userservice 实例。

现在我们重启 zuul-server 并访问 http://localhost:5555/actuator/routes” 端点，得到的服务路由映射关系就变成了如下结果：

```json
{
	   "/user/**":"userservice",
	  "/userservice/**":"userservice"
}
```

可以看到在原有路由信息的基础上，Zuul 生成了一条新的路由信息，对应配置文件中的配置。现在，相当于将访问某一个服务的入口变成了两个：一个是系统自动映射的路由，一个是通过配置所生成的路由。当我们不希望系统自动映射的路由被外部使用时，我们就可以通过` ignored-services `配置项把它们从服务路由中去掉。再次以 userservice 为例，ignored-services 配置项的使用方法如下所示

```yaml
zuul:
   routes:
     ignored-services: 'userservice'
     userservice: /user/**
```

让我们考虑另外一个比较常见的应用场景，在一个大型的微服务架构中，可能会有非常多的微服务。这就需要对这些服务进行全局性的规划，可以通过模块或子系统的方式进行管理。表现在路由信息上，在各个服务请求地址上添加一个前缀用来标识模块和子系统是一项最佳实践。针对这种场景，就可以用到 Zuul 提供的“prefix”配置项，示例如下所示：

```yaml
zuul:
	prefix:  /springhealth
    routes:
      ignored-services: 'userservice'
       userservice: /user/**
```

我们将“prefix”配置项设置为“/springhealth”，代表所有配置的路由请求地址之前都会自动添加“/springhealth”前缀，用来识别这些请求的都属于 springhealth 微服务系统。

现在访问 http://localhost:5555/actuator/routes” 端点，可以看到所配置的前缀已经生效，如下所示：

```json
{
	    "/springhealth/user/**":"userservice" 
}
```

**基于静态配置映射服务路由**

让我们考虑这样一个场景，如果系统中存在一个第三方服务，该服务无法注册到我们的 Eureka 注册中心，还会暴露了一个 HTTP 端点供 SpringHealth 系统进行调用。我们知道 Spring Cloud 中各个服务之间都是通过轻量级的 HTTP 协议进行交互的。而 HTTP 协议具备技术平台无关性，只要能够获取服务的 HTTP 端口地址，原则上我们就可以进行远程访问，而不用关注该服务在实现上采用了何种技术和工具。

因此，针对这一场景，我们可以在配置文件中添加如下的静态路由配置，这样访问`“/thirdpartyservice/**”`时，Zuul 会将该请求转发到外部的第三方服务http://thirdparty.com/thirdpartyservice中

```yaml
zuul:
  routes:
    thirdpartyservice:
      path: /thirdpartyservice/**
	  url: http://thirdparty.com/thirdpartyservice
```

现在的服务路由信息就会变成如下结果：

```json
{
	    "/springhealth/thirdpartyservice/**":"http://thirdparty.com/thirdpartyservice",
	    "/springhealth/user/**":"userservice"
}
```

Zuul 能够与 Ribbon 进行整合。而这种整合也来自手工设置静态服务路由的方式，具体实现方式如下所示：

```yaml
zuul:
  routes:
    thirdpartyservice:
      path: /thirdpartyservice/**
      serviceId: thirdpartyservice

ribbon:
  eureka:
    enabled: false
 
thirdpartyservice:
  ribbon:
	listOfServers: http://thirdpartyservice1:8080,http://thirdpartyservice2:8080
```

这里，我们配置了一个 thirdpartyservice 路由信息，通过`“/ thirdpartyservice /**”`映射到 serviceId 为“thirdpartyservice”的服务中。

然后我们希望通过自定义 Ribbon 的方式来实现客户端负载均衡，这时候就需要关闭 Ribbon 与 Eureka 之间的关联。

可以通过`“ribbon.eureka.enabled: false”`配置项完成这一目标。在不使用 Eureka 的情况下，我们需要手工指定 Ribbon 的服务列表。`“users.ribbon.listOfServers”`配置项为我们提供了这方面的支持，如在上面的示例中“http://thirdpartyservice1:8080，http://thirdpartyservice2:8080”就为 Ribbon 提供了两个服务定义作为实现负载均衡的服务列表

# 工作原理

**ZuulFilter 的定义与 ZuulRegistry**

在 Zuul 中，ZuulFilter 是 Zuul 中的关键组件，我们来看一下它在设计上的抽象过程。在 Netflix Zuul 中存在一个 IZuulFilter 接口，定义如下：

```java
public interface IZuulFilter {
    boolean shouldFilter();
    Object run() throws ZuulException;
}
```

很明显，IZuulFilter 接口的 shouldFilter() 方法决定是否需要执行该过滤器，一般情况该方法都会返回 true，但我们可以使用该方法来根据场景动态设置过滤器是否生效。而 run() 方法显然代表着该 Filter 具体需要实现的业务逻辑。

IZuulFilter的直接实现类是 ZuulFilter，ZuulFilter 是一个抽象类，额外提供了如下所示的两个抽象方法：

```java
//过滤器类型
abstract public String filterType();
//过滤器顺序	 
abstract public int filterOrder();
```

其中 filterType() 代表过滤器的类型，内置过滤器分为 PRE、ROUTING、POST 和 ERROR 四种，本课时后续内容会对这些过滤器进行具体展开；而 filterOrder() 方法用于设置过滤器的执行顺序，这个顺序用数字进行表示，数字越小则越先执行。

然后在 ZuulFilter 类中最核心的就是负责执行 Filter 的 runFilter() 方法，该方法的执行流程并不复杂，如下所示：

```java
public ZuulFilterResult runFilter() {
        ZuulFilterResult zr = new ZuulFilterResult();
        if (!isFilterDisabled()) {
            if (shouldFilter()) {
                Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
                try {
                    Object res = run();
                    zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
                } catch (Throwable e) {
                    t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                    zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                    zr.setException(e);
                } finally {
                    t.stopAndLog();
                }
            } else {
                zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
            }
        }
        return zr;
}
```

既然有了过滤器，系统中就应该存在一个管理过滤器的组件。有时候，我们称这个组件为过滤器链。而在 Netflix Zuul 中，则使用了一个称为过滤器注册表 FilterRegistry 的组件来保存和维护所有 ZuulFilter。FilterRegistry 类定义如下：

```java
public class FilterRegistry {
 
    private static final FilterRegistry INSTANCE = new FilterRegistry();
 
    public static final FilterRegistry instance() {
        return INSTANCE;
    }
 
    private final ConcurrentHashMap<String, ZuulFilter> filters = new ConcurrentHashMap<String, ZuulFilter>();
 
    private FilterRegistry() {
    }
 
    public ZuulFilter remove(String key) {
        return this.filters.remove(key);
    }
 
    public ZuulFilter get(String key) {
        return this.filters.get(key);
    }
 
    public void put(String key, ZuulFilter filter) {
        this.filters.putIfAbsent(key, filter);
    }
 
    public int size() {
        return this.filters.size();
    }
 
    public Collection<ZuulFilter> getAllFilters() {
        return this.filters.values();
    } 
}
```

FilterRegistry 的实现意外得简单，这里采用了经典的单例模式来创建 FilterRegistry 实例。在 FilterRegistry 内部，可以看到就是直接使用了线程安全的 ConcurrentHashMap 来缓存 ZuulFilter。

**ZuulFilter 的加载与 FilterLoader**

在 Zuul 中，FilterLoader 负责 ZuulFilter 的加载。FilterLoader 类是 Zuul 的一个核心类，这点从它的注释说明中就能看出：它用来在源码变化时编译、载入和校验过滤器。针对这个核心类，我们将从它的变量入手，对它进行分析

```javascript
//Filter 文件名与修改时间的映射
private final ConcurrentHashMap<String, Long> filterClassLastModified = new ConcurrentHashMap<String, Long>();
//Filter 名称与代码的映射
private final ConcurrentHashMap<String, String> filterClassCode = new ConcurrentHashMap<String, String>();
//Filter 名称与名称的映射，作用是判断该 Filter 是否存在
private final ConcurrentHashMap<String, String> filterCheck = new ConcurrentHashMap<String, String>();
//Filter 类型与 List<ZuulFilter> 的映射
private final ConcurrentHashMap<String, List<ZuulFilter>> hashFiltersByType = new ConcurrentHashMap<String, List<ZuulFilter>>();
//前面提到的 FilterRegistry 单例
private FilterRegistry filterRegistry = FilterRegistry.instance();
//动态代码编译器实例，Zuul 提供的默认实现是 GroovyCompiler
static DynamicCodeCompiler COMPILER;
//ZuulFilter 工厂类
static FilterFactory FILTER_FACTORY = new DefaultFilterFactory();
```

这些变量中唯一值得注意的就是 DynamicCodeCompiler。顾名思义，它一种动态代码编译器。DynamicCodeCompiler 接口定义如下：

```java
public interface DynamicCodeCompiler {
    Class compile(String sCode, String sName) throws Exception;
 
    Class compile(File file) throws Exception;
}
```

可以看到 DynamicCodeCompiler 提供了两个接口，一个是从代码编译到类，另一个是从文件编译到类。GroovyCompiler 是该接口的实现类，用于把 Groovy 代码编译为 Java 类。GroovyCompile 的实现如下所示：

```java
public class GroovyCompiler implements DynamicCodeCompiler {
 
    GroovyClassLoader getGroovyClassLoader() {
        return new GroovyClassLoader();
    }
 
    @Override
    public Class compile(String sCode, String sName) {
        GroovyClassLoader loader = getGroovyClassLoader();
        Class groovyClass = loader.parseClass(sCode, sName);
        return groovyClass;
    }
 
    @Override
    public Class compile(File file) throws IOException {
        GroovyClassLoader loader = getGroovyClassLoader();
        Class groovyClass = loader.parseClass(file);
        return groovyClass;
	}
}
```

注意， GroovyCompile 的核心代码同样比较简单。我们通过 `groovy.lang.GroovyClassLoader `构建一个类加载器，然后调用它的 parseClass 方法即可将一个 Groovy 文件加载为一个 Java 类。

有了 GroovyCompile 提供的动态加载和编译代码的能力，我们再回到 FilterLoader。FilterLoader 分别提供了 getFilter、putFilter 和 getFiltersByType 这三个工具方法，其中实际涉及加载和存储 Filter 的方法只有 putFilter，该方法如下所示：

```java
public boolean putFilter(File file) throws Exception {
        String sName = file.getAbsolutePath() + file.getName();
        if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
            LOG.debug("reloading filter " + sName);
            filterRegistry.remove(sName);
        }
        ZuulFilter filter = filterRegistry.get(sName);
        if (filter == null) {
            Class clazz = COMPILER.compile(file);
            if (!Modifier.isAbstract(clazz.getModifiers())) {
                filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
                List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
                if (list != null) {
                    hashFiltersByType.remove(filter.filterType()); 
                }
                filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
                filterClassLastModified.put(sName, file.lastModified());
                return true;
            }
        }
 
        return false;
}
```

上述 putFilter 方法通过文件加载 ZuulFilter，整体执行流程也比较简洁明了。首先通过修改时间判断文件是否被修改过，如果被修改过则从 FilterRegistry 移除这个 Filter。然后根据 FilterRegistry 是否缓存了这个 Filter 来决定是否使用前面介绍 DynamicCodeCompiler 动态加载代码，而根据加载的代码再使用 FilterFactory 工厂类来实例化 ZuulFilter。

FilterFactory 的实现类是 DefaultFilterFactory，它的实例创建过程也很直接，就是利用了反射机制，如下所示：

```java
@Override
public ZuulFilter newInstance(Class clazz) throws InstantiationException, IllegalAccessException {
        return (ZuulFilter) clazz.newInstance();
}
```

顺着 FilterLoader 类的 putFilter 方法继续往前走，我们需要明确该方法中传入的 File 对象从何而来。实际上这个 File 对象是由 FilterFileManager 类提供的。顾名思义，这个类的作用是管理这些 File 对象。FilterFileManager 中包含了如下所示的 manageFiles 和 processGroovyFiles 方法，后者调用了 FilterLoader 的 putFilter 方法以进行 Groovy 文件的处理

```java
void manageFiles() throws Exception, IllegalAccessException, InstantiationException {
        List<File> aFiles = getFiles();
        processGroovyFiles(aFiles);
}
	 
void processGroovyFiles(List<File> aFiles) throws Exception, InstantiationException, IllegalAccessException {
        for (File file : aFiles) {
            FilterLoader.getInstance().putFilter(file);
        }
}
```

那么这个 manageFiles 方法是什么时候被调用的呢？注意到在 FilterFileManager 中存在一个每隔 pollingIntervalSeconds 秒就会轮询一次的后台守护线程，该线程就会调用 manageFiles 方法来重新加载 Filter，如下所示：

```java
void startPoller() {
        poller = new Thread("GroovyFilterFileManagerPoller") {
            public void run() {
                while (bRunning) {
                    try {
                        sleep(pollingIntervalSeconds * 1000);
                        manageFiles();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        poller.setDaemon(true);
        poller.start();
}
```

分析到这里，我们对 ZuulFilter 的加载和管理流程已经非常清晰了。Zuul 通过文件来存储各种 ZuulFilter 的定义和实现逻辑，然后启动一个守护线程定时轮询这些文件，确保变更之后的文件能够动态加载到 FilterRegistry 中

**RequestContext 与上下文**

]我们可以通过该对象将业务信息放到请求上下文（Context）中，并使其在各个 ZuulFilter 中进行传递。

我们必须有一种共识，即每个新的请求都是由一个独立的线程进行处理，诸如 Tomcat 之类的服务器会为我们启动了这个线程。也就是说，HTTP 请求的所有参数总是绑定在处理请求的线程中。基于这一点，我们不难想象，对 RequestContext 的访问必须设计成线程安全，Zuul 使用了非常常见和实用的 ThreadLocal，如下所示：

```java
protected static final ThreadLocal<? extends RequestContext> threadLocal = new ThreadLocal<RequestContext>() {
        @Override
        protected RequestContext initialValue() {
            try {
                return contextClass.newInstance();
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        }
};
```

可以看到，这里使用了线程安全的 ThreadLocal 来存放所有的 RequestContext。作为一个实现上的技巧，请注意，这里重写了 ThreadLocal 的 initialValue() 方法。重写目的是确保ThreadLocal 的 get() 方法始终会获取一个 RequestContext 实例。这样做的原因是因为默认情况下 ThreadLocal 的 get() 方法可能会返回 null，而通过重写 initialValue() 方法可以在返回 null 时自动初始化一个 RequestContext 实例。

用于获取 RequestContext 的 getCurrentContext 方法如下所示：

```java
public static RequestContext getCurrentContext() {
	    RequestContext context = threadLocal.get();
        return context;
}
```

我们在开发业务代码时可以先获取这个 RequestContext 对象，进而获取该对象中所存储的各种业务数据。RequestContext 为我们提供了大量的 getter/setter 方法对来完成这一操作。与 Zuul 中的其他组件一样，通过分析代码，我们发现 RequestContext 的实现也是直接对 ConcurrentHashMap 做了一层封装，其中 Key 是 String 类型，而 Value 是 Object 类型，因此我们可以将任何想要传递的数据放到 RequestContext 中。

**HTTP 请求与过滤器执行**

接下去我们分析如何使用加载好的 ZuulFilter。Zuul 提供了 FilterProcessor 类来执行 Filter，参考如下所示的 runFilters 方法：

```java
public Object runFilters(String sType) throws Throwable {

        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
}
```

注意这个方法传入了 Filter 的类型，然后通过 FilterLoader 获取所属类型的 ZuulFilter 列表。通过遍历列表并调用 processZuulFilter 方法来完成对每个 Filter 的执行。processZuulFilter 方法的结构也比较简单，核心就是通过调用 ZuulFilter 自身的 runFilter 方法，然后获取执行结果状态。

我们同时看到 FilterProcessor 基于 runFilters 方法衍生出 postRoute()、preRoute()、error() 和 route() 这四个方法，分别对应 Zuul 中的四种过滤器类型。其中 PRE 过滤器在请求到达目标服务器之前调用，ROUTING 过滤器把请求发送给目标服务，POST 过滤器在请求从目标服务返回之后执行，而 ERROR 过滤器则在发生错误时执行。同时，这四种过滤器有一定的执行顺序

每一种过滤器对应了一个 HTTP 请求的执行生命周期。因此，我们可以在不同的过滤器中添加针对该生命周期的特定执行逻辑。例如，在日常开发过程中，常见的做法是在 PRE 过滤器中执行身份验证和记录请求日志等操作；而在 POST 过滤器返回的响应中添加消息头或者各种统计信息等。

基于对 Zuul 请求生命周期的理解，我们最后来看一下 ZuulServlet。在 Spring Cloud 中所有请求都是 HTTP 请求，Zuul 作为一个服务网关同样也需要完成对 HTTP 请求的响应。ZuulServlet 是 Zuul 中对 HttpServlet 接口的一个实现类。我们知道 Servlet 中最核心的就是 service 方法，ZuulServlet 的 service 方法如下所示：

```java
@Override
public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
 
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();
 
            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }
 
        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
}
```

注意这里的 postRoute()、preRoute()、error() 和 route() 这四个方法同样只是对 ZuulRunner 中同名方法的直接封装。你可以尝试通过这段代码梳理 Zuul 中四种过滤器的执行顺序。

**Spring Cloud 集成 ZuulFilter**

针对 Zuul 的整体架构，在设计上的初衷是把所有 ZuulFilter 通过 Groovy 文件的形式进行管理和加载。但是，很遗憾，在 Spring Cloud 体系中并没有使用此策略来加载 ZuulFilter。

那么 Spring Cloud 的 ZuulFilter是如何实现加载的呢？

spring-cloud-netflix-zuul 同样是一个 Spring Boot 应用程序，所以我们还是直接进入 ZuulServerAutoConfiguration 类，并找到了如下所示的 ZuulFilterConfiguration 类：

```java
@Configuration
protected static class ZuulFilterConfiguration {
        @Autowired
        private Map<String, ZuulFilter> filters;
 
        @Bean
        public ZuulFilterInitializer zuulFilterInitializer(
                CounterFactory counterFactory, TracerFactory tracerFactory) {
            FilterLoader filterLoader = FilterLoader.getInstance();
            FilterRegistry filterRegistry = FilterRegistry.instance();
            return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory, filterLoader, filterRegistry);
        }
}
```

这里我们看到了熟悉的 FilterLoader 和 FilterRegistry，通过 ZuulFilterInitializer 的构造函数引入了过滤器加载的流程。同时，我们发现 filters 变量是一个 Key 为 String、Value 为 ZuulFilter 的 Map，而该变量上添加了 @Autowired 注解。这就意味着 Spring 会把 ZuulFilter 的 bean 自动装载到 Map 对象中。

我们在 ZuulServerAutoConfiguration 以及它的子类 ZuulProxyAutoConfiguration 中发现了很多添加了 @Bean 注解并以“Filter”结尾的类，其形式如下所示：

```java
@Bean
public ServletDetectionFilter servletDetectionFilter() {
        return new ServletDetectionFilter();
}
```

这个 ServletDetectionFilter 类就是 ZuulFilter 的子类，在 Spring 容器中会被自动注入ZuulFilterConfiguration 的 filters 对象中。

获取了容器中自动注入的 ZuulFilter 之后，我们继续跟进 ZuulFilterInitializer，发现了如下所示的 contextInitialized 方法：

```java
@PostConstruct
public void contextInitialized() {
        log.info("Starting filter initializer");
 
        TracerFactory.initialize(tracerFactory);
        CounterFactory.initialize(counterFactory);
 
        for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
            filterRegistry.put(entry.getKey(), entry.getValue());
        }
}
```

注意到该方法开启了 @PostConstruct 注解，因此在 Spring 容器中，一旦 ZuulFilterInitializer 的构造函数执行完成，就会执行这个 contextInitialized 方法。而在该方法中，我们通过 FilterRegistry 的 put 方法把从容器中自动注入的各个 Filter 添加到 FilterRegistry 中。同样，在 ZuulFilterInitializer 也存在一个被添加了 @PostConstruct 注解的 contextDestroyed 方法，该方法中调用了 filterRegistry 的 remove 方法删除 Filter。

至此，Spring Cloud 通过 Netflix Zuul 提供的 FilterLoader 和 FilterRegistry 等工具类完成了两者之间的集成