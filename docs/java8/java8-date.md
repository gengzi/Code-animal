[TOC]

## 目标

* 了解java8 的date 相关操作

  参考：java8 实战书籍 第12章  java8 简明教程

## Date

java8 提供新的处理时间的api，是为了优化之前的时间处理方式。

```java
// 常见的几个类
// LocalDate  本地时间，年-月-日
// LocalTime   本地时间，时-分-秒-毫秒
// LocalDateTime 本地时间，年-月-日 时-分-秒-毫秒
// DateTimeFormatter  线程安全，格式化时间
 
// 常规方法
// 获取 localdate
LocalDate localDate = LocalDateTime.now().toLocalDate();
// 获取 localtime
LocalTime localTime = LocalDateTime.now().toLocalTime();
 
// 格式化时间
LocalDateTime now = LocalDateTime.now();
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
String format = dateTimeFormatter.format(now);
System.out.println(format);
 
// 其他方法，参见官方api
 
 
```

在阿里java开发手册中指出：

【强制】SimpleDateFormat 是线程不安全的类，一般不要定义为 static 变量，如果定义为 static，必须加锁，或者使用 DateUtils 工具类。

正例：注意线程安全，使用 DateUtils。亦推荐如下处理：

```java
// 为每个线程，绑定一个 simpdateformat
private static final ThreadLocal df = new ThreadLocal() {
     @Override
     protected DateFormat initialValue() {
         return new SimpleDateFormat("yyyy-MM-dd");
     }
};
```

 

说明：如果是 JDK8 的应用，可以使用 Instant 代替 Date，LocalDateTime 代替 Calendar， DateTimeFormatter 代替 SimpleDateFormat，官方给出的解释：simple beautiful strong immutable thread-safe。

网上提供的一些java8 date 工具类

参考：[jdk8时间工具类](https://blog.csdn.net/feicongcong/article/details/78224494)

[DateUtil工具类——基于jdk8APi，各类转换和时间处理非常齐全](https://blog.csdn.net/hejie1997/article/details/94596289)

也可以参考 唯品会提供的工具类，这个是基于jdk1.7

https://github.com/DarLiner/vjtools/blob/master/vjkit/src/main/java/com/vip/vjtools/vjkit/time/DateUtil.java

具体还是需要自己测试下

```java
// 提供几个经常用到的
 
    /**
     * 获取当前时间，限定时区
     *
     * @return LocalDateTime
     */
    public static LocalDateTime getCurrentLocalDateTime() {
        return LocalDateTime.now(Clock.system(ZoneId.of("Asia/Shanghai")));
    }
 
 
 
public static final String DATE_FMT_8 = "HH:mm:ss";
 
    /**
     * 判断当前时分秒，是否在规定范围中
     * @param start 开始时间
     * @param end  结束时间
     * @return 在，true ，不在 false
     */
    public boolean isTimeInRange(String start, String end) {
        if (StringUtils.isBlank(start) || StringUtils.isBlank(end)) {
            throw new IllegalArgumentException("some date parameters is null or blank");
        }
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(DATE_FMT_8);
        LocalTime startTime = LocalTime.from(dateTimeFormatter.parse(start));
        LocalTime endTime = LocalTime.from(dateTimeFormatter.parse(end));
        if(startTime.isAfter(endTime)){
            throw new IllegalArgumentException("some date parameters is dateBein after dateEnd");
        }
        LocalTime now = LocalTime.now();
        return (startTime.isBefore(now) && endTime.isAfter(now) || startTime.compareTo(now) == 0 || endTime.compareTo(now) == 0);
    }
 
// 测试
        boolean timeInRange = isTimeInRange("21:55:11", "22:55:11");
 
 
```

// TODO 整理好一个完整的 java8  未完待续
