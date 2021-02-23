[TOC]

## 目标

* 了解MDC基本概念和用法

  参考：[MDC官方文档](http://www.logback.cn/08%E7%AC%AC%E5%85%AB%E7%AB%A0MDC.html) 推荐参考

阅读前需要先了解logback的基本内容。以下内容，都是从官方文档引用，只是增加了一些示例情况。

## 诊断上下文映射 (MDC)

### 引入MDC 的目的

在一个多线程程序中，不同线程处理不同客户端的请求，如果对每个客户端都实例化一个新的且独立的 logger对象，用于区分一个客户端与另一个客户端的日志输出，会导致 logger 急剧增加并且会增加维护成本。所以提出了一种轻量级的技术：给每个为客户端服务的 logger 打一个标记，用于区分不同客户端的日志输出。这个技术就是MDC（Mapped Diagnostic Context ）诊断上下文映射 。

### MDC 的作用

先看下MDC 提供的方法，完成的方法列表请查看 [MDC javadocs](http://www.slf4j.org/api/org/slf4j/MDC.html)。

> 有没有感觉跟ThreadLocal的方法类似,可以试着理解MDC跟ThreadLocal差不多。

**MDC为每个请求打上唯一的标记，需要打印的上下文信息放到 `MDC`中会自动在每个日志条目中包含这些信息。**

```java
package org.slf4j;
 
public class MDC {
  // 将上下文的值作为 MDC 的 key 放到线程上下文的 map 中
  public static void put(String key, String val);
 
  // 通过 key 获取上下文标识
  public static String get(String key);
 
  // 通过 key 移除上下文标识
  public static void remove(String key);
 
  // 清除 MDC 中所有的 entry
  public static void clear();
}
```

在`MDC` 类中只包含静态方法。它让开发人员可以在 MDC中放置信息，而后通过特定的 logback 组件去获取即可。`MDC` 在 **每个线程的基础上** 管理上下文信息。通常，当为一个新客户端启动服务时，开发人员会将特定的上文信息插入到 MDC 中。例如，客户端 ID，客户端 IP 地址，请求参数等。如果 logback 组件配置得当的话，会自动在每个日志条目中包含这些信息。

注意：**子线程不会自动继承父线程的MDC。**

> 这里跟ThreadLocal效果是一致的，子线程无法获取父线程ThreadLocal设置的内容，如果需要传递设置的本地变量可以使用InheritableThreadLocal。

### 用法示例

代码：

你可以在 `MDC` 放置尽可能多的键值对。多次插入同一个 key，新值会覆盖旧值。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
 
public class SimpleMDC {
    static public void main(String[] args) throws Exception {
        // 你可以选择在任何时候将值放入 MDC 中
        MDC.put("clientId", "客户端01");
        Logger logger = LoggerFactory.getLogger(SimpleMDC.class);
        MDC.put("addressIp", "192.168.0.1");
        logger.info("执行任务");
        logger.info("完成");
        MDC.put("clientId", "客户端02");
        MDC.put("addressIp", "192.168.0.2");
        logger.info("执行任务");
        logger.info("完成");
        // 演示子线程打印日志，不会继承父线程（main）的MDC
        new Thread(() -> {
            logger.info("子线程-执行任务");
        }).start();
    }
}
```

logback.xml 配置

使用 %X{key} 的形式关联MDC设置的clientId 和 addressIp的内容。

```
%X{clientId} %X{addressIp}
```

完整配置

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  <layout>
    <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%X{clientId} %X{addressIp}]  %thread %logger Line:%-3L - %msg%n </Pattern>
  </layout>
</appender>
```

打印的日志内容：

```shell
# 打印的日志
2021-01-16 18:32:25.630 INFO  [ 客户端01 192.168.0.1]  main fun.gengzi.baselog.test.SimpleMDC Line:16  - 执行任务
2021-01-16 18:32:25.635 INFO  [ 客户端01 192.168.0.1]  main fun.gengzi.baselog.test.SimpleMDC Line:17  - 完成
2021-01-16 18:32:25.636 INFO  [ 客户端02 192.168.0.2]  main fun.gengzi.baselog.test.SimpleMDC Line:20  - 执行任务
2021-01-16 18:32:25.636 INFO  [ 客户端02 192.168.0.2]  main fun.gengzi.baselog.test.SimpleMDC Line:21  - 完成
# 子线程打印，父线程设置的MDC没有被继承
2021-01-16 18:32:25.733 INFO  [  ]  子线程 fun.gengzi.baselog.test.SimpleMDC Line:24  - 子线程-执行任务
```

从结果看，每条日志都追加上了MDC设置的信息。

### 高级用法

通常，服务器上的多个线程为多个客户端提供服务。尽管 MDC 中的方法是静态的，但是是以每个线程为基础来进行管理的，允许每个服务线程都打上一个 `MDC` 标记。`MDC` 中的 `put()` 与 `get()` 操作仅仅只影响当前线程中的 `MDC`。其它线程中 `MDC` 不会受到影响。给定的 `MDC` 信息在每个线程的基础上进行管理。每个线程都有一份 `MDC` 的拷贝。因此，在对 `MDC` 进行编程时，**开发人员没有必要去考虑线程安全或者同步问题。它自己会安全的处理这些问题。**

#### 代码示例

需求：为每个web请求，追加请求的路径，并打印在每条日志中。

思路：可以选择Filter对每个请求进行拦截，设置MDC内容即可。

```java
public class TraceFilter implements Filter {
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("<<< 初始化TraceFilter>>>");
    }
 
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
            final HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
            String requestURI = httpServletRequest.getRequestURI();
            MDC.put("requestURI", requestURI);
            // 放行
            filterChain.doFilter(servletRequest, servletResponse);
        } finally {
            // 请求MDC 的内容
            MDC.clear();
        }
    }
 
    @Override
    public void destroy() {
        log.info("<<< 销毁TraceFilter>>>");
        MDC.clear();
    }
}
```

配置logback.xml

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  <layout>
    <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%X{requestURI}]  %thread %logger Line:%-3L - %msg%n </Pattern>
  </layout>
</appender>
```

#### 一些问题

正如我们之前看到的，对多个客户端进行请求时，`MDC` 非常有用。在 web 应用中管理用户认证，一个简单的处理方式是在 `MDC` 中设置用户名，在用户退出时进行移除。不幸的是，使用这个技术并不总是可靠的。**因为 `MDC` 是在单线程的基础进行管理，服务器重复使用线程可能会导致 `MDC` 包含错误的信息**。

为了在处理请求时，`MDC` 中包含的信息一直都是正确的。一种可能的处理方式是在开始处理之前存储用户名，结束时进行移除。[`Filter`](http://java.sun.com/javaee/5/docs/api/javax/servlet/Filter.html) 可以用来处理这种情况。

### MDC 与线程管理

MDC 的副本不能总是由工作线程从初始化线程继承。当使用 `java.util.concurrent.Executors` 进行线程管理时就是这种情况。例如，`newCachedThreadPool` 方法会创建一个 `ThreadPoolExecutor`，跟其它线程池代码一样，有着复杂的线程创建逻辑。

在这些情况下，推荐在提交任务去执行之前，在源线程上调用 `MDC.getCopyOfContextMap()`。当任务运行的时候，它的第一个动作，应该是调用 `MDC.setContextMapValues()` 将原始 MDC 值的存储副本与新的 `Executor` 管理线程关联起来。

#### 代码示例

构造一个线程池，在线程池中打印日志。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
 
import java.util.Map;
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
 
public class ExecutorMDCTest {
    private static final AtomicInteger num = new AtomicInteger(1);
 
    static ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            5,
            5 + 1,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingDeque<>(10_00), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "test-thread" + num.getAndIncrement());
        }
    });
 
    static public void main(String[] args) throws Exception {
        Logger log = LoggerFactory.getLogger(ExecutorMDCTest.class);
        MDC.put("bl-traceid", "988f54f4bdd34920a53d908827a9fa59");
        Map<String, String> copyOfContextMap = MDC.getCopyOfContextMap();
        // 线程池
        threadPoolExecutor.execute(() -> {
            // 会丢失日志
            log.info("测试打印日志4：{}", "data");
            MDC.setContextMap(copyOfContextMap);
            log.info("设置后-测试打印日志5：{}", "data");
        });
 
    }
}
```

配置logback.xml

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  <layout>
    <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%X{bl-traceid}]  %thread %logger Line:%-3L - %msg%n </Pattern>
  </layout>
</appender>
```

打印日志：

```shell
# 日志
2021-01-16 19:28:07.791 INFO  []  test-thread1 fun.gengzi.baselog.test.ExecutorMDCTest Line:36  - 测试打印日志4：data
2021-01-16 19:28:07.797 INFO  [988f54f4bdd34920a53d908827a9fa59]  test-thread1 fun.gengzi.baselog.test.ExecutorMDCTest Line:38  - 设置后-测试打印日志5：data
```

从结果看，在设置   MDC.setContextMap 后成功打印了MDC中 bl-traceid 的内容。

### MDCInsertingServletFilter

在 web 应用中，获取给定 HTTP 请求的主机名，请求 URI 以及 user-agent 是十分有用的。[`MDCInsertingServletFilter`](https://logback.qos.ch/xref/ch/qos/logback/classic/helpers/MDCInsertingServletFilter.html) 通过如下的 key，将数据插入到 MDC 中。

| MDC key             | MDC value                                                    |
| ------------------- | ------------------------------------------------------------ |
| `req.remoteHost`    | 由 [getRemoteHost()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/ServletRequest.html#getRemoteHost()) 返回 |
| `req.xForwardedFor` | 请求头 ["X-Forwarded-For"](http://en.wikipedia.org/wiki/X-Forwarded-For) 的值 |
| `req.method`        | 由 [getMethod()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getMethod()) 返回 |
| `req.requestURI`    | 由 [getRequestURI()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getRequestURI()) 返回 |
| `req.requestURL`    | 由 [getRequestURL()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getRequestURL()) 返回 |
| `req.queryString`   | 由 [getQueryString()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getQueryString()) 返回 |
| `req.userAgent`     | 请求头 "User-Agent" 的值                                     |

想要使用 `MDCInsertingServletFilter`，需要在 *web.xml* 中添加以下行：

```xml
<filter>
  <filter-name>MDCInsertingServletFilter</filter-name>
  <filter-class>
    ch.qos.logback.classic.helpers.MDCInsertingServletFilter
  </filter-class>
</filter>
<filter-mapping>
  <filter-name>MDCInsertingServletFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

**如果你的 web 应用有多个过滤器，请确保 MDCInsertingServletFilter 在其它过滤器之前声明** 。例如，假设 web 应用的主要处理是在过滤器 'F' 中完成，那么如果 `MDCInsertingServletFilter` 在 'F' 之后被调用，那么在 'F' 中被调用的代码将看不到 `MDCInsertingServletFilter` 设置的 MDC 的值。(这不是废话吗......)

一旦使用了该过滤器，通过 %X 可以输出相对应的 MDC 的值。例如，想要在一行上打印远程主机，后面跟着请求的 URI，令一行打印日志，后面跟着日志消息。那么你应该如下设置 `PatternLayout`：

```
%X{req.remoteHost} %X{req.requestURI}%n%d - %m%n
```

 

### 注意点

* 通常，`put()` 操作应该由 `remove()` 操作来平衡。否则的话，`MDC` 将会保留旧值。我们推荐无论何时，在 finally 代码块中调用 `remove` 操作，用来确保该操作的执行，而不用关心代码的具体执行路径。

 

 
