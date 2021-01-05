[TOC]

## 目标

* 了解动态切换Redis数据库

* 了解Spring提供的一些注解和接口

  参考：[SpringBoot2+Redis动态切换db数据源(db)最佳实践](https://blog.csdn.net/ron03129596/article/details/108847907)  推荐参考

  [PostConstruct官方说明](https://docs.oracle.com/javaee/5/api/javax/annotation/PostConstruct.html)

  [如何动态切换数据库 ](https://github.com/redisson/redisson/issues/1698)

  [为什么 Redis 默认 16 个库？90% 以上程序员不知道！](https://zhuanlan.zhihu.com/p/121358605)

  [EnvironmentAware接口的作用 ](https://blog.csdn.net/bazhuayu_1203/article/details/78658196)

  [SpringBoot属性绑定Environment和Binder](https://blog.csdn.net/jiachunchun/article/details/94569179)
  [Spring Boot 又升级了？2.0 你搞懂了吗？！](https://zhuanlan.zhihu.com/p/50412242)

  [Spring Boot中的属性绑定的实现](https://www.jb51.net/article/159332.htm)

  

### 前言

在某些场景下，不同业务服务使用同一个Redis数据库，为了以便于得到相对独立的数据库，需要一个服务对应一个数据库，在Redis中默认提供了16个数据库（序号0-15），那么就需要动态的切换数据库。

切换数据库命令,只需要选择序号即可。

```shell
127.0.0.1:0>select 1
"OK"
```

默认的数据库数量可以通过修改Redis配置文件修改.参考：[Redis 配置](https://www.runoob.com/redis/redis-conf.html)

```shell
# 设置数据库的数量，默认数据库为0，可以使用SELECT 命令在连接上指定数据库id
databases 16
```

注意在集群模式下，不支持使用select命令来切换db，**因为Redis集群模式下只有一个db0**。

## 动态切换数据库

以下内容基于  Spring boot 2.2.7.RELEASE、 Spring data Redis 、 Redisson框架 环境下。

### 思路

#### 第一种：

最简单的想法，每次需要切换数据库时，就执行切换数据库命令，拿到切换后的Rredis连接，再操作Redis数据库即可。(**先说明这个方式有很大问题，会有线程并发安全等问题，不过建议了解一下**)

映射到代码中 RedisTemplate并没有提供执行切换数据库的方法，只能在RedisConnectionFactory中指定。

在Redis 提供的连接方式，选择数据库必须Config对象中指定，才能生效。

代码示例：

```java
   // -- RedissonClient 是Redisson 框架提供的

    @Autowired
    private RedisTemplate redisTemplate;
    /**
     * 切换数据库
     *
     * @return
     */
    public RedisTemplate switchDatabase(Integer dbIndex) {
        logger.info("重新建立redis数据库连接开始");
        // 配置类
        Config config = new Config();
        config.setCodec(StringCodec.INSTANCE);
        SingleServerConfig singleConfig = config.useSingleServer();
        singleConfig.setAddress("redis://127.0.0.1:6379");
        singleConfig.setPassword("***");
        // 指定连接的数据库
        singleConfig.setDatabase(dbIndex);
        RedissonClient redissonClient = Redisson.create(config);
        RedissonConnectionFactory redisConnectionFactory = new RedissonConnectionFactory(redissonClient);
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        logger.info("重新建立redis数据库连接结束");
        return redisTemplate;
    }
```

测试代码：

```java

    /**
     * 动态切换数据库，并设置数据
     * @param code 键
     * @param dbindex Redis 数据库序号  
     */
public ReturnData dynamicDatabaseSelection(@RequestParam("code") String code, @RequestParam("dbindex") Integer dbindex) {
        Set<ZSetOperations.TypedTuple<Object>> tuples = new HashSet<>();
        DefaultTypedTuple typedTuple = new DefaultTypedTuple("zhangsan", 88D);
        DefaultTypedTuple typedTuple1 = new DefaultTypedTuple("zhangsan", 77D);
        DefaultTypedTuple typedTuple2 = new DefaultTypedTuple("lisi", 68D);
        DefaultTypedTuple typedTuple3 = new DefaultTypedTuple("wangwu", 120D);
        tuples.add(typedTuple);
        tuples.add(typedTuple1);
        tuples.add(typedTuple2);
        tuples.add(typedTuple3);
        redisUtil.switchDatabase(dbindex);
//        try {
//            logger.info("当前业务执行业务开始");
//            Thread.sleep(30000);
//            logger.info("耗时结束");
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
    	// 执行切换数据库
        redisUtil.zsSetAndTime(code, tuples);
        // 设置数据
        Set<ZSetOperations.TypedTuple<Object>> set = redisUtil.zsGetReverseWithScores(code, 0, -1);
        ReturnData ret = ReturnData.newInstance();
        ret.setSuccess();
        ret.setMessage(set);
        return ret;
    }
```

正常情况下， 执行上述代码会正常切换到对应数据库，并设置数据。那么采用这种方式，就可以解决动态切换数据库的问题了吗？会有以下问题

* 如果有多个请求同时进入，当第一个请求切换数据库触发，另一个请求同时切换数据库。共用了同一个RedisTemplate会导致第一个请求的设置的数据，可能被写入到第二个请求切换的数据库中，会出现线程并发问题。（**可以把线程休眠的代码放开，测试**）

* 程序每次切换数据库，都要重新与Redis服务器建立连接，耗时长（如下连接大约耗时1秒，如果多个请求同时切换，连接速度将更加缓慢）。

  ```java
  // 连接日志
  2021-01-05 21:28:46.105 INFO  http-nio-8089-exec-10 fun.gengzi.codecopy.dao.RedisUtil Line:928 - 重新建立redis数据库连接开始
  2021-01-05 21:28:46.316 INFO  http-nio-8089-exec-10 org.redisson.Version Line:41  - Redisson 3.12.0
  2021-01-05 21:28:46.590 INFO  redisson-netty-22-18 org.redisson.connection.pool.MasterPubSubConnectionPool Line:168 - 1 connections initialized for 127.0.0.1/127.0.0.1:6379
  2021-01-05 21:28:47.560 INFO  redisson-netty-22-19 org.redisson.connection.pool.MasterConnectionPool Line:168 - 24 connections initialized for 127.0.0.1/127.0.0.1:6379
  2021-01-05 21:28:47.563 INFO  http-nio-8089-exec-10 fun.gengzi.codecopy.dao.RedisUtil Line:938 - 重新建立redis数据库连接结束
  
  ```

  

* 当切换多次，会重复创建多个Rediscliet ，浪费资源。当系统存在多个 RedisClient 势必要占用内存和线程数 ，并对Redis服务器保持链接，占用服务器资源。

```shell
# info 查看Redis服务器信息
# info clients 已连接客户端信息，包含以下域：	
    # connected_clients : 已连接客户端的数量（不包括通过从属服务器连接的客户端）
    # client_longest_output_list : 当前连接的客户端当中，最长的输出列表
    # client_longest_input_buf : 当前连接的客户端当中，最大输入缓存
    # blocked_clients : 正在等待阻塞命令（BLPOP、BRPOP、BRPOPLPUSH）的客户端的数量
127.0.0.1:0>info clients
"# Clients
connected_clients:303
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0

# 查询内存占用
127.0.0.1:0>info memory
```

使用Redis Desktop Manager 工具，也可以查看。每切换一次数据库，增加25个连接。

![sAtTjs.png](https://s3.ax1x.com/2021/01/05/sAtTjs.png)

#### 第二种

了解到第一种方式的问题，目的在于解决上述问题。考虑到使用Redis 数据库个数是有限的，为每一个Redis 数据库创建一个连接池，不用每次切换都重新创建，复用之前的连接即可。上述有请求并发安全问题，最好是每一个数据库建立一个RedisTemplate，使用Map<String, RedisTemplate>来存储RedisTemplate，需要哪个RedisTemplate，就根据数据库序号获取RedisTemplate进行操作数据库。

那么在项目初始化时，就将需要使用的数据库连接建好是不错的选择。

## 代码实现

环境： Redis 5.0.8 版本、Redis Desktop Manager 工具

开发环境： Spring boot 2.2.7.RELEASE、 Spring data Redis 、 Redisson框架



### 构建多个RedisTemplate

#### yml 配置

在application.yml 加入以下自定义配置

```yml
redissondb:
  address: "redis://127.0.0.1:6379"
  password: 111
  # 数据库序号集合
  databases: [2,3,4,5,6]   
```

#### 初始化

读取配置初始化 RedisTemplate Bean

代码参考：[RedisBeanInit.java](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/config/RedisBeanInit.java)

初始化类，在执行前会先执行RedisRegister 类，然后从Spring容器中获取redisTemplate ，将其设置到Map中。

```java
/**
 * <h1>RedisTemplate 初始化类 </h1>
 *
 * @author gengzi
 * @date 2020年12月16日22:38:46
 */
@AutoConfigureBefore({RedisAutoConfiguration.class})  // 要在RedisAutoConfiguration 自动配置前执行
@Import(RedisRegister.class) // 配置该类前，先加载 RedisRegister 类
@Configuration // 配置类
// 实现 EnvironmentAware 用于获取全局环境
// 实现 ApplicationContextAware 用于获取Spring Context 上下文
public class RedisBeanInit implements EnvironmentAware, ApplicationContextAware {
    private Logger logger = LoggerFactory.getLogger(RedisBeanInit.class);

    // 用于获取环境配置
    private Environment environment;
    // 用于绑定对象
    private Binder binder;
    // Spring context
    private ApplicationContext applicationContext;
    // 线程安全的hashmap
    private Map<String, RedisTemplate> redisTemplateMap = new ConcurrentHashMap<>();

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    /**
     * 设置环境
     *
     * @param environment
     */
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
        this.binder = Binder.get(environment);
    }


    @PostConstruct // Constructor >> @Autowired >> @PostConstruct 用于执行一个 非静态的void 方法，常应用于初始化资源
    public void initAllRedisTemlate() {
        logger.info("<<<初始化系统的RedisTemlate开始>>>");
        RedissondbConfigEntity redissondb;
        try {
            redissondb = binder.bind("redissondb", RedissondbConfigEntity.class).get();
        } catch (Exception e) {
            logger.error("读取redissondb环境配置失败", e);
            return;
        }
        List<Integer> databases = redissondb.getDatabases();
        if (CollectionUtils.isNotEmpty(databases)) {
            databases.forEach(db -> {
                Object bean = applicationContext.getBean("redisTemplate" + db);
                if (bean != null && bean instanceof RedisTemplate) {
                    redisTemplateMap.put("redisTemplate" + db, (RedisTemplate) bean);
                } else {
                    throw new RrException("初始化RedisTemplate" + db + "失败，请检查配置");
                }
            });
        }
        logger.info("已经装配的redistempleate，map:{}", redisTemplateMap);
        logger.info("<<<初始化系统的RedisTemlate完毕>>>");
    }

    @Bean
    public RedisManager getRedisManager() {
        return new RedisManager(redisTemplateMap);
    }
}
```

代码参考：[**RedisManager.java**](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/config/RedisManager.java)

用于根据数据库序号获取对应的RedisTemplate

```java
/**
 * <h1>redis管理</h1>
 *
 * @author gengzi
 * @date 2020年12月16日22:37:18
 */
public class RedisManager {

    private Map<String, RedisTemplate> redisTemplateMap = new ConcurrentHashMap<>();

    /**
     * 构造方法初始化 redisTemplateMap 的数据
     *
     * @param redisTemplateMap
     */
    public RedisManager(Map<String, RedisTemplate> redisTemplateMap) {
        this.redisTemplateMap = redisTemplateMap;
    }

    /**
     * 根据数据库序号，返回对应的RedisTemplate
     *
     * @param dbIndex 序号
     * @return {@link RedisTemplate}
     */
    public RedisTemplate getRedisTemplate(Integer dbIndex) {
        RedisTemplate redisTemplate = redisTemplateMap.get("redisTemplate" + dbIndex);
        if (redisTemplate == null) {
            throw new RrException("Map不存在该redisTemplate");
        }
        return redisTemplate;
    }

}
```



代码参考：[**RedisBeanInit.java**](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/config/RedisBeanInit.java)

RedisTemplate Bean 初始化类，用于读取yml配置，创建多个RedisTemplate Bean 并注册到Spring容器。

```java
/**
 * <h1>redistemplate初始化</h1>
 * <p>
 * 作用：
 * <p>
 * 读取系统配置，系统启动时，读取redis 的配置，初始化所有的redistemplate
 * 并动态注册为bean
 *
 * @author gengzi
 * @date 2021年1月5日22:16:29
 */
@Configuration
// 实现 EnvironmentAware 用于获取环境配置
// 实现 ImportBeanDefinitionRegistrar 用于动态注册bean
public class RedisRegister implements EnvironmentAware, ImportBeanDefinitionRegistrar {

    private Logger logger = LoggerFactory.getLogger(RedisRegister.class);

    // 用于获取环境配置
    private Environment environment;
    // 用于绑定对象
    private Binder binder;

    /**
     * 设置环境
     *
     * @param environment
     */
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
        this.binder = Binder.get(environment);
    }

    /**
     * 注册bean
     *
     * @param importingClassMetadata
     * @param registry
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        logger.info("《《《动态注册bean开始》》》");
        RedissondbConfigEntity redissondb;
        try {
            redissondb = binder.bind("redissondb", RedissondbConfigEntity.class).get();
        } catch (Exception e) {
            logger.error("读取redissondb环境配置失败", e);
            return;
        }
        List<Integer> databases = redissondb.getDatabases();
        if (CollectionUtils.isNotEmpty(databases)) {
            databases.forEach(db -> {
                // 单机模式，集群只能使用db0
                Config config = new Config();
                config.setCodec(StringCodec.INSTANCE);
                SingleServerConfig singleConfig = config.useSingleServer();
                singleConfig.setAddress(redissondb.getAddress());
                singleConfig.setPassword(redissondb.getPassword());
                singleConfig.setDatabase(db);
                RedissonClient redissonClient = Redisson.create(config);
                // 构造RedissonConnectionFactory
                RedissonConnectionFactory redisConnectionFactory = new RedissonConnectionFactory(redissonClient);
                // bean定义
                GenericBeanDefinition redisTemplate = new GenericBeanDefinition();
                // 设置bean 的类型
                redisTemplate.setBeanClass(RedisTemplate.class);
                // 设置自动注入的形式，根据名称
                redisTemplate.setAutowireMode(AutowireCapableBeanFactory.AUTOWIRE_BY_NAME);
                // redisTemplate 的属性配置
                redisTemplate(redisTemplate, redisConnectionFactory);
                // 注册Bean
                registry.registerBeanDefinition("redisTemplate" + db, redisTemplate);
            });
        }
        logger.info("《《《动态注册bean结束》》》");

    }

    /**
     * redisTemplate 的属性配置
     *
     * @param redisTemplate          泛型bean
     * @param redisConnectionFactory 连接工厂
     * @return
     */
    public GenericBeanDefinition redisTemplate(GenericBeanDefinition redisTemplate, RedisConnectionFactory redisConnectionFactory) {
        RedisSerializer<String> stringRedisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        // 解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        // key采用String的序列化方式，value采用json序列化方式
        // 通过方法设置属性值
        redisTemplate.getPropertyValues().add("connectionFactory", redisConnectionFactory);
        redisTemplate.getPropertyValues().add("keySerializer", stringRedisSerializer);
        redisTemplate.getPropertyValues().add("hashKeySerializer", stringRedisSerializer);
        redisTemplate.getPropertyValues().add("valueSerializer", jackson2JsonRedisSerializer);
        redisTemplate.getPropertyValues().add("hashValueSerializer", jackson2JsonRedisSerializer);
        return redisTemplate;
    }
}
```

要点：根据序号集合，使用GenericBeanDefinition循环创建redisTemplate多个bean，再使用BeanDefinitionRegistry 将这些Bean注册到Spring容器,再根据序号，将其加入到Map中。 这里使用了Binder 将自定义配置映射成为  RedissondbConfigEntity 如下：

代码参考：[**RedissondbConfigEntity.java** ](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/entity/RedissondbConfigEntity.java)

yml配置对象属性映射类

```java
/**
 * <h1>redisconfig 配置实体类</h1>
 *
 * @author gengzi
 * @date 2020年12月16日14:09:41
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class RedissondbConfigEntity implements Serializable {
    // 地址
    private String address;
    // 密码
    private String password;
    // 所有的db序号
    private List<Integer> databases = new ArrayList<>();
}
```

####工具方法

代码参考：[RedisUtil.java](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/dao/RedisUtil.java)

```java
   
    @Autowired
    private RedisManager redisManager; 
   /**
     * zset 添加元素
     *
     * @param key
     * @param tuples
     * @return
     */
    public long zsSetAndTime(String key, Set<ZSetOperations.TypedTuple<Object>> tuples, Integer db) {
        try {
            // 根据db序号，获取对应的 RedisTemplate
            RedisTemplate redisTemplate = redisManager.getRedisTemplate(db);
            Long count = redisTemplate.opsForZSet().add(key, tuples);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
```

### 测试方法

```java
    @ApiOperation(value = "redis 动态选择数据库 新方式", notes = "redis 动态选择数据库 新方式")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "code", value = "code", required = true),
            @ApiImplicitParam(name = "dbindex", value = "dbindex", required = true)})
    @PostMapping("/dynamicDatabaseSelectionNew")
    @ResponseBody
    public ReturnData dynamicDatabaseSelectionNew(@RequestParam("code") String code, @RequestParam("dbindex") Integer dbindex) {
        Set<ZSetOperations.TypedTuple<Object>> tuples = new HashSet<>();
        DefaultTypedTuple typedTuple = new DefaultTypedTuple("zhangsan", 88D);
        DefaultTypedTuple typedTuple1 = new DefaultTypedTuple("zhangsan", 77D);
        DefaultTypedTuple typedTuple2 = new DefaultTypedTuple("lisi", 68D);
        DefaultTypedTuple typedTuple3 = new DefaultTypedTuple("wangwu", 120D);
        tuples.add(typedTuple);
        tuples.add(typedTuple1);
        tuples.add(typedTuple2);
        tuples.add(typedTuple3);
        // 选择数据库，并设置数据
        redisUtil.zsSetAndTime(code, tuples, dbindex);
        Set<ZSetOperations.TypedTuple<Object>> set = redisUtil.zsGetReverseWithScores(code, 0, -1, dbindex);
        ReturnData ret = ReturnData.newInstance();
        ret.setSuccess();
        ret.setMessage(set);
        return ret;
    }
```

#### 启动日志

```java
2021-01-05 22:38:54.274 INFO  main fun.gengzi.codecopy.business.redis.config.RedisRegister Line:89  - 《《《动态注册bean开始》》》
2021-01-05 22:38:54.727 INFO  main org.redisson.Version Line:41  - Redisson 3.12.0
2021-01-05 22:38:57.326 INFO  redisson-netty-2-19 org.redisson.connection.pool.MasterPubSubConnectionPool Line:168 - 1 connections initialized for 127.0.0.1/127.0.0.1:6379
2021-01-05 22:38:57.325 INFO  redisson-netty-2-18 org.redisson.connection.pool.MasterConnectionPool Line:168 - 24 connections initialized for 127.0.0.1/127.0.0.1:6379
// -- 忽略中间连接Redis数据库日志
2021-01-05 22:39:01.385 INFO  main fun.gengzi.codecopy.business.redis.config.RedisRegister Line:122 - 《《《动态注册bean结束》》》

2021-01-05 22:39:13.307 INFO  main fun.gengzi.codecopy.business.redis.config.RedisBeanInit Line:68  - <<<初始化系统的RedisTemlate开始>>>
2021-01-05 22:39:13.682 INFO  main fun.gengzi.codecopy.business.redis.config.RedisBeanInit Line:87  - 已经装配的redistempleate，map:{redisTemplate6=org.springframework.data.redis.core.RedisTemplate@60e5d4fb, redisTemplate5=org.springframework.data.redis.core.RedisTemplate@19b8bcb5, redisTemplate2=org.springframework.data.redis.core.RedisTemplate@36d9efa6, redisTemplate4=org.springframework.data.redis.core.RedisTemplate@f11fad9, redisTemplate3=org.springframework.data.redis.core.RedisTemplate@68812b74}
2021-01-05 22:39:13.683 INFO  main fun.gengzi.codecopy.business.redis.config.RedisBeanInit Line:88  - <<<初始化系统的RedisTemlate完毕>>>

```

当在多次切换数据库，不会增加数据库连接，也不会出现请求并发问题。

可以观察 Redis Desktop Manager  服务器信息的变化，看是否达到预期。

### 注意

其中使用了不少Spring提供的注解和接口，有必要了解一下，平时有些注解和接口是不怎么用到的。