[TOC]

## 目标

* 了解Redis Lua脚本知识

  参考:[使用redisTemplate设置过期时间是不是不能保证原子性？](https://bbs.csdn.net/topics/394868148)

  [在Redis中设置了过期时间的Key，需要注意哪些问题？](http://www.zzvips.com/article/75061.html)

  [Springboot整合Redis以及Lua脚本的使用](https://www.cnblogs.com/better-farther-world2099/articles/12197103.html)

  [SpringBoot通过redisTemplate调用lua脚本 并打印调试信息到redis log](https://blog.csdn.net/fsw4848438/article/details/81540495)
  [基于spring-data-redis的RedisTemplate 和 lua 脚本的 redis 分布式锁的实现](https://blog.csdn.net/shujianhua/article/details/90175566)

  [Lua利用cjson读写json](https://www.cnblogs.com/dalianpai/p/12730296.html)

  [Redis设置日志目录及loglevel](https://www.cnblogs.com/tusheng/articles/10282875.html)

  [lua 脚本示例](https://www.cnblogs.com/dalianpai/p/12730296.html)

  [lua 脚本 spring boot 示例](https://www.jianshu.com/p/366d1b4f0d13)

  [Lua 脚本语法说明 （支持 Lua 5.1）](https://blog.csdn.net/don211/article/details/7608912)

  [redis 的命令集合](https://www.redis.net.cn/order/3609.html)

  [redis 官方文档对 lua脚本的说明 和 示例](http://www.redis.cn/commands/eval.html)

  

### 提出一个问题？

在某些业务场景下，使用Redis作为缓存数据，需要先设置redis 的key和value值，再设置key的过期时间。思考一下，这是一个原子操作吗？

示例代码如下：

```java
    public long zsSetAndTime(String key, long time, Set<ZSetOperations.TypedTuple<Object>> tuples, Integer db) {
        try {
            RedisTemplate redisTemplate = redisManager.getRedisTemplate(db);
            // 设置缓存数据
            Long count = redisTemplate.opsForZSet().add(key, tuples);
            if (time > 0) {
                // 设置key的过期时间
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
```

很显然并不是一个原子操作（不可分割的），如果不是原子操作，可能设置值后再设置key的过期时间出错了，该key将永久存在于Redis服务器中。在某些苛刻的业务场景下，也是不允许的。

在Redis 2.0.0 版本以上提供了String 数据类型的 SETEX 命令，设置key对应字符串value，并且设置key在给定的seconds时间之后超时过期。

可参考： [SETEX key seconds value](http://www.redis.cn/commands/setex.html)

```shell
redis 127.0.0.1:6379> SETEX mykey 60 redis
OK
redis 127.0.0.1:6379> TTL mykey
60
redis 127.0.0.1:6379> GET mykey
"redis
```

SETEX是原子的，相当于将 SET 命令 和 EXPIRE 命令放在一个事务中执行。

RedisTemplate 也提供了对应的方法

```java
redisTemplate.opsForValue().set(key, value, time, timeUnit);
```

但是Redis 并没有提供对其他数据类型，设置值并设置超时时间的命令。

### 保证原子性操作的两种做法

* 事务

  Redis 提供了事务操作，事务可以一次执行多个命令，并且事务执行是一个隔离操作，并不会因为其他命令中断操作。事务是一个原子操作，事务中的命令要么全部被执行，要么全部都不执行。

  可参考：[事务](redis.cn/topics/transactions.html)

* Lua 脚本

  在Redis 2.6.0 版本提供了EVAL 命令执行Lua 脚本。

  Redis 使用单个 Lua 解释器去运行所有脚本，并且， Redis 也保证脚本会以原子性(atomic)的方式执行： 当某个脚本正在运行的时候，不会有其他脚本或 Redis 命令被执行。 这和使用 MULTI / EXEC 包围的事务很类似。

  可参考：[EVAL简介](http://www.redis.cn/commands/eval.html)

**本篇来介绍使用Lua脚本实现ZSET 类型的设值并设置超时时间的实现。**

## Redis Lua 脚本

推荐阅读：[EVAL](http://www.redis.cn/commands/eval.html)  建议阅读上述参考链接后，再查看以下内容。

### 命令

#### EVAL 命令

```shell
# EVAL 命令
# EVAL命令后是一段 Lua5.1的脚本程序 ，后面的 2 是指有几个key。zhangsan lisi 都是key，30 和 10 是后面的值
# KEYS[x] 是Redis 提供的键，从序号1开始，在lua 脚本中是全局变量
# ARGV[x] 是Redis 提供的值，从序号1开始，在lua 脚本中是全局变量
127.0.0.1:0>EVAL "return {KEYS[1], KEYS[2], ARGV[1], ARGV[2]} " 2 zhangsan lisi 30 10
 1)  "zhangsan"
 2)  "lisi"
 3)  "30"
 4)  "10
```

#### 调用执行Redis 命令

```shell
# redis.call() 和 redis.pcall() 方法
# 使用redis.call() 调用set 命令，设置数据
127.0.0.1:0>EVAL "return redis.call('set',KEYS[1],ARGV[1])" 1 testkey 11
"OK"

# call() 和 pcall() 方法的区别，call方法仅返回一个错误，pcall方法会捕获以lua表的形式返回

```

#### EVALSHA

```shell
# EVALSHA 命令
# EVALSHA命令的作用和EVAL命令一致，不过命令接收的参数是一个 SHA1 值，执行EVAL命令每次都要发送lua脚本主体，Redis为了避免多次编译同一脚本，浪费资源和时间，就将lua脚本主体转成SHA1 并加入缓存，这样下次再次执行该脚本，只需要传递 SHA1 ，就可以执行lua脚本了
> set foo bar
OK
> eval "return redis.call('get','foo')" 0
"bar"
> evalsha 6b1bf486c81ceb7edf3c093f4c48582e38c0e791 0
"bar"
> evalsha ffffffffffffffffffffffffffffffffffffffff 0
(error) `NOSCRIPT` No matching script. Please use [EVAL](/commands/eval).

```

客户端库的底层实现可以一直乐观地使用 EVALSHA 来代替 EVAL ，并期望着要使用的脚本已经保存在服务器上了，只有当 NOSCRIPT 错误发生时，才使用 EVAL 命令重新发送脚本，这样就可以最大限度地节省带宽。

其他相关内容，请参阅 [EVAL](http://www.redis.cn/commands/eval.html)

### 使用场景

需要原子操作。

期望每次执行一系列Redis命令。

### Lua 脚本语法

请参考:[Lua 脚本语法说明 （支持 Lua 5.1）](https://blog.csdn.net/don211/article/details/7608912)

**注意点：Redis 脚本不允许创建全局变量，避免引入全局变量的一个诀窍是：将脚本中用到的所有变量都使用 local 关键字定义为局部变量。**

### 常用类库

Redis为了方便操作使用lua脚本，默认集成了一系列的类库，以供使用：

具体参考：[可用库](http://www.redis.cn/commands/eval.html) 着重说下cjson lib. 在后续操作，使用的比较多

* cjson 类库

  参考：[Lua利用cjson读写json示例分享](https://www.jb51.net/article/57785.htm)

  主要用于操作Json，提供了解析Json对象和编译Json对象的功能

  官方文档：https://github.com/mpx/lua-cjson

  Lua CJSON模块为Lua提供JSON支持。

  简单的方法示例：

  ```lua
  local cjson = require "cjson"
  local sampleJson = [[{"age":"23","testArray":{"array":[8,9,11,14,25]},"Himi":"himigame.com"}]];
  --解析json字符串
  local data = cjson.decode(sampleJson);
  --打印json字符串中的age字段
  print(data["age"]);
  --打印数组中的第一个值
  print(data["testArray"]["array"][1]);  
  ```

  **并未找到json数组获取size的方法，所以在操作Json数组时，可能要手动指定数组中元素的个数**

  

### 调试与日志

编写的Lua脚本，需要查看执行中流程和一些元素的值，那么就需要打印日志，方便调试和查看。

在 Lua 脚本中，可以通过调用 redis.log 函数来写 Redis 日志(log)：

```lua
redis.log(loglevel,message)

-- 示例
redis.log(redis.LOG_WARNING, "Something is wrong with this script.")
-- 打印后，会在redis 服务器的log日志中，打印
```

两个参数：

* loglevel 日志级别 ，可以是以下任意一个值，这些等级(level)和标准 Redis 日志的等级相对应。

  redis.LOG_DEBUG
  redis.LOG_VERBOSE
  redis.LOG_NOTICE
  redis.LOG_WARNING

  **只有设置日志级别与Redis服务器设置的日志级别一致才能被打印。**

* message 打印的信息

#### 开启Redis 日志

编辑redis.conf 配置文件,定义日志级别和日志位置，重启Redis服务器

```shell
loglevel notice
logfile "/data/logs/redis.log"
```



## Spring boot 集成

环境： Redis 5.0.8 版本、Redis Desktop Manager 工具

开发环境： Spring boot 2.2.7.RELEASE、 Spring data Redis 、 Redisson框架

在开发之前，一定要开启Redis 的日志，方便之后的调试（一开始，我以为lua脚本执行一下就成功了，没打印日志，报错了半天，打印了日志，一目了然...）

业务实现：实现实时排行榜接口，使用Redis实现，并设置过期时间（保证原子性）。

### 资料参考

[Redistemplate 提供的lua 脚本执行示例](https://docs.spring.io/spring-data/redis/docs/2.4.2/reference/html/#scripting)

[Redisson框架提供的示例代码](https://github.com/redisson/redisson/wiki/10.-%E9%A2%9D%E5%A4%96%E5%8A%9F%E8%83%BD#104-%E8%84%9A%E6%9C%AC%E6%89%A7%E8%A1%8C)

### Controller

代码参考：[RedisShowController.java ](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/controller/RedisShowController.java)

```java
 @ApiOperation(value = "redis lua 实现zset添加数据并设置过期时间", notes = "redis lua 脚本测试")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "code", value = "code", required = true)})
    @PostMapping("/scriptTest")
    @ResponseBody
    public ReturnData scriptTest(@RequestParam("code") String code) {
        // 构造模拟数据
        Set<ZSetOperations.TypedTuple<Object>> tuples = new HashSet<>();
        DefaultTypedTuple typedTuple = new DefaultTypedTuple("zhangsan", 88D);
        DefaultTypedTuple typedTuple1 = new DefaultTypedTuple("zhangsan", 77D);
        DefaultTypedTuple typedTuple2 = new DefaultTypedTuple("lisi", 68D);
        DefaultTypedTuple typedTuple3 = new DefaultTypedTuple("wangwu", 120D);
        tuples.add(typedTuple);
        tuples.add(typedTuple1);
        tuples.add(typedTuple2);
        tuples.add(typedTuple3);
        ZsetAddEntity zsetAddEntity = new ZsetAddEntity();
        zsetAddEntity.setTuples(tuples);
        zsetAddEntity.setSize(tuples.size());
        LuaScriptExecEntity luaScriptExecEntity = new LuaScriptExecEntity();
        luaScriptExecEntity.setInfo(zsetAddEntity);
        luaScriptExecEntity.setTtl(30000);
	   // 将obj转为json字符串
        String jsonInfo = JsonUtils.objectToJson(luaScriptExecEntity);
        logger.info("lua 数据：{}", jsonInfo);

        RedisTemplate redisTemplate = redisManager.getRedisTemplate(3);
        // 序列化方式
        RedisSerializer<String> stringRedisSerializer = new StringRedisSerializer();
        // 调用lua 脚本
        String result = (String) redisTemplate.execute(script,stringRedisSerializer,stringRedisSerializer,Collections.singletonList(code), jsonInfo);
        ReturnData ret = ReturnData.newInstance();
        ret.setSuccess();
        ret.setMessage(result);
        return ret;
    }
```

核心代码：

redisTemplate.execute() 该方法提供了对lua脚本的执行。

```java
/*
使用提供的RedisSerializer来执行给定的RedisScript来序列化脚本参数和结果。
    参数：
    RedisScript–要执行的脚本
    argsSerializer –用于序列化args的RedisSerializer
    resultSerializer –用于序列化脚本返回值的RedisSerializer
    keys –任何需要传递给脚本的键
    args –需要传递给脚本的所有args
    返回值：
    脚本的返回值；如果RedisScript.getResultType()为null，则返回null，这可能表示抛出状态答复（即“ OK”）
*/
// 调用lua 脚本
String result = (String) redisTemplate.execute(script,stringRedisSerializer,stringRedisSerializer,Collections.singletonList(code), jsonInfo);
```

#### 注意

特别注意的是： 在传参给lua 脚本的时候，redistemplate 会把key 和value 进行默认的序列化（如果不指定的情况下）
默认的序列化，要看redisTemplate 是否配置了自定义的序列化，如果没有的话，就会采用默认的，jdk提供的序列化。

正常情况下，key 的序列化方式一般都是String，value一般是jdk的序列化，或者jackson2Json 等等。考虑到执行lua脚本，如果使用这些序列化方式，在操作数据的时候，Lua脚本没有对应的解析方法，很有可能解析失败，导致拿不到数据。所以，这里的key和value 的序列化，都使用了StringRedisSerializer 来处理，传递的Redis 数据部分，手动转为了Json字符串，在lua脚本中，该数据依然是一个字符串，使用cjson类库解析，就会非常简单。

```java
// 将obj转为json字符串
String jsonInfo = JsonUtils.objectToJson(luaScriptExecEntity);
logger.info("lua 数据：{}", jsonInfo);
```



### RedisScript

代码参考：[RedisLuaScriptConfig.java ](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/config/RedisLuaScriptConfig.java)

一般对于每一个脚本程序，都保存为一个lua后缀的脚本文件方便执行使用。

```java
@Configuration
public class RedisLuaScriptConfig {

    /**
     * 构造一个 script 脚本
     *
     * @return
     */
    @Bean
    public RedisScript<String> script() {
        Resource resource = new ClassPathResource("/redislua/zset_add_expire.lua");
        return RedisScript.of(resource, String.class);
    }

}
```



### Lua脚本程序

[zset_add_expire.lua](https://github.com/gengzi/codecopy/blob/master/src/main/resources/redislua/zset_add_expire.lua)

文件位置：src\main\resources\redislua\zset_add_expire.lua

```lua
redis.log(redis.LOG_NOTICE,"<<<脚本开始>>>")
-- 获取key
local key = KEYS[1]
-- 字符串拼接 .. 连接
redis.log(redis.LOG_NOTICE,"redis key：" .. key)
-- 解析json数据
local content = cjson.decode(ARGV[1])
-- 获取数组长度
local length = content["info"]["size"]
redis.log(redis.LOG_NOTICE,"arr length：" .. length)
local ttl = content["ttl"]
for i = 1, length , 1 do
    redis.log(redis.LOG_NOTICE,"分数：" .. content["info"]["tuples"][i]["score"])
    redis.log(redis.LOG_NOTICE,"姓名：" .. content["info"]["tuples"][i]["value"])
    -- 设置数据
    local current = redis.call('ZADD', key, content["info"]["tuples"][i]["score"] , content["info"]["tuples"][i]["value"])
end
-- 设置key的ttl
local num = redis.call('Expire', KEYS[1], ttl)
redis.log(redis.LOG_NOTICE,"<<<脚本结束>>>")
return "ok"
```



### 执行日志

打开Redis服务器的日志文件,可以看到打印的日志

```shell
1:M 23 Dec 2020 22:15:55.670 * <<<脚本开始>>>
1:M 23 Dec 2020 22:15:55.670 * redis key：aa
1:M 23 Dec 2020 22:15:55.670 * arr length：4
1:M 23 Dec 2020 22:15:55.670 * 分数：88
1:M 23 Dec 2020 22:15:55.670 * 姓名：zhangsan
1:M 23 Dec 2020 22:15:55.670 * 分数：77
1:M 23 Dec 2020 22:15:55.670 * 姓名：zhangsan
1:M 23 Dec 2020 22:15:55.670 * 分数：68
1:M 23 Dec 2020 22:15:55.670 * 姓名：lisi
1:M 23 Dec 2020 22:15:55.670 * 分数：120
1:M 23 Dec 2020 22:15:55.670 * 姓名：wangwu
1:M 23 Dec 2020 22:15:55.670 * <<<脚本结束>>>
```

查看是否存储成功

![1608734541401](https://s3.ax1x.com/2020/12/23/rcnzWD.png)



### 其他命令

```shell
# 选择库(序号)
> select 1 
```

## 注意

对于某些简单脚本，可以直接在Redis client 的命令行中测试，以便于快速测试。

在运行Spring boot 工程（Maven）测试时，最好先clean再运行，防止上一次的Lua脚本修改未生效。需要及时生效lua脚本，可以修改运行编译（target）下class找到lua 脚本修改，这样会及时生效，测试完毕后，再将内容拷贝至真实文件中即可。