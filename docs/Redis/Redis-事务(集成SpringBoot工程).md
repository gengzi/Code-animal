[TOC]

## 目标

* 了解Redis事务

  参考：深入分布式缓存 书籍

  [spring-data-redis的事务操作深度解析--原来客户端库还可以攒够了事务命令再发？](https://www.cnblogs.com/grey-wolf/p/10142937.html)

  [Redis官方文档](http://www.redis.cn/topics/transactions.html)

 

## Redis 事务

### 命令

命令参考：https://www.redis.net.cn/order/3638.html

```shell
## 乐观锁
# watch 用于监视 key，一旦key 在事务之前被其他命令改动，会导致事务失败
127.0.0.1:0>watch  key
"OK"
# unwatch 取消监视 key，当事务执行完成，会取消所有监视的key
127.0.0.1:0> UNWATCH
OK
 
## 事务
# multi 事务开始，标识一个事物块的开始
127.0.0.1:0>multi
"OK"
# exec 事务提交，批量将事务内命令提交
127.0.0.1:0>multi
"OK"
127.0.0.1:0>set key1 111
"QUEUED"
127.0.0.1:0>expire key1 3000
"QUEUED"
127.0.0.1:0>exec
 1)  "OK"
 2)  "OK"
 3)  "OK"
 4)  "1"
 5)  "OK"
# discard 放弃事务，用于取消事务，放弃执行事务块内的所有命令
redis 127.0.0.1:6379> MULTI
OK
redis 127.0.0.1:6379> PING
QUEUED
redis 127.0.0.1:6379> SET greeting "hello"
QUEUED
redis 127.0.0.1:6379> DISCARD
OK
 
```

 

### 文档

[事务](http://www.redis.cn/topics/transactions.html) 建议阅读

事务可以一次执行多个命令.

- 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
- 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。(注意：这里的全部执行，包含遇到错误命令也会执行剩下的命令，不会因为一个错误命令，停止执行事务。Redis事务不支持回滚)

[EXEC](http://www.redis.cn/commands/exec.html) 命令负责触发并执行事务中的所有命令：

- 如果客户端在使用 [MULTI](http://www.redis.cn/commands/multi.html) 开启了一个事务之后，却因为断线而没有成功执行 [EXEC](http://www.redis.cn/commands/exec.html) ，那么事务中的所有命令都不会被执行。
- 另一方面，如果客户端成功在开启事务之后执行 [EXEC](http://www.redis.cn/commands/exec.html) ，那么事务中的所有命令都会被执行。

乐观锁，事务通过watch 命令为 Redis 事务提供 check-and-set （CAS）行为。

被 [WATCH](http://www.redis.cn/commands/watch.html) 的键会被监视，并会发觉这些键是否被改动过了。 如果有至少一个被监视的键在 [EXEC](http://www.redis.cn/commands/exec.html) 执行之前被修改了， 那么整个事务都会被取消， [EXEC](http://www.redis.cn/commands/exec.html) 返回[nil-reply](http://www.redis.cn/topics/protocol.html#nil-reply)来表示事务已经失败。

### 要点

* Redis事务可以执行一些列命令，是原子操作。
* Redis事务执行会顺序执行命令，正确的命令被执行，不正确的命令会报错，但是不影响正确命令的执行和结果。
* Redis事务不支持回滚，可以使用watch key 命令，实现乐观锁，以支持串行化的事物隔离级别。
* 需要由客户端处理事务的错误（选择回滚还是删除数据或者其他操作）

### 应用场景

需要原子操作。

期望每次执行一系列Redis命令。

## Spring 示例

环境： Redis 5.0.8 版本、Redis Desktop Manager 工具

开发环境： Spring boot 2.2.7.RELEASE、 Spring data Redis 、 Redisson框架

官方文档：[Spring  data redis 官方文档](https://docs.spring.io/spring-data/redis/docs/2.4.2/reference/html/#tx)

### 代码示例

代码参考：[RedisShowController.java](https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/business/redis/controller/RedisShowController.java)

```java
    @ApiOperation(value = "redis事务", notes = "redis事务")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "code", value = "code", required = true)})
    @PostMapping("/affair")
    @ResponseBody
    public ReturnData affair(@RequestParam("code") String code) {
        RedisTemplate redisTemplate = redisManager.getRedisTemplate(3);
        try {
            List<Object> txResults = (List<Object>) new SessionCallback<List<Object>>() {
                public List<Object> execute(RedisOperations operations) throws DataAccessException {
                    operations.watch("key");
                    operations.watch("zz");
                    operations.multi();
                    operations.opsForSet().add("key", "value1");
                    operations.opsForZSet().add("zz", "11", 77D);
                    //TODO 使用字符串类型，获取zset类型的数据，会导致报错
                   // operations.opsForValue().get("zz");
                    return operations.exec();
                }
            }.execute(redisTemplate);
        } catch (Exception e) {
            logger.error("执行redis事务失败，原因：{}", e.getMessage());
            // 执行回滚逻辑
            redisTemplate.delete("key");
            redisTemplate.delete("zz");
        }
 
        ReturnData ret = ReturnData.newInstance();
        ret.setSuccess();
        ret.setMessage("");
        return ret;
    }
```

要点：这里引入了 SessionCallback ，由于`RedisTemplate`不能保证在同一连接中运行事务中的所有操作。当需要以相同的方式执行多个操作时`connection`（例如在使用Redis事务时），Spring Data Redis提供了SessionCallback接口来支持同一个RedisTemplate 执行Redis事务中的命令。

代码说明： 当放开报错代码（使用错误的类型获取数据），执行事务，会导致事务失败。但是报错代码前面的命令都会执行成功。**需要手动的处理事物异常，可以根据业务的特点，来选择是回滚数据还是放弃数据，或者不做任何操作，抛出异常，程序中断。**

### 事务使用要点

* 只读操作放在写操作前
* 写操作不依赖于前序中的写操作的结果
* 使用乐观锁避免一致性的问题，但是对相同的key 并发事务成功率低。

### Redis事务与脚本

本质是一样的，都是为了实现原子化的操作。都不支持自动回滚，需要程序员来控制错误。只不过事务在最初的版本就提供了，后续版本中才加入脚本。不过在未来，可能会删除事务。使用脚本可能会更加简单。

 

 

 

 

 