[TOC]

## 目标

* 了解 spring security  认证授权流程

  参考：spring security 实战书籍

  [前后端分离Spring security 从healer的token获取Session](https://www.cnblogs.com/zhangxh20/p/13376920.html?utm_source=tuicool)

  [资源：在线生成ascii字符画网站](https://www.jianshu.com/p/fca56d635091)

  [spring-security 系列文章](https://github.com/BUG9/spring-security)

  小问题解决：[Invalid ON UPDATE clause for 'create_date' column](https://www.cnblogs.com/hiit/p/11313872.html)

 

spring security 中文文档：https://www.springcloud.cc/spring-security-zhcn.html

## spring security 实现认证鉴权的流程

（对于流程中的细节，可能不是真正的执行过程，因为我也没有一个代码代码走，但是不影响理解。）

security  通过认证，鉴权两个步骤来实现，整个认证鉴权过程。

参考：[官方文档](https://docs.spring.io/spring-security/site/docs/5.4.1/reference/html5/#servlet-hello-auto-configuration)

![1604826568790](index_files/1604826568790.png)

当启用 Spring Security 的配置，向每个请求的Servlet容器`Filter`命名的bean注册`springSecurityFilterChain` （spring安全过滤器链）

DelegatingFilterProxy 提供了在Servlet容器的生命周期和Spring的生命周期之间进行桥接，Servlet容器允许Filter使用其自己的标准注册，但是它不知道Spring定义的Bean。 DelegatingFilterProxy可以通过标准Servlet容器机制进行注册，但是将所有工作委托给实现的Spring Bean Filter。

**简单的理解，如果你想在security 认证授权过程中，加入自己的校验规则（比如增加校验图片验证码），可以实现一个标准的Filter，将其加入到SecurityFilterChain 。即可将这个Filter 生效。**

SecurityFilterChain 已经默认提供了一系列过滤器：

可以参考： https://docs.spring.io/spring-security/site/docs/5.4.1/reference/html5/#servlet-securityfilterchain

说几个重要的：

UsernamePasswordAuthenticationFilter  用户名密码认证过滤器

BasicAuthenticationFilter 基本认知过滤器

SessionManagementFilter session管理过滤器

ExceptionTranslationFilter 异常转换过滤器

（抛出 AccessDeniedException或AuthenticationException ，会执行ExceptionTranslationFilter）

FilterSecurityInterceptor  使用该类来实现授权

### 认证

security   提供了多种认证方式（用户名密码，oauth 2.0 登陆 等等）。

以最常用的基于自定义数据库的用户名和密码登陆为例，当用户发送登陆请求，将会携带用户的登陆名，密码进入系统，当认证过滤器拦截到当前请求的url （/login）是登陆path,先执行获取根据用户名获取用户详细信息，从用户详细信息中获取密码，与请求中的密码参数做对比，一致，就认为认证成功，不一致，就认为认证失败。

具体可以参考官方文档：https://docs.spring.io/spring-security/site/docs/5.4.1/reference/html5/#servlet-authentication-unpwd

几个比较关键的类：

UserDetailsService 根据用户名获取用户的详情（UserDetails）

UserDetails 用户详情类，包含了用户名，密码，权限集合，等等关于用户信息

**如果想实现自定义数据库表实现认证授权，只要把 UserDetailsService ，UserDetails  实现了，就对接上spring security 的认证授权功能。**

### 授权

在获取用户详细时，会将该用户的权限（角色，或者权限），已经设置了。那么用户访问什么内容，就会触发鉴权流程。

security 提供了一个投票决策管理器（AccessDecisionManager），通过投票的形式，来判断用户访问该内容是否已经授予权限。

* 已经实现的投票选民

  RoleVoter（角色选民），AuthenticatedVoter（认证选民），Jsr250Voter（jsr250选民 比如 @Resource）可以理解为java注解） 等等。

* 每个选民有三种投票方式

  ACCESS_DENIED（拒绝 -1 ） ACCESS_GRANTED （授予 1 ）  ACCESS_ABSTAIN （弃权 0 ）

* 也实现了三种投票机制。

  AffirmativeBased :

  默认的，如果一个或多个投票选民，投出 ACCESS_GRANTED ，也就是授予权限，那就表明，访问的这个内容，是有权限的。不论是否有投票选民投  ACCESS_DENIED 拒绝选票。如果所有人弃权，那么就走默认策略（isAllowIfAllAbstainDecisions），拒绝。

  ConsensusBased：

  如果投票选民，投出的授予选票大于拒绝选票，那就认为有权限访问。 如果授予选票等于拒绝选票，走默认策略，授予。如果授予选票小于拒绝选票，拒绝。如果所有投票选民弃权，走默认策略，拒绝。

  UnanimousBased ：如果一个或多个投票选民，投出 ACCESS_GRANTED ，也就是授予权限，那就表明，访问的这个内容，是有权限的。但是如果有一个投票选民，投出拒绝，直接拒绝访问该内容。如果所有人弃权，那么就走默认策略，拒绝。

 

为什么要提供一个投票决策管理器，来实现检测是否有权限？

因为对于内容的访问权限，配置的方式有多种，比如，角色（hasRole），权限（hasAuthority），注解形式 配置。那么对于同一个内容的访问权限，就需要大家一起投票，比如角色选民（RoleVoter）认为这个用户有权限，认证选民（AuthenticatedVoter） 认为这个用户没有权限，如果使用 AffirmativeBased  投票机制，那么这个用户就有权限，如果使用 UnanimousBased  投票机制，那么这个用户就没有权限。

### 综上所述

使用  spring security  实现认证鉴权，提供了默认实现和接口，功能强大。当然，这只是比较重要的一部分内容，目的对整个流程有一个简单的认识。

 

 

 

 

 

 

 

 