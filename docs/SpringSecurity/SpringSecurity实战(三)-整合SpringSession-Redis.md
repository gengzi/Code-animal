[TOC]

## 目标

* 了解与Spring session ，redis 的整合

  参考：spring security 实战书籍 6.6 章节

## 集群会话

单机提供单服务只能存在于测试环境，正式环境部署工程，一般都是集群部署或者单机多服务部署。看下两者会话信息的不同：

单机单服务：通过一个服务中间件实例，例如tomcat，来提供服务。使用session来保持会话信息，session 信息被存储在内存中。

集群部署或者单机多服务：采用了多台服务器，或者多个tomcat提供服务，但是多台服务器或者多个tomcat ，session 是无法共享的。大概有三种解决方案：

* session 保持
* session 复制
* session 共享

### session 保持

一般依靠负载均衡组件（例如nginx）ip哈希负载策略来将相同客户端的请求转发到同一个tomcat实例上。这样同一个客户的会话信息，只会存在于一台tomcat，就无需对session 进行处理。优点：简单。缺点：可能会出现负载失衡，可能使用同个ip出口，导致这些请求都转发到了一个tomcat 上。

**如果就部署个两台，这种方式简单快捷，优先使用。**

### session 复制

将每个集群服务器的session信息，复制到其他服务器上，保持会话一致。缺点：实现难度大，消耗资源多。

### session 共享

集群部署推荐方式，将session 数据都抽离到第三方容器中，每个服务器实例都可以操作这个第三方容器，实现对session 的存取。优点： 第三方容器数据容量相较于服务器内存大很多，服务实例中断，不会导致会话信息丢失。缺点：需要引入第三方组件，降低系统的稳定性，增加网络开销等。

## Session 共享实现

数据容器：Redis  采用Redisson框架（具有内存中数据网格功能的Redis Java客户端）

集成session 框架：spring session

项目环境：spring boot 2.2.7 

简单说下Redisson，通常使用该框架，来实现redis 分布式锁的实现。当然还包含了很多其他工程。可以与spring 的众多框架结合（spring boot ，spring data，spring cache ，spring session），官方地址：https://github.com/redisson/redisson

spring session 中文文档：https://www.springcloud.cc/spring-session.html

### 代码实践

代码参考：https://github.com/gengzi/gengzi_spring_security

#### 核心依赖

spring session 与 redis 结合，只需要导入  spring-session-data-redis 。 无需再次引入 spring-session-core 。官方配置文档：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/guides/boot-redis.html

redisson 提供的 spring session 结合的配置和依赖：https://github.com/redisson/redisson/wiki/14.-Integration-with-frameworks#147-spring-session

只说明核心依赖，其他请参考：[pom.xml](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/pom.xml)

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <!--data-redis-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
         <!--session-redis-->
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
        <!--redisson-->
        <dependency>
            <groupId>org.redisson</groupId>
            <!-- for Spring Data Redis v.2.2.x -->
            <artifactId>redisson-spring-data-22</artifactId>
            <version>3.13.6</version>
        </dependency>
```

 

### 核心配置

代码参考：[RedissonSpringDataConfig](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/config/RedissonSpringDataConfig.java)

这个也是 redisson 官方提供的配置类，另外需要在 resources 目录配置  redisson.yml ，用于配置连接redis服务器的配置。

```java
@Configuration
// 启用redis管理会话
@EnableRedisHttpSession
public class RedissonSpringDataConfig extends AbstractHttpSessionApplicationInitializer {
 
    @Bean
    public RedissonConnectionFactory redissonConnectionFactory(RedissonClient redisson) {
        return new RedissonConnectionFactory(redisson);
    }
 
    @Bean(destroyMethod = "shutdown")
    public RedissonClient redisson(@Value("classpath:/redisson.yml") Resource configFile) throws IOException {
        Config config = Config.fromYAML(configFile.getInputStream());
        return Redisson.create(config);
    }
 
}
```

简单说下，AbstractHttpSessionApplicationInitializer 的作用：

spring session 提供了一个 `springSessionRepositoryFilter`  的类，来使得spring session 替换 HttpSession。那为了确保我们的Servlet容器（即Tomcat）对每个请求都使用`springSessionRepositoryFilter`，需要继承 AbstractHttpSessionApplicationInitializer ，来保证spring 会加载我们的 配置。

#### redisson.yml

配置在 resources 目录下：

更多配置请参考Redisson 的说明：https://github.com/redisson/redisson/wiki/Table-of-Content 提供了各个模式的配置

代码参考：[redisson.yml](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/resources/redisson.yml)

```yml
singleServerConfig:
  address: "redis://127.0.0.1:6379"
  password: xxx
  database: 1
 
```

#### spring session 的配置

请参考：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/guides/boot-redis.html

如果不使用 Redisson ，也可以直接提供的默认配置来连接redis 服务器

application.yml

```yml
  session:
      # 会话存储类型
    store-type: redis
```

 

### Spring Security 整合 Spring session

官方文档说明：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/#spring-security-concurrent-sessions

代码参考：[**WebSecurityConfig**](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/config/WebSecurityConfig.java)

```java
    // ------------  会话相关配置 start -------------
    // 会话存储库
    @Autowired
    private FindByIndexNameSessionRepository sessionRepository;
 
    // spring session 会话注册表
    @Bean
    public SpringSessionBackedSessionRegistry sessionRegistry() {
        return new SpringSessionBackedSessionRegistry<>(this.sessionRepository);
    }
    // ------------  会话相关配置 end -------------
 
 
        @Override
    protected void configure(HttpSecurity http) throws Exception {
 
        http.
                .sessionManagement((sessionManagement) -> sessionManagement
                        .maximumSessions(100)
                        .sessionRegistry(sessionRegistry()));
    }
 
```

简单说明一下：FindByIndexNameSessionRepository 返回了一个 session 的实例对象

这个配置其实是 Spring Security并发会话控制 ， maximumSessions （最大session个数）控制单个用户同时活动的数量。测试环境，建议调大，便于测试。生产环境，看业务需要，如果限制单个用户只能在一端登陆，就设置为1.

* Spring Security记住我支持

  这个暂时不说明，后面会细说。需要的直接看官方文档：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/#spring-security-rememberme

## 问题解决

1. 报错：RedisConnectionFactory is required 异常

   原因，导错包了，仅导入了spring-session-core的依赖，导致每次启动项目都不加载redisson 的配置，每次去创建 RedisTemplate 都提示 RedisConnectionFactory  为null。

 

 

 

 
