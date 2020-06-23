[TOC]

## 目标

* 实践Redis布隆过滤器

  参考：[windows10安装最新的redis](https://www.cnblogs.com/MC-Curry/p/11066455.html)

  [Centos中安装Redis插件bloom-filter](https://www.cnblogs.com/nxjblog/p/12680235.html)

  [实例解读什么是Redis缓存穿透、缓存雪崩和缓存击穿](https://www.cnblogs.com/liuqiyun/p/10831638.html)

  [SpringBoot+Redis布隆过滤器防恶意流量击穿缓存的正确姿势](https://blog.csdn.net/lifetragedy/article/details/103945885) 推荐阅读

  [网易考拉缓存业务选型实践](https://zhuanlan.zhihu.com/p/66396320) 推荐阅读

  [Redis 的雪崩、穿透和击穿，如何应对](https://doocs.github.io/advanced-java/#/./docs/high-concurrency/redis-caching-avalanche-and-caching-penetration) 推荐阅读

  [springboot 相关学习项目](https://github.com/wyh-spring-ecosystem-student/spring-boot-student/tree/releases)

## Redis环境

要求： Redis版本在4.0 以上 需要安装redis-bloom插件

官方地址：

[redis](https://redis.io/)

[RedisBloom](https://github.com/RedisBloom/RedisBloom)

本安装在ubuntu18.0安装（windows10 的linux子系统），windows 版本的redis不知道是否支持redis-bloom插件。

### 安装Redis

```shell
## 注意ubuntu的镜像源，需要切换到国内的镜像源
# 更新软件
$ apt-get update
# 安装redis
$ apt-get install redis-server -y
# 启动redis
$ redis-server
## 启动界面，可以看到版本  Redis 4.0.9 (00000000/0) 64 bit
```

### 安装Redis-bloom插件

```shell
# 下载Redis-bloom,可以去看下最新的发行版本
$ wget https://github.com/RedisBloom/RedisBloom/archive/v2.2.2.tar.gz
# 解压
$ tar -zxvf v2.2.2.tar.gz
# 进入目录编译 make
## 注意，如果没有安装 make 需要先安装
$ apt-get install make -y
$ apt-get install gcc -y
# 编译完成后，会生成一个 rebloom.so
$ cd ./RedisBloom-2.2.2
$ make
# 将rebloom.so 位置配置到 redis 配置文件中
$ vi /etc/redis/redis.conf
$ loadmodule /mnt/f/ruanjiuanbao/RedisBloom-2.2.2/RedisBloom-2.2.2/redisbloom.so
# 重复redis 服务
$ service redis-server restart
```

### 测试 Redis-bloom 功能

计算布隆过滤器的容量可以计算，[Bloom Filter Calculator](https://krisives.github.io/bloom-calculator/)

```shell
$ redis-cli
## 创建一个布隆过滤器 出错率越接近0 ，对内存消耗越大，对cpu利用越高
# bf.reserve codehole(过滤器名称) 0.01(error_rate错误率) 100(initial_size初始尺寸)
$ 127.0.0.1:6379> bf.reserve test 0.01 100
# bf.add {key} {item} 添加元素
$ 127.0.0.1:6379> bf.add test 22
# bf.exists {key} {item}
$ 127.0.0.1:6379> bf.exists test 22
(integer) 1
## 不存在
127.0.0.1:6379> bf.exists test 11
(integer) 0
```

## 代码实践

https://github.com/gengzi/codecopy

springboot 2.2.7.RELEASE

数据库 mysql5.7：resources/db/product.sql                   

代码具体参考：fun.gengzi.codecopy.business.product  业务代码

fun.gengzi.codecopy.config 缓存配置

fun.gengzi.codecopy.dao redis工具

### 思路

本实现基于Spring boot cache  和 Spring boot redis

编写java操作redis布隆过滤器的方法（添加元素，判断是否存在元素）

从数据库中查询所有要被查询的key，存放到redis布隆过滤器

当请求进入，先查询是否在缓存中，有，返回。

无，查询key是否在redis布隆过滤器中，存在，查询数据库返回数据（查询结果为空，也缓存数据）

不存在，直接响应

### BloomFilterHelper

fun.gengzi.codecopy.dao.BloomFilterHelper

redis 借助于 bitmap 来实现。也可以参考 https://redislabs.com/ 实现java操作redis布隆过滤器

```java
 
import com.google.common.base.Preconditions;
import com.google.common.hash.Funnel;
import com.google.common.hash.Hashing;
 
/**
* 布隆过滤器工具类
*
* @param <T>
*/
public class BloomFilterHelper<T> {
    private int numHashFunctions;
 
    private int bitSize;
 
    private Funnel<T> funnel;
 
    public BloomFilterHelper(Funnel<T> funnel, int expectedInsertions, double fpp) {
        Preconditions.checkArgument(funnel != null, "funnel不能为空");
        this.funnel = funnel;
        bitSize = optimalNumOfBits(expectedInsertions, fpp);
        numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, bitSize);
    }
 
    int[] murmurHashOffset(T value) {
        int[] offset = new int[numHashFunctions];
 
        long hash64 = Hashing.murmur3_128().hashObject(value, funnel).asLong();
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);
        for (int i = 1; i <= numHashFunctions; i++) {
            int nextHash = hash1 + i * hash2;
            if (nextHash < 0) {
                nextHash = ~nextHash;
            }
            offset[i - 1] = nextHash % bitSize;
        }
 
        return offset;
    }
 
    /**
     * 计算bit数组的长度
     */
    private int optimalNumOfBits(long n, double p) {
        if (p == 0) {
            p = Double.MIN_VALUE;
        }
        return (int) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }
 
    /**
     * 计算hash方法执行次数
     */
    private int optimalNumOfHashFunctions(long n, long m) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }
}
```

### redis 工具类

fun.gengzi.codecopy.dao.RedisUtil

```java
/**
     * 根据给定的布隆过滤器添加值
     * @param bloomFilterHelper  bloom布隆过滤器解析类
     * @param key redis 的key
     * @param value redis 的value
     * @param <T> 值的类型
     */
    public <T> void addByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }
    }
 
 
    /**
     * 根据给定的布隆过滤器判断值是否存在
     * @param bloomFilterHelper bloom布隆过滤器解析类
     * @param key redis 的key
     * @param value redis 的value
     * @param <T> 值的类型
     * @return 存在 true
     */
    public <T> boolean includeByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }
        return true;
    }
```

### 业务类

fun.gengzi.codecopy.business.product.service.impl.ProductCacheServiceImpl

```java
    // redis 布隆过滤器
    private static final BloomFilterHelper<String> myBloomFilterHelper = new BloomFilterHelper<>(
            (Funnel<String>) (from, into) ->
                    into.putString(from, Charsets.UTF_8).putString(from, Charsets.UTF_8),
            1500000, 0.00001);
 
/**
     * 将可能的key 都存入Redis布隆过滤器
     */
    @Override
    public void putRedisBloomKey() {
        List<Integer> allId = productDao.getAllId();
        allId.forEach(id -> {
            redisUtil.addByBloomFilter(myBloomFilterHelper, ProductConstants.PRODUCT_ID, id + "");
        });
    }
 
    /**
     * 根据产品id 获取产品信息  -  解决缓存穿透： Redis布隆过滤器
     *
     * @param id
     * @return
     */
    @Cacheable(cacheManager = "loclRedisCacheManagers", value = "PRODUCT_INFO_ID_NOPREFIX", key = "#id", cacheNames = {"PRODUCT_INFO_ID_NOPREFIX"})
    @Override
    public Product getOneProductCacheInfoRedisBloom(Integer id) {
        boolean isContain = redisUtil.includeByBloomFilter(myBloomFilterHelper, ProductConstants.PRODUCT_ID, id + "");
        if (isContain) {
            return productDao.findById(id.longValue()).orElseThrow(() -> new RrException("error ", RspCodeEnum.FAILURE.getCode()));
        } else {
            throw new RrException("error ", RspCodeEnum.FAILURE.getCode());
        }
    }
```

### jmeter 压测

请看这位大佬的文章：[SpringBoot+Redis布隆过滤器防恶意流量击穿缓存的正确姿势](https://blog.csdn.net/lifetragedy/article/details/103945885)

这里引用一下结论：

大概可以知道，120万数据总计在redis中占用内存不超过8mb。

使用redis布隆过滤器，查询key 的效率是：平均在80ms左右。

生产中的实践：

**往往在生产上，我们经常会把上千万或者是上亿的记录"load"进bloomfilter，然后拿它去做“防击穿”或者是去重的动作。**

只要bloomfilter中不存在的key直接返回客户端false，配合着nginx的动态扩充、cdn、waf、接口层的缓存，整个网站抗6位数乃至7位数的并发其实是件非常简单的事。

 

 