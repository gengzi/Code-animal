[TOC]

## 目标

* 了解 java8 在项目中的使用

  参考： [这些Java8官方挖的坑，你踩过几个？](https://blog.csdn.net/u013256816/article/details/106485126/)

  [关于Java Stream的使用心得](https://blog.csdn.net/Cceking/article/details/85417661?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

  [JDK8 stream 在项目中的应用](https://blog.csdn.net/daxiaoge111/article/details/99935647)

  [Java 8 开发的 4 大顶级技巧](https://www.cnblogs.com/javastack/p/13030786.html)

  [Java8——优化 ](https://blog.csdn.net/rubulai/article/details/88710318)

## 实战

项目中使用：

### 函数式接口，默认方法

在设计接口时，如果有需求，可以设计函数式接口，使用。

或者使用java8提供的函数式接口使用。

默认方法用于扩展原有的接口，或者在接口中增加静态方法

fun.gengzi.codecopy.java8.lamdba.utils.MyComparator

```java
// 这里使用     Supplier  Consumer 两个函数式接口，实现了一个通用的设置信息的方法
/**
     * 通用身份证设置性别
     * <p>
     * supplier 产生一个给定类型的结果  不需要参数
     * <p>
     * consumer 不返回结果，但是需要一个参数 ，比如 集合.foreach  用的就是 consumer
     */
    private void setSexFromIdCardNo(String idCardNo, Supplier<String> supplier, Consumer<String> consumer) {
        boolean flag = StringUtils.isNoneBlank(idCardNo) && StringUtils.isNoneBlank(supplier.get()) && !"0".equals(supplier.get());
        if (flag) {
            int genderByIdCard = IdcardUtil.getGenderByIdCard(idCardNo);
            if (genderByIdCard == 1) {
                consumer.accept(backInfo("m"));
            } else if (genderByIdCard == 0) {
                consumer.accept("F");
            }
        }
    }
 
// 调用，使用对象的get 和 set 方法，为对象设置性别
MyComparator myComparator = new MyComparator();
setSexFromIdCardNo("410327188510154456", myComparator::getSex, myComparator::setSex);
System.out.println(myComparator.getSex());
```

 

### Optional

对于设计接口的某些方法返回值，可以考虑设计为 Optional<xx> 修饰，以避免在调用这些接口时，出现空指针异常。

但是对于某些入参字段的判空校验，依然使用 （ == null ）校验方式

具体参考：https://github.com/gengzi/codecopy/

fun.gengzi.codecopy.business.authentication.service.impl.AuthenticationServiceImpl

```java
// 不建议使用 isPresent 方法和 get 方法来判断和获取值，因为跟 判空操作没有区别,而且并不直观
// 类似 if(!optional.isPresent()){ String s = optinal.get();}
// 在使用中，考虑在返回空时，将其转换为 其他对象，或者抛出异常 的情况使用
 
// AES 根据密钥解密签名内容，返回一个 Optional<String> 如果解密失败，抛出认证失败异常。由全局的异常处理器捕获，响应至前台
final String reqParams = AESUtils.decrypt(sign, aeskey)
                .orElseThrow(() -> new RrException("认证失败", RspCodeEnum.FAILURE.getCode()));
 
// RSA 加密，加密失败，会将加密内容设置为 空字符串，返回
String numNo = RSAUtils.encrypt("1", secretkey).orElse("");
 
```

 

### Stream

#### 枚举类

枚举类提供，获取code 或者 获取其他字段的 Map 集合

之前的实现可能是 根据枚举的 values 方法循环，查找

java8实现：

枚举的 values() 返回一个 枚举类型的数组，通过 Arrays.stream 转成流，通过 collect 将其转换为 Map。每次根据Map 的key 获取value

具体代码参考： https://github.com/gengzi/codecopy/blob/master/src/main/java/fun/gengzi/codecopy/constant/RspCodeEnum.java

```java
 
 
// key 返回码 value 描述, 注意如果 key 重复，会报异常，设置第三个参数，规定值的变化，使用 新值替换旧值
    public static final Map<Integer, String> CODE_TO_DESC = Arrays.stream(values()).collect(
            Collectors.toMap(RspCodeEnum::getCode, RspCodeEnum::getDesc, (oldValue, newValue) -> newValue));
    /**
     * 根据响应码 返回 描述信息
     *
     * @param code 返回码
     * @return 描述信息
     */
    public static String fromCodeToDesc(Integer code) {
        return CODE_TO_DESC.getOrDefault(code, FAILURE.desc);
    }
 
```

#### for循环

对于简单的循环，或者需要index 下标的，依然可以使用普通for 循环实现。对于集合类的，可以使用java8 forEach 实现

```
xx.forEach( param -> { });
```

#### 集合类过滤和映射等操作

一般在获取从dao层拿到的数据，可能会进行过滤和映射，可以使用Stream 提供的方法来实现。

### 时间API

不建议依靠java8 自己实现时间工具类，可以使用一些开源组件的时间工具类。

比如 commons-lang3，或者 hutool 等等

## java8 其他改变

* caffeine: Java 8高性能缓存库包 , 据评测性能比 Guava 更好
* 重复注解
* Nashorn JavaScript引擎
* 优化 hashmap 结构 ，数组+红黑树
* 将 方法区 变成了 元空间

