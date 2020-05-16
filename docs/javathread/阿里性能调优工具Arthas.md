[TOC]

## 目标

* 了解Arthas 性能调优工具

  参考: [Arthas - Java 线上问题定位处理的终极利器](https://blog.csdn.net/u013735734/article/details/102930307)

  [spring-boot 速成(3) actuator](https://www.cnblogs.com/yjmyzz/p/spring-boot-actuator-tutorial.html)

  [JVM内存区域详解（Eden Space、Survivor Space、Old Gen、Code Cache和Perm Gen）](https://blog.csdn.net/qq_38763620/article/details/77373830)

  [java性能监控利器Arthas](https://blog.csdn.net/hao134838/article/details/102999833)

## Arthas

是一个Java诊断工具，支持JDK 6+，支持Linux/Mac/Winodws

官方文档：https://alibaba.github.io/arthas/

国内文档地址：https://arthas.gitee.io/

安装包： https://arthas.gitee.io/arthas-boot.jar  国内

提供的测试项目： https://arthas.gitee.io/arthas/arthas-demo.jar

**强烈建议根据官方文档的，在线教程(推荐) 学习了解。**

idea 插件[arthas-idea](https://plugins.jetbrains.com/plugin/13581-arthas-idea)  [汪小哥](https://github.com/WangJi92/arthas-idea-plugin) 这位大哥写的，简化一些复杂命令

### 简单使用

先执行要检测诊断的项目：参见 https://blog.csdn.net/u013735734/article/details/102930307 提供的代码段

```java
import java.util.HashSet;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import lombok.extern.slf4j.Slf4j;
 
/**
* <p>
* Arthas Demo
* 公众号：未读代码
*
* @Author niujinpeng
*/
@Slf4j
public class Arthas {
 
    private static HashSet hashSet = new HashSet();
    /** 线程池，大小1*/
    private static ExecutorService executorService = Executors.newFixedThreadPool(1);
 
    public static void main(String[] args) {
        // 模拟 CPU 过高，这里注释掉了，测试时可以打开
        // cpu();
        // 模拟线程阻塞
        thread();
        // 模拟线程死锁
        deadThread();
        // 不断的向 hashSet 集合增加数据
        addHashSetThread();
    }
 
    /**
     * 不断的向 hashSet 集合添加数据
     */
    public static void addHashSetThread() {
        // 初始化常量
        new Thread(() -> {
            int count = 0;
            while (true) {
                try {
                    hashSet.add("count" + count);
                    Thread.sleep(10000);
                    count++;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
 
    public static void cpu() {
        cpuHigh();
        cpuNormal();
    }
 
    /**
     * 极度消耗CPU的线程
     */
    private static void cpuHigh() {
        Thread thread = new Thread(() -> {
            while (true) {
                log.info("cpu start 100");
            }
        });
        // 添加到线程
        executorService.submit(thread);
    }
 
    /**
     * 普通消耗CPU的线程
     */
    private static void cpuNormal() {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                while (true) {
                    log.info("cpu start");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
 
    /**
     * 模拟线程阻塞,向已经满了的线程池提交线程
     */
    private static void thread() {
        Thread thread = new Thread(() -> {
            while (true) {
                log.debug("thread start");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        // 添加到线程
        executorService.submit(thread);
    }
 
    /**
     * 死锁
     */
    private static void deadThread() {
        /** 创建资源 */
        Object resourceA = new Object();
        Object resourceB = new Object();
        // 创建线程
        Thread threadA = new Thread(() -> {
            synchronized (resourceA) {
                log.info(Thread.currentThread() + " get ResourceA");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info(Thread.currentThread() + "waiting get resourceB");
                synchronized (resourceB) {
                    log.info(Thread.currentThread() + " get resourceB");
                }
            }
        });
 
        Thread threadB = new Thread(() -> {
            synchronized (resourceB) {
                log.info(Thread.currentThread() + " get ResourceB");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info(Thread.currentThread() + "waiting get resourceA");
                synchronized (resourceA) {
                    log.info(Thread.currentThread() + " get resourceA");
                }
            }
        });
        threadA.start();
        threadB.start();
    }
}
 
```

* 查看系统 和 Thread 的信息

```shell
#  启动arthas
$ java -jar .\arthas-boot.jar
# 启动后，会打印出来现在所有的java进程，选择要检测的java程序
# 看到arthas 图标出现后，就代表检测成功了
 
# 全局检测
$ dashboard
 
# 输出
ID       NAME                      GROUP             PRIORIT STATE    %CPU     TIME     INTERRU DAEMON
5        Attach Listener           system            5       RUNNABLE 0        0:0      false   true
17       DestroyJavaVM             main              5       RUNNABLE 0        0:0      false   false
3        Finalizer                 system            8       WAITING  0        0:0      false   true
6        Monitor Ctrl-Break        main              5       RUNNABLE 0        0:0      false   true
2        Reference Handler         system            10      WAITING  0        0:0      false   true
4        Signal Dispatcher         system            9       RUNNABLE 0        0:0      false   true
14       Thread-1                  main              5       BLOCKED  0        0:0      false   false
15       Thread-2                  main              5       BLOCKED  0        0:0      false   false
16       Thread-3                  main              5       TIMED_WA 0        0:0      false   false
29       Timer-for-arthas-dashboar system            10      RUNNABLE 0        0:0      false   true
22       arthas-shell-server       system            5       TIMED_WA 0        0:0      false   true
28       as-command-execute-daemon system            10      TIMED_WA 0        0:0      false   true
19       job-timeout               system            5       TIMED_WA 0        0:0      false   true
Memory                   used   total   max    usage   GC
heap                     41M    243M    3607M  1.15%   gc.ps_scavenge.count      2
ps_eden_space            29M    63M     1331M  2.23%   gc.ps_scavenge.time(ms)   12
ps_survivor_space        10M    10M     10M    99.85%  gc.ps_marksweep.count     0
ps_old_gen               1M     169M    2705M  0.05%   gc.ps_marksweep.time(ms)  0
nonheap                  22M    23M     -1     96.87%
code_cache               4M     4M      240M   1.93%
Runtime
os.name                                              Windows 10
os.version                                           10.0
java.version                                         1.8.0_111
java.home                                            D:\bcinstall\jdk\jre
systemload.average                                   -1.00
processors                                           12
uptime                                               736s
 
 
 
heap ： 堆
ps_eden_space ： 新生代-伊甸园
ps_survivor_space ：  新生代中-幸存者
ps_old_gen ： 老年代
nonheap ： 非堆内存
    保存虚拟机自己的静态(reflective)数据，例如类（class）和方法（method）对象。Java虚拟机共享这些类数据。这个区域被分割为只读的和只写的。
code_cache ： 代码缓存区
    HotSpot Java虚拟机包括一个用于编译和保存本地代码（native code）的内存，叫做“代码缓存区”（code cache）。
 
 
gc.ps_scavenge.count  ： 垃圾回收次数
gc.ps_scavenge.time ：垃圾回收消耗时间
gc.ps_marksweep.count ： 标记-清除算法的次数
gc.ps_marksweep.time ：标记-清除算法的消耗时间
 
 
STATE 状态
    RUNNABLE 运行中
    TIMED_WAITIN 调用了以下方法的线程会进入TIMED_WAITING：
        Thread#sleep()
        Object#wait() 并加了超时参数
        Thread#join() 并加了超时参数
        LockSupport#parkNanos()
        LockSupport#parkUntil()
    WAITING 当线程调用以下方法时会进入WAITING状态：
        Object#wait() 而且不加超时参数
        Thread#join() 而且不加超时参数
        LockSupport#park()
    BLOCKED 阻塞，等待锁
 
# 查看所有的线程
$ thread
 
# 查看某个线程 thread Id
$ thread 14
# 输出
"Thread-1" Id=14 BLOCKED on java.lang.Object@37ea7c84 owned by "Thread-2" Id=15
    at fun.gengzi.thread.Arthas.lambda$deadThread$4(Arthas.java:125)
    -  blocked on java.lang.Object@37ea7c84
    -  locked java.lang.Object@74a2f34e
    at fun.gengzi.thread.Arthas$$Lambda$2/511833308.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:745)
 
Affect(row-cnt:0) cost in 11 ms.
 
提示 Thread-1 被 thread-2 拥有的  java.lang.Object@37ea7c84 阻塞
    在 fun.gengzi.thread.Arthas.lambda$deadThread$4(Arthas.java:125) 发生
    在 java.lang.Object@37ea7c84 上被阻塞
    锁定了java.lang.Object@74a2f34e
 
# 查看死锁信息
$ thread -b
# 输出
"Thread-1" Id=14 BLOCKED on java.lang.Object@37ea7c84 owned by "Thread-2" Id=15
    at fun.gengzi.thread.Arthas.lambda$deadThread$4(Arthas.java:125)
    -  blocked on java.lang.Object@37ea7c84
    -  locked java.lang.Object@74a2f34e <---- but blocks 1 other threads!（阻塞了一个其他线程）
    at fun.gengzi.thread.Arthas$$Lambda$2/511833308.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:745)
 
Affect(row-cnt:0) cost in 11 ms.
 
 
# 排列出 CPU 使用率 Top N 的线程
$  thread -n 5
# 输出
[arthas@44800]$ thread -n 5
"Reference Handler" Id=2 cpuUsage=0% WAITING on java.lang.ref.Reference$Lock@6915cdae
    at java.lang.Object.wait(Native Method)
    -  waiting on java.lang.ref.Reference$Lock@6915cdae
    at java.lang.Object.wait(Object.java:502)
    at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
    at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)
 
"Finalizer" Id=3 cpuUsage=0% WAITING on java.lang.ref.ReferenceQueue$Lock@d7f8ed9
    at java.lang.Object.wait(Native Method)
    -  waiting on java.lang.ref.ReferenceQueue$Lock@d7f8ed9
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
    at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
 
"Signal Dispatcher" Id=4 cpuUsage=0% RUNNABLE
"Attach Listener" Id=5 cpuUsage=0% RUNNABLE
 
"job-timeout" Id=19 cpuUsage=0% TIMED_WAITING on java.util.TaskQueue@2013be8
    at java.lang.Object.wait(Native Method)
    -  waiting on java.util.TaskQueue@2013be8
    at java.util.TimerThread.mainLoop(Timer.java:552)
    at java.util.TimerThread.run(Timer.java:505)
 
Affect(row-cnt:0) cost in 113 ms.
 
 
```

* 统计方法耗时 trace 和 monitor

  这里需要启动另一个工程，https://github.com/gengzi/GsjBlog 这里使用的是这个工程，调用了一个 登陆的接口，包：club.gsjglob.action.UserController 方法  login ，在浏览器端请求，来查看这个方法的一些执行情况。（可以使用自己的web工程做测试）

```shell
# trace 跟踪统计方法耗时
$ trace club.gsjglob.action.UserController  login
# 输出
[arthas@241]$ trace club.gsjglob.action.UserController  login
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 194 ms.
`---ts=2020-05-15 13:33:25;thread_name=http-nio-8080-exec-2;id=12;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.ParallelWebappClassLoader@648b39c
    `---[23.17502ms] club.gsjglob.action.UserController:login()
        +---[0.02849ms] club.gsjglob.dto.User:getIsadmin() #60
        +---[14.269801ms] club.gsjglob.service.IUserService:login() #63
        `---[0.017177ms] club.gsjglob.domain.GsjUser:getUserid() #64
 
可以看到执行到的各个方法耗时
[14.269801ms] club.gsjglob.service.IUserService:login() #63 这个耗时比较长
可以继续查找这个耗时长的方法，直到确定那个方法比较耗时，这里的方法耗时，主要体现在连接查询mysql数据
 
# monitor 命令监控统计方法的执行情况
# 每5秒统计一次 club.gsjglob.action.UserController 类的 login 方法执行情况。
$ monitor -c 5 club.gsjglob.action.UserController  login
# 输出 每五秒钟执行一次
 timestamp            class                               method  total  success  fail  avg-rt(ms)  fail-rate                            
--------------------------------------------------------------------------------------------------------------                           
 2020-05-15 13:40:40  club.gsjglob.action.UserController  login   14     14       0     9.60        0.00%     
 
 
```

* 观察方法的入参出参信息

```shell
# watch -h 查看一下具体的参数
# watch 查看入参和出参
$ watch club.gsjglob.action.UserController login '{params[0],returnObj}'
# 输出
[arthas@241]$ watch club.gsjglob.action.UserController login '{params[0],returnObj}'
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 47 ms.
ts=2020-05-15 13:53:56; [cost=8.543274ms] result=@ArrayList[
    @User[club.gsjglob.dto.User@2abe0213],
    @GsjUser[club.gsjglob.domain.GsjUser@531ded4c],
]
 
可以看到
入参是 @User[club.gsjglob.dto.User@2abe0213],
出参是 @GsjUser[club.gsjglob.domain.GsjUser@531ded4c],
 
 
# 查看入参和出参，出参的 一个属性 Userid
$ watch club.gsjglob.action.UserController login '{params[0],returnObj.getUserid()}'
 
可以看出 对于入参和出参 可以调用他们可以调用的方法，来具体查看你需要查看的信息，更多用法，看 watch -h
 
 
```

* 观察方法的调用路径

```shell
# stack 观察方法调用路径
$ stack club.gsjglob.action.UserController login
# 输出
[arthas@241]$ stack club.gsjglob.action.UserController login
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 50 ms.
ts=2020-05-15 14:08:06;thread_name=http-nio-8080-exec-6;id=16;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.ParallelWebappClassLoader@648b39c
    @sun.reflect.GeneratedMethodAccessor209.invoke()
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:221)
        at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:137)
        at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:110)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandleMethod(RequestMappingHandlerAdapter.java:777)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:706)
        at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:943)
 
 
 
```

* 方法调用时空隧道

```shell
# tt 命令方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测 。
$ tt -t   club.gsjglob.action.UserController login
 
[arthas@241]$ tt -t   club.gsjglob.action.UserController login
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 93 ms.
 INDEX    TIMESTAMP            COST(ms)   IS-RET  IS-EXP   OBJECT          CLASS                           METHOD                        
------------------------------------------------------------------------------------------------------------------------------------------
 1000     2020-05-15 14:10:06  8.938093   true    false    0x123d07bf      UserController                  login                         
 1001     2020-05-15 14:10:06  4.306842   true    false    0x123d07bf      UserController                  login                         
 1002     2020-05-15 14:10:06  1.866464   true    false    0x123d07bf      UserController                  login                         
 1003     2020-05-15 14:10:06  1.850198   true    false    0x123d07bf      UserController                  login                         
 1004     2020-05-15 14:10:07  2.277625   true    false    0x123d07bf      UserController                  login                         
 1005     2020-05-15 14:10:07  1.754767   true    false    0x123d07bf      UserController                  login                         
 1006     2020-05-15 14:10:07  5.819472   true    false    0x123d07bf      UserController                  login                         
 1007     2020-05-15 14:10:07  1.744265   true    false    0x123d07bf      UserController                  login                         
 1008     2020-05-15 14:10:07  4.954611   true    false    0x123d07bf      UserController                  login                         
 1009     2020-05-15 14:10:07  2.020023   true    false    0x123d07bf      UserController                  login                         
 1010     2020-05-15 14:10:07  1.79095    true    false    0x123d07bf      UserController                  login                         
 1011     2020-05-15 14:10:11  1.66948    true    false    0x123d07bf      UserController                  login                         
 1012     2020-05-15 14:10:13  26.496748  false   true     0x123d07bf      UserController                  login                         
 1013     2020-05-15 14:10:17  5.981863   false   true     0x123d07bf      UserController                  login                         
 1014     2020-05-15 14:10:17  6.267794   false   true     0x123d07bf      UserController                  login                         
 1015     2020-05-15 14:10:17  4.178886   false   true     0x123d07bf      UserController                  login                         
 1016     2020-05-15 14:10:17  7.940246   false   true     0x123d07bf      UserController                  login                         
 1017     2020-05-15 14:10:18  10.339824  false   true     0x123d07bf      UserController                  login                         
 1018     2020-05-15 14:10:18  8.588795   false   true     0x123d07bf      UserController                  login       
 
 
 IS-EXP = true  表示有异常
 
 # 查看记录的方法调用信息： tt -l 
$ tt -l
 # 查看某一个调用记录的详细信息  tt -i INDEX
 $  tt -i 1018
# 输出
[arthas@241]$ tt -i 1018
 INDEX            1018                                                                                                  
 GMT-CREATE       2020-05-15 14:10:18                                                                                                    
 COST(ms)         8.588795                                                                                                               
 OBJECT           0x123d07bf                                                                                                             
 CLASS            club.gsjglob.action.UserController                                                                                     
 METHOD           login                                                                                                                  
 IS-RETURN        false                                                                                                                  
 IS-EXCEPTION     true                                                                                                                   
 PARAMETERS[0]    @User[                                                                                                                 
                      adminname=@String[gsjadmin],                                                                                       
                      adminpasswd=@String[gsjadmin],                                                                                     
                      isadmin=@String[admin],                                                                                            
                  ]                                                                                                                      
 PARAMETERS[1]    @RequestFacade[                                                                                                        
                      request=@Request[org.apache.catalina.connector.Request@49a22c5e],                                                  
                      sm=@StringManager[org.apache.tomcat.util.res.StringManager@4d9da1b2],                                              
                  ]                                                                                                                      
 PARAMETERS[2]    @ResponseFacade[                                                                                                       
                      sm=@StringManager[org.apache.tomcat.util.res.StringManager@4d9da1b2],                                              
                      response=@Response[org.apache.catalina.connector.Response@6145ce6e],                                               
                  ]                                                                                                                      
 THROW-EXCEPTION  redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool                        
                      at redis.clients.util.Pool.getResource(Pool.java:50)                                                                  
                      at redis.clients.jedis.JedisPool.getResource(JedisPool.java:86)                                                       
                      at club.gsjglob.tools.JedisClientPool.set(JedisClientPool.java:15)                                                    
                      at club.gsjglob.action.UserController.login(UserController.java:76)                                                   
                      at sun.reflect.GeneratedMethodAccessor209.invoke(Unknown Source)                                                      
                      at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)   
 
 
这里输出了调用这个方法的参数 和 出现的异常信息，发现是 redis 相关的异常。
 
 # 重新发起调用某一个记录  tt -i INDEX -p 
$ tt -i 1018 -p
 
 
```

 

## 其他

jps jvm提供的，可以查看当前所有的java进程

## 其他诊断工具

*  Bistoury

  github地址： [bistoury](https://github.com/qunarcorp/bistoury)

  Bistoury是去哪儿网的java应用生产问题诊断工具，提供了一站式的问题诊断方案

  Bistoury在保留arthas和vjtools的所有功能之外，还提供了更加丰富的功能

  [动态对方法添加监控](https://github.com/qunarcorp/bistoury/blob/master/docs/cn/monitor.md)

  [在线debug功能](https://github.com/qunarcorp/bistoury/blob/master/docs/cn/debug.md)

  [线程级cpu使用率监控](https://github.com/qunarcorp/bistoury/blob/master/docs/cn/jstack.md)

* [vjtools](https://github.com/vipshop/vjtools)

  唯品会开源的诊断工具

  github地址：[vjtools](https://github.com/vipshop/vjtools)

  视频：[《VJTools如何利用佛性技术玩转JVM》](http://kai.vkaijiang.com/product/course?courseID=120897)

  文档：[《入门科普，围绕JVM的各种外挂技术》](https://mp.weixin.qq.com/s/cwU2rLOuwock048rKBz3ew)

 