[TOC]

## 目标

* 了解Spring Security 集成图片验证码

  参考：spring security 实战书籍

## 实现的方式

验证码是区分人与机器的有效方式，几乎所有系统的登陆，都要求输入验证码。

Spring Security 集成图片验证码，有两种方式：

1. 使用Filter过滤器(简单方式)

2. 使用Spring Security 提供的认证配置的方式

分三篇介绍这两种方法：

## 自定义验证码过滤器（简单方式）

业务需求：用户进入登录页面，输入用户名，密码，验证码，进行登录。要求校验用户输入的验证码是否正确，不正确拒绝用户登陆。

思路： 在不使用Spring Security 时，基本做法就是，用户访问登陆页面，页面加载图片验证码，后端生成验证码图片返回前端，并在session 中存储验证码的key 和 value。当用户登陆，后端根据cookie中的sessionid 匹配对应的session ，从session中获取验证码的值，与用户输入的验证码匹配，相等，就认为验证码正确，继续进行下来的用户登陆验证。不相等，就认为验证码输入有误，拒绝登陆。通常也是创建一个过滤器来实现，那么使用Spring Security 后，已知Spring Security 提供了一个过滤器链，进行认证和授权操作。那么自定义一个验证码过滤器，将其加入到这个过滤器链中，当匹配到用户登陆时，立刻对验证码进行校验。成功，就继续执行用户认证操作，不成功，拒绝登陆。并且需要将这个过滤器加入到UsernamePasswordAuthenticationFilter 前面。

### 代码实践

项目环境：spring boot 2.2.7 Jpa java8 mysql5.7

代码参考：https://github.com/gengzi/gengzi_spring_security

基本依赖，参考前面几篇文章，不在阐述。

由于上篇已经对接了spring session  和 redisson 框架，就是要实现前后端分离的形式，也就不再依赖cookie 和 session 。所以下面的实现，就基于redis 来实现了。当然也会简单说下，使用cookie 和session 的实现方法。

 

###  生成图片验证码部分

依赖：pom.xml

我是用的 hutool 提供的验证码实现，也可以使用  EasyCaptcha（效果不错） 等等。

```xml
        <!--Hutool是一个小而全的Java工具类库-->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.3.6</version>
        </dependency>
```

 

#### 页面

[**login.html**](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/resources/templates/login.html)  直接参考，太长了不粘了

![DU9iFO.png](https://s3.ax1x.com/2020/11/24/DU9iFO.png)

#### controller 层

代码参考：[**CaptchaController.java**](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/sys/controller/CaptchaController.java)

前后端分离方式（不依赖cookie）：前端获取验证码会传入一个 code ，也就是一个uuid随机码，code 作为Redis 的key，生成的验证码值，作为redis 的value。登陆时，将code  和 验证码值，都传入。从redis 中获取对应的验证码。

```java
    @ApiOperation(value = "获取登陆随机验证码", notes = "获取登陆随机验证码")
    @ApiImplicitParams({@ApiImplicitParam(name = "code", value = "code", required = true)})
    @GetMapping("/getLoginCode")
    @ResponseBody
    public String getLoginCode(@RequestParam("code") String code, HttpServletResponse response) {
        ReturnData ret = ReturnData.newInstance();
        if (StringUtils.isBlank(code)) {
            return "error";
        }
        //定义图形验证码的长、宽、验证码字符数、干扰线宽度
        ShearCaptcha captcha = CaptchaUtil.createShearCaptcha(200, 100, 4, 4);
        String verificationCode = captcha.getCode();
        log.info("verificationCode:{}", verificationCode);
        // 存入redis中
        redisUtil.set(String.format(RedisKeyContants.VALIDCODEKEY, code), verificationCode, 180);
        //图形验证码写出，可以写出到文件，也可以写出到流
        return "data:image/png;base64," + captcha.getImageBase64();
    }
```

细节：

这里对存入的验证码，设置了超时时间为3分钟。防止这个验证码长时间不验证，还存储在redis 中。

返回的图形验证码，这里是返回的 base64 编码的图片信息，也可以考虑返回文件或者流。

 

依赖cookie 的实现：

```java
// 伪代码
 
public void getLoginCode(HttpServletRequest request , HttpServletResponse response) {
     // 生成验证码图片和文本
     Image img = xxx;
     String captchaText = xxx;
     // 设置到 session 中
     request.getSession().setAttribute("captcha",captchaText);
     // 设置响应内容
     response.setContentType("image/jpeg");
     // 获取响应流
     outPutStream = response.getOutputStream();
     // 将图片写入流中
}
 
```

 

### 验证“图片验证码”部分

#### 图片过滤器

代码参考：[ValidateCodeFilter.java](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/filter/ValidateCodeFilter.java)

校验是否为登陆请求（Post   /login），是，从redis 中获取验证码对比，成功，删除redis存储的验证码，继续进行用户认证。失败，响应验证码错误。

```java
@AllArgsConstructor
@Component
public class ValidateCodeFilter extends OncePerRequestFilter {
    private final static String OAUTH_TOKEN_URL = "/login";
    private UserAuthenticationFailureHandler authenticationFailureHandler;
    @Autowired
    private RedisUtil redisUtil;
 
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        if (request.getServletPath().equals(OAUTH_TOKEN_URL)
                && request.getMethod().equalsIgnoreCase("POST")) {
            try {
                //校验验证码
                validate(request);
            } catch (AuthenticationException e) {
                //失败处理器
                authenticationFailureHandler.onAuthenticationFailure(request, response, e);
                return;
            }
        }
 
        filterChain.doFilter(request, response);
    }
 
    private void validate(HttpServletRequest request) {
        String uuid = request.getParameter("uuid");
        String validCode = request.getParameter("validCode");
 
        // 校验一下随机验证码
        String validCodeByRedis = (String) redisUtil.get(String.format(RedisKeyContants.VALIDCODEKEY, uuid));
        boolean flag = false;
        if (validCode.equals(validCodeByRedis)) {
            flag = true;
            redisUtil.del(String.format(RedisKeyContants.VALIDCODEKEY, uuid));
        }
 
        if (!flag) {
            throw new RrException(RspCodeEnum.ERROR_VALIDCODE.getDesc(), RspCodeEnum.ERROR_VALIDCODE.getCode());
        }
    }
}
 
```

#### 核心配置

代码参考：[WebSecurityConfig.java](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/config/WebSecurityConfig.java)

将验证码过滤器加入到 Spring Security 过滤器链中。

```java
// ------------ 验证码过滤器 -------------
@Autowired
private ValidateCodeFilter validateCodeFilter;
 
 
protected void configure(HttpSecurity http) throws Exception {
        // 自定义表单认证方式
        http...
            // ------  配置  ----------
                addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class)
            // -----  配置  ----------
                .authorizeRequests()
            ...
  }
```

 

## 注意

* 在spring 容器中，实现过滤器建议继承 OncePerRequestFilter，OncePerRequestFilter 保证每一个请求只会通过一次当前过滤器，Filter 并不能保证。

 

 

 

 