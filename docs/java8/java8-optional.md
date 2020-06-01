[TOC]

## 目标

* 了解java8 提供的 Optional 用法

  参考： java8 实战 书籍

## Optional 解决的问题

* 避免空指针异常（NullPointerException）

  经常遇到的异常就是空指针异常，具体表现  对象实例.调用方法，提示对象实例是一个null，抛出异常。

  使用 Optional 修饰对象，避免出现空指针异常

* 优化判空校验过程

  通常在业务代码中，包含大量的判空校验，比如传入的参数，调用第三方接口返回结果，调用方法返回的结果

  具体代码：

  ```java
  // 请求参数判空
  if(param1 == null){
      throw new IllegalArgumentException("param1 参数为空");
  }
  if(param2 == null){
      throw new IllegalArgumentException("param2 参数为空");
  }
  // 第三方接口 或者 调用方法
   Person person = xxx.getinfo(id);
   if(person != null){
       if(person.getUsername() != null ){
           // xxx
       }
   }else {
       return "error";
   }
 
  ```

  使用Optional 减少繁杂的判空检验。但并不是要消除空指针异常的出现，对于某些你确定它不能为null的对象实例，或者数据，就不要添加判空校验，如果出现问题了，直接抛出NullPointerException，你会很清楚哪里出现了问题

  参考 java8 实战书籍：

  引入Optional 类的意图并非要消除每一个null引用。与此相反，它的目标是帮助你更好地设计出普适的API， 让程序员看到方法签名，就能了解它是否接受一个Optional的值。

 

## Optional api 了解

```
// 三种创建方式
Optional.empty(); 返回一个空的 Optional
Optional.of(xx);  依照一个 xx 的非空对象，获取一个Optional
Optional.ofNullable(xx); 可接受一个 xx 的空对象，获取一个 optional
```

![tJqolj.png](https://s1.ax1x.com/2020/06/01/tJqolj.png)

##  实践

### 调用其他接口或者方法返回结果，判空优化

以下代码中的对象示例，可以参考  https://github.com/gengzi/codecopy.git 项目，

fun.gengzi.codecopy.business.shorturl.controller.ShortUrlGeneratorController  实践

fun.gengzi.codecopy.java8.lamdba.utils.MyOptional 测试Optional 示例

通常对于调用第三方接口或者是在微服务中调用其他系统提供的服务，都需要做判空校验

```java
// 旧判断
// 从一个接口获取一个结果对象，判断对象是否为null，为null 抛出异常，使用全局的异常处理器，响应到前端
// 不为null，判断其中一个字段是否为null，或者是空字符串，是，抛异常，不是执行else 代码
Shorturl shortUrl = prescriptionShortUrlGeneratorService.getPrescriptionShortUrl(longurl);
if(shorturlRsp == null ){
            throw new RrException("访问接口失败", RspCodeEnum.FAILURE.getCode());
        }else{
            if(shorturlRsp.getShorturl() == null || StringUtils.isBlank(shorturlRsp.getShorturl())){
                throw new RrException("转换失败", RspCodeEnum.FAILURE.getCode());
            }else{
                // 执行业务逻辑，如果还需要判断的，继续写 判空操作
            }
        }
// ------------------ 使用 optional 优化 --------------------
// 校验接口响应结果
// 如果shortUrl 为null，抛异常
Shorturl shorturl1 = Optional.ofNullable(shortUrl)
    .orElseThrow(() -> new RrException("访问接口失败", RspCodeEnum.FAILURE.getCode()));
// 校验某个字段
// 如果校验通过，shortUrlStr 就是对应的结果值，不通过，抛异常
String shortUrlStr = Optional.ofNullable(shortUrl)
    .map(s -> s.getShorturl())  // 映射对象中的shorturl字段
    .filter( s-> StringUtils.isNoneBlank(s)) // 判断字段不是一个空字符串
    .orElseThrow(() -> new RrException("转换失败", RspCodeEnum.FAILURE.getCode())); // orElseThrow  如果有值返回值，没有值，抛出一个异常
 
 
// 也可以使用  orElse 返回指定类型的值
String shortUrlStr = Optional.ofNullable(shorturlRsp).map(s -> s.getShorturl()).filter( s-> StringUtils.isNoneBlank(s)).orElse("默认值xxx");
 
// 也可以使用 orElseGet 返回一个方法生成的值
  String shortUrlStr2 = Optional.ofNullable(shorturlRsp).map(s -> s.getShorturl()).filter(s -> StringUtils.isNoneBlank(s)).orElseGet(() -> "生成一个值");
 
 
```

在设计接口api 的时候，可以直接返回 Optional<xxx>

```java
public interface PrescriptionShortUrlGeneratorService extends ShortUrlGeneratorService{
    Optional<Shorturl> getPrescriptionShortUrl(String longUrl);
}
```

在jpa 的CrudRepository 接口中，根据id 查询数据的接口，返回值就是 optional 修饰的

```java
    /**
     * Retrieves an entity by its id.
     *
     * @param id must not be {@literal null}.
     * @return the entity with the given id or {@literal Optional#empty()} if none found.
     * @throws IllegalArgumentException if {@literal id} is {@literal null}.
     */
    Optional<T> findById(ID id);
```

 

### 入参参数校验

针对controller 的参数校验，可以使用 @valid 相关注解，或者自定义注解的形式来进行校验处理。可以不使用Optional 来处理。

spring boot 2 版本以上，已经包含了该依赖。

pom：

```xml
<!--添加依赖-->
<dependency>
<groupId>javax.validation</groupId>
<artifactId>validation-api</artifactId>
<version>2.0.1.Final</version>
</dependency>
```

具体参考：

[@Valid注解是什么](https://blog.csdn.net/weixin_38118016/article/details/80977207)

[Controller 层参数校验方案](https://www.jianshu.com/p/aa8b3163b30a)  

[[Spring] Web层AOP方式进行参数校验](https://www.jianshu.com/p/e4f1b1ffc32e)

[[SpringMVC] Web层注解式参数校验](https://www.jianshu.com/p/abbc765c99d6)

[SpringBoot使用 ValidationApi 进行参数校验](https://www.jianshu.com/p/6e843e6b25d6)

 

### 其他的一些用法

#### faltmap

 将两层 Optional 合并成一个,将其返回。如果不存在值，返回一个空的Optional

第一个示例

```java
public class Person{
    Integer id;
    String name;
    Optional<List<Role>> roles;
    }

//  flatMap Optional<Optional<Roles>> 将两层 Optional 合并成一个，返回
public void fun04() {
        Person person = new Person();
        String id = Optional.ofNullable(person)
                .flatMap(person1 -> person1.getRoles())
                .map(roles -> roles.get(0).getId())
                .get();
    }
 
```

 第二个示例

```java
// 原业务判断
public String fun(Person person, Role role) {
        return "";
    }

public Optional<String> fun06(
	Optional<Person> person, Optional<Role> role) {
	if (person.isPresent() && role.isPresent()) {
		return Optional.of(fun(person.get(), role.get()));
	} else {
		return Optional.empty();
	}
}

// -------------------- 组合两个 optional 对象 -------------------
// 具体参考 java8 实战 书籍，10.3.5 两个 Optional 对象的组合

public  void fun05(){
   Optional<Person> person1 = Optional.ofNullable(new Person());
   Optional<Role> role = Optional.ofNullable(new Role());

        //  调用方法 ，有两个参数，一个是 person 一个是 role ，在方法中执行， 通过 flatmap 和 map 可以不用判断是否为 null
        Optional<String> optionalS = person1.flatMap(person2 -> role.map(role1 -> fun(person2, role1)));
}

```

* filter

```java
    /**
     * filter 进行过滤判断
     */
    @Test
    public void fun05(){
        Person person = new Person();
        String s = Optional.ofNullable(person)
                .filter(person1 -> person1.getName().equals("张三"))
                .map(person1 -> person1.getName()).orElse("xx");
    }
```

* 三种创建 optional 方式

```java
    /**
     * 创建optional 的三种方式
     */
    @Test
    public void fun01() {
        // 创建一个空的optional
        Optional<Object> empty = Optional.empty();

        // 依照一个非空值，获取一个optional
        // 如果为null ，会报空指针异常
        Optional<String> ss = Optional.of("ss");

        // 可接受null的optional
        Optional<Object> o = Optional.ofNullable(null);
    }
```



 