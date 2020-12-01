[TOC]

## 目标

* 了解自定义认证方式集成图片验证码

  参考：Spring Security 实战 书籍

  [使用redis时遇到的一些异常](https://blog.csdn.net/xyc_csdn/article/details/72841389)

  [尽管创建了自己的实现，但仍创建了额外的DaoAuthenticationProvider](https://github.com/spring-projects/spring-security/issues/5364)

 

## 自定义认证配置

业务需求：用户进入登录页面，输入用户名，密码，验证码，进行登录。要求校验用户输入的验证码是否正确，不正确拒绝用户登陆。

思路： 依赖于[上篇](https://gengzi.github.io/Code-animal/#/./docs/SpringSecurity/SpringSecurity%E5%AE%9E%E6%88%98(%E4%BA%94)-%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)介绍的Spring Security 的认证流程，得知对于表单登陆，DaoAuthenticationProvider 是最终实现认证过程的Provider。我们期望将验证码的验证也加入其中，可以新建一个AuthenticationProvider继承DaoAuthenticationProvider 重写认证方法，用来实现集成图片验证码的功能。那么关于图片验证码的参数，就需要传递到这个自定义的Provider中，从上篇，依然可以得知 UsernamePasswordAuthenticationToken 可以在支持这个token类型的provider 中流转，从而获取参数。而且为了提供额外的参数支持，可以将额外参数设置到 details 参数中。为了方便设置这个details参数数据，还提供了 AuthenticationDetailsSource 和 WebAuthenticationDetails 供设置参数使用，我们实现接口加入设置图片验证码的参数即可。

### 代码实践

项目环境：spring boot 2.2.7 Jpa java8 mysql5.7

代码参考：https://github.com/gengzi/gengzi_spring_security 请切换到 captcha 分支

基本依赖，参考前面几篇文章，不在阐述。

由于前几篇已经对接了spring session 和 redisson 框架，就是要实现前后端分离的形式，也就不再依赖cookie 和 session 。所以下面的实现，就基于redis 来实现了。当然也会简单说下，使用cookie 和session 的实现方法。

### 生成图片验证码部分

可以参考：[SpringSecurity实战(四)-集成图片验证码-过滤器方式实现](https://gengzi.github.io/Code-animal/#/./docs/SpringSecurity/SpringSecurity%E5%AE%9E%E6%88%98(%E5%9B%9B)-%E9%9B%86%E6%88%90%E5%9B%BE%E7%89%87%E9%AA%8C%E8%AF%81%E7%A0%81-%E8%BF%87%E6%BB%A4%E5%99%A8%E6%96%B9%E5%BC%8F%E5%AE%9E%E7%8E%B0)

不再过多阐述。

### 验证“图片验证码”部分

依照上面的思路：我们需要自定义 AuthenticationProvider ， AuthenticationDetailsSource  和WebAuthenticationDetails  。

WebAuthenticationDetails 作用：该类实现 Serializable 接口，主要用于设置 remoteAddress  和 sessionId 两个参数，供AuthenticationProvider 获取使用。

#### 设置"图片验证码"参数

用户登陆请求会携带四个参数：

```shell
uuid: 434e1fba-edf9-4c58-87c0-bccc5b162fff   # redis存储验证码的key
username: admin   #用户名
password: 111      #密码
validCode: tk2zdf #输入的验证码
```

CaptchaWebAuthenticationDetails

代码参考：[**CaptchaWebAuthenticationDetails**](https://github.com/gengzi/gengzi_spring_security/blob/captcha/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/detail/CaptchaWebAuthenticationDetails.java)

根据用户输入的验证码信息，从redis 中获取正确的验证码值，进行对比。成功设置flag 参数为true，失败，false。

```java
 
/**
* <h1>验证码Web身份验证详细信息</h1>
* <p>
* 扩展WebAuthenticationDetails 的参数，增加 flag 参数。
* <p>
* flag 参数用来判断 验证码是否正确的标识 true 正确，false 不正确
*
* @author gengzi
* @date 2020年11月27日15:34:38
*/
public class CaptchaWebAuthenticationDetails extends WebAuthenticationDetails {
 
    private RedisUtil redisUtil;
 
    // 验证码是否正确
    boolean flag = false;
 
    public CaptchaWebAuthenticationDetails(HttpServletRequest request, RedisUtil redisUtil) {
        super(request);
        this.setRedisUtil(redisUtil);
        validate(request);
        // 这里设置了 redisUtil 会导致序列化 springsecuritycontext 时，包含 redisUtil 这个类，这个类不能被序列化，所以用完就干掉
        this.setRedisUtil(null);
 
    }
 
    private void validate(HttpServletRequest request) {
        String uuid = request.getParameter("uuid");
        String validCode = request.getParameter("validCode");
        // 校验一下随机验证码
        String validCodeByRedis = (String) redisUtil.get(String.format(RedisKeyContants.VALIDCODEKEY, uuid));
        if (validCode.equals(validCodeByRedis)) {
            flag = true;
            redisUtil.del(String.format(RedisKeyContants.VALIDCODEKEY, uuid));
        }
    }
 
    public RedisUtil getRedisUtil() {
        return redisUtil;
    }
 
    public void setRedisUtil(RedisUtil redisUtil) {
        this.redisUtil = redisUtil;
    }
 
    public boolean isFlag() {
        return flag;
    }
 
    public void setFlag(boolean flag) {
        this.flag = flag;
    }
}
```

特别注意：此类用到了redis，redis也会被当做一个参数进入到 provider 中，当认证完成，去序列化springsecuritycontext会出现一个报错，序列化失败（java.io.NotSerializableException: xxx）xxx代表就是序列化的类。所以不用了，就把它设置为null吧。

CaptchaWebAuthenticationDetailsSource

代码参考：[CaptchaWebAuthenticationDetailsSource.java](https://github.com/gengzi/gengzi_spring_security/blob/captcha/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/detail/CaptchaWebAuthenticationDetailsSource.java)

```java
/**
* <H1>验证码Web身份验证详细信息源</H1>
* <p>
 * 用于替换 UsernamePasswordAuthenticationFilter 中默认的  AuthenticationDetailsSource 属性
* 此类需要在核心配置中配置
* 代码示例：
* <p>
* protected void configure(HttpSecurity http) throws Exception {
 *    http...
 *        .formLogin()
 *        .authenticationDetailsSource(captchaWebAuthenticationDetailsSource)  // 替换原有的authenticationDetailsSource
 *        ....
 *        ;
* }
*
* @author gengzi
* @date 2020年11月27日15:43:06
*/
@Component
public class CaptchaWebAuthenticationDetailsSource implements AuthenticationDetailsSource<HttpServletRequest, WebAuthenticationDetails> {
 
    @Autowired
    private RedisUtil redisUtil;
 
    @Override
    public WebAuthenticationDetails buildDetails(HttpServletRequest context) {
        CaptchaWebAuthenticationDetails captchaWebAuthenticationDetails = new CaptchaWebAuthenticationDetails(context, redisUtil);
        return captchaWebAuthenticationDetails;
    }
}
```

#### 自定义AuthenticationProvider

代码参考：[CaptchaProvider.java ](https://github.com/gengzi/gengzi_spring_security/blob/captcha/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/provider/CaptchaProvider.java)

替换原有的DaoAuthenticationProvider 实现

```java
 
/**
* <h1>验证码认证提供者</h1>
* <p>
* 重写 additionalAuthenticationChecks 方法，增加图片验证码的校验判断
* 成功，执行父类校验，走原有流程
* 失败，抛出异常，告知用户验证码错误
*
* 该类需要在核心配置中配置，以便于替换原有的 AuthenticationProvider
* 代码示例：
 *       @Autowired
 *       private AuthenticationProvider authenticationProvider;
 *          protected void configure(AuthenticationManagerBuilder auth) throws Exception {
 *         // 设置 userDetailsService 和  authenticationProvider 都会创建一个 Provider。 如果仅需要一个，请只设置一个
 *         auth.authenticationProvider(authenticationProvider);
 *     }
*
*
*
* @author gengzi
* @date 2020年11月26日13:51:32
*/
@Slf4j
@Component
public class CaptchaProvider extends DaoAuthenticationProvider {
 
    /**
     * 使用构造方法，将 userDetailsService 和 encoder 注入
     *
     * @param userDetailsService 用户详细服务
     * @param encoder            密码加密方式
     */
    public CaptchaProvider(UserDetailsService userDetailsService, PasswordEncoder encoder) {
        this.setUserDetailsService(userDetailsService);
        this.setPasswordEncoder(encoder);
    }
 
    /**
     * 验证码验证是否正确
     *
     * @param userDetails
     * @param authentication
     * @throws AuthenticationException
     */
    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        // 获取验证码是否通过的标识
        CaptchaWebAuthenticationDetails details = (CaptchaWebAuthenticationDetails) authentication.getDetails();
        boolean flag = details.isFlag();
        if (!flag) {
            // 不成功 ，抛出异常
            throw new CaptchaErrorException(RspCodeEnum.ERROR_VALIDCODE.getDesc());
        }
        // 成功，调用父类验证
        super.additionalAuthenticationChecks(userDetails, authentication);
    }
}
 
```

这里还提供了一个 CaptchaErrorException ，主要用于返回 认证异常，这样可以走认证失败的事件。

代码参考：[CaptchaErrorException.java ](https://github.com/gengzi/gengzi_spring_security/blob/captcha/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/exception/CaptchaErrorException.java)

```java
/**
* <h1>验证码错误异常</h1>
*
* @author gengzi
* @date 2020年11月26日13:49:19
*/
public class CaptchaErrorException extends AuthenticationException {
    public CaptchaErrorException(String msg) {
        super("验证码输入错误");
    }
}
```

 

#### 核心配置

代码参考：[WebSecurityConfig.java ](https://github.com/gengzi/gengzi_spring_security/blob/captcha/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/config/WebSecurityConfig.java)

太长了，只粘重要部分。

```java
 
 
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private CaptchaWebAuthenticationDetailsSource captchaWebAuthenticationDetailsSource;
    @Autowired
    private AuthenticationProvider authenticationProvider;
 
   /**
     * 认证管理器配置方法
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
//        auth.userDetailsService(userDetailsService);
//        // 增加认证提供者
        // 设置 userDetailsService 和  authenticationProvider 都会创建一个 Provider。 如果仅需要一个，请只设置一个
        auth.authenticationProvider(authenticationProvider);
    }
 
 
        @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 自定义表单认证方式
        http.apply(otherSysOauth2LoginAuthenticationSecurityConfig).and()
                .authorizeRequests()
                // 放行swagger-ui相关的路径
                .antMatchers(IgnoringUrlConstant.IGNORING_URLS).permitAll()
                .antMatchers(IgnoringUrlConstant.IGNORING_STATIC_URLS).permitAll()
                .antMatchers(IgnoringUrlConstant.OAUTH2_URLS).permitAll()
                .antMatchers("/getLoginCode").permitAll()
                .antMatchers("/codeBuildNew/**").permitAll()  // 都可以访问
                .anyRequest().authenticated().and().formLogin().loginPage("/login.html").loginProcessingUrl("/login").permitAll().and()
                .csrf().disable()// csrf 防止跨站脚本攻击
                .formLogin()
 
  // ----------------------------- start -------------------------------
                .authenticationDetailsSource(captchaWebAuthenticationDetailsSource)  // 替换原有的authenticationDetailsSource
  // ---------------------------- end  -----------------------------------
                .successHandler(userAuthenticationSuccessHandler)
                .failureHandler(userAuthenticationFailureHandler).and()
                .sessionManagement((sessionManagement) -> sessionManagement
                        .maximumSessions(100)
                        .sessionRegistry(sessionRegistry()));
    }
 
}
 
 
```

 

### 注意

在配置 authenticationProvider 时， 出现了一个错误。

我进行了如下的配置：

```java
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
       // 设置了 userDetailsService
        auth.userDetailsService(userDetailsService);
        // 设置了认证提供
        auth.authenticationProvider(authenticationProvider);
    }
```

我的预期是执行认证过程，CaptchaProvider 会替换原有的DaoAuthenticationProvider 认证，也就是仅有一个认证Provider。但是出现了两个 Provider，导致用户验证码输入错误，CaptchaProvider 认证不通过，但是DaoAuthenticationProvider  认证就通过了，因为它不会校验验证码是否正确。导致用户在输入错误验证码的时候，依然可以正确登陆。查阅github 上的问题后，得知设置 userDetailsService 创建一个 Provider，再设置一个 authenticationProvider 会再创建一个 Provider。

所以，解决办法很简单，删除掉userDetailsService 的配置，因为不在这里配置userDetailsService ，我也在 Provider 中进行了注入。

问题原因： [尽管创建了自己的实现，但仍创建了额外的DaoAuthenticationProvider](https://github.com/spring-projects/spring-security/issues/5364)

 

 

 

 

 
