[TOC]

## 目标

* 了解Spring Security的认证流程

  参考：Spring Security 实战书籍

  [SpringSecurity使用JustAuth扩展第三方登录](https://zhuanlan.zhihu.com/p/144670329) 推荐阅读

  [尽管创建了自己的实现，但仍创建了额外的DaoAuthenticationProvider ](https://github.com/spring-projects/spring-security/issues/5364)

  [Spring boot + Spring Security 多种登录认证方式配置（一）](https://blog.csdn.net/qq_36521507/article/details/103365805)

  [使用redis时遇到的一些异常](https://blog.csdn.net/xyc_csdn/article/details/72841389)


在第一篇已经简单说明了，认证授权的简单流程和概念，本文详细介绍一下，在认证流程中，具体执行了什么那些类。

## 认证流程

简单流程：

用户发起登陆请求 ---》到Security过滤器链 --- 》 根据请求路径（/login）匹配对应的认证过滤器 --- 》执行预验证处理方法，从请求中获取用户名（username）和密码（password），使用传入的信息构造 Token认证令牌---》 Spring Security 执行 providermanager 循环所有的认证provider ， 每个provider执行真正的验证流程，根据Token认证令牌，检索用户信息，执行认证逻辑。 如果认证成功，重新构造Token 认证令牌（认证成功的并授权）返回 ，并执行认证成功事件。如果认证失败，执行认证失败事件- 》认证授权结束

这个是Spring Security 提供基础认证流程，涉及到比较关键的类。

![DBUXy6.png](https://s3.ax1x.com/2020/11/26/DBUXy6.png)



解释下各个类的作用和关键方法

* AbstractAuthenticationProcessingFilter 

  基于浏览器的基于HTTP的身份验证请求的抽象处理器。建议看看这个类提供的注释。描述了认证流程。

  如果请求与setRequiresAuthenticationRequestMatcher(RequestMatcher)匹配，则此过滤器将拦截请求并尝试从该请求执行身份验证。

  认证由attemptAuthentication()方法执行，该方法必须由子类实现。

  

* UsernamePasswordAuthenticationFilter

  是AbstractAuthenticationProcessingFilter  的子类

  登陆表单认证过滤器，这也是大多数系统支持的登陆方式，表单认证。

  拦截路径 /login Post请求，必传参数 username  password

  核心方法：attemptAuthentication()  执行预认证，这个方法也是我们实现该Filter 必须要实现的方法。

  此方法一般做简单校验，和输入参数的获取，构造 UsernamePasswordAuthenticationToken  用户密码认证令牌。

  调用 ProviderManager 来循环执行认证

  重要代码展示：

  ```java
  	// 在构造方法就设置了拦截的请求路径 和 请求方式
      public UsernamePasswordAuthenticationFilter() {
  		super(new AntPathRequestMatcher("/login", "POST"));
  	}
  
  	// 尝试认证
  	public Authentication attemptAuthentication(HttpServletRequest request,
  			HttpServletResponse response) throws AuthenticationException {
          // 请求方式的校验
  		if (postOnly && !request.getMethod().equals("POST")) {
  			throw new AuthenticationServiceException(
  					"Authentication method not supported: " + request.getMethod());
  		}
  
          // 从request 中获取用户名 和密码
  		String username = obtainUsername(request);
  		String password = obtainPassword(request);
  
  		if (username == null) {
  			username = "";
  		}
  
  		if (password == null) {
  			password = "";
  		}
  
  		username = username.trim();
  		// 封装 token ，设置用户名和密码
  		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
  				username, password);
          
  		// 允许子类设置额外的信息到 token 中，方便在 认证（provider）流程中使用
  		// Allow subclasses to set the "details" property
  		setDetails(request, authRequest);
          
  		// 传递token 调用认证流程，进行认证
  		return this.getAuthenticationManager().authenticate(authRequest);
  	}
  
  ```

  

* UsernamePasswordAuthenticationToken  

  用户名密码认证token， 简单的理解，就是对认证参数的封装，用于在认证提供者（Provider）中流转使用.其中有个参数需要注意 details ，该参数能够携带账户之外的信息。 比如 sessionid 等等其他参数信息

  重要代码展示：

  ```java
  // 参照上面的，就是把 用户名 和密码 传入，创建 token	
  public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
  		super(null);
  		this.principal = principal;   // 用户名
  		this.credentials = credentials;  // 密码
  		setAuthenticated(false);
  	}
  
  	// 设置额外信息，这个方法是  UsernamePasswordAuthenticationFilter 执行的
  	protected void setDetails(HttpServletRequest request,
  			UsernamePasswordAuthenticationToken authRequest) {
  		authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
  	}
  	// 这里引入了一个  authenticationDetailsSource 认证详细信息源
  	// 干啥的，就是来构造 Details 的。下面代码是其中一个实现
  	
  	public WebAuthenticationDetails buildDetails(HttpServletRequest context) {
  		return new WebAuthenticationDetails(context);
  	}
  	// 下面的构造方法中，获取了 sessionid   remoteAddress 两个数据，封装成一个  WebAuthenticationDetails。 这样在 UsernamePasswordAuthenticationToken 中，就可以getDetails 获取到WebAuthenticationDetails 这个实体，那么也就能获取到这两个设置的参数了
  	public WebAuthenticationDetails(HttpServletRequest request) {
  		this.remoteAddress = request.getRemoteAddr();
  
  		HttpSession session = request.getSession(false);
  		this.sessionId = (session != null) ? session.getId() : null;
  	}
  
  
  ```




* ProviderManager

  认证管理器，在一个系统中会出现多个认证管理器，此类的作用就是循环所有的认证Provider，只要有一个认证Provider验证通过。那么就认为认证成功，也就是用户登陆成功。

  举例一些认证提供者（provider）：DaoAuthenticationProvider （Dao认证），RememberMeAuthenticationProvider（记住我）等

  核心代码展示：
  

  ```java
  public Authentication authenticate(Authentication authentication)
  			throws AuthenticationException {
  		Class<? extends Authentication> toTest = authentication.getClass();
  		AuthenticationException lastException = null;
  		AuthenticationException parentException = null;
  		Authentication result = null;
  		Authentication parentResult = null;
  		boolean debug = logger.isDebugEnabled();
  		// 循环所有的 provider
  		for (AuthenticationProvider provider : getProviders()) {
              // 判断当前 provider 是否支持 认证令牌的类型
  			if (!provider.supports(toTest)) {
                  // 不支持，循环下一个
  				continue;
  			}
  			// 执行真正的认证逻辑，传入参数 token
  			result = provider.authenticate(authentication);
          }
      
      
      // 判断当前认证是否支持  对应的Token，比如  UsernamePasswordAuthenticationFilter
      public boolean supports(Class<?> authentication) {
  		return (xxxxToken.class
  				.isAssignableFrom(authentication));
  	}
  ```

  

  **对于认证流程来说，只要有一个Provider认证成功，就是登陆成功。不管其他Provider有没有认证通过**

* AbstractUserDetailsAuthenticationProvider

  抽象用户详细信息身份验证提供程序，该类就是支持  UsernamePasswordAuthenticationToken 这个认证令牌

  重要方法：authenticate() 执行认证逻辑方法， retrieveUser() 根据用户名查询系统用户信息方法。

  简单看下认证逻辑

  ```java
  public Authentication authenticate(Authentication authentication)
  			throws AuthenticationException {
      // 断言判断，token 是否是 UsernamePasswordAuthenticationToken
  		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
  				() -> messages.getMessage(
  						"AbstractUserDetailsAuthenticationProvider.onlySupports",
  						"Only UsernamePasswordAuthenticationToken is supported"));
  
  		// 确定一下用户名
  		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
  				: authentication.getName();
  	
  		// 查询用户详情 UserDetails
  	    user = retrieveUser(username,
  						(UsernamePasswordAuthenticationToken) authentication);
  
  
  		// 预先检查，就是判断一下用户是否 可用，是否过期等等
  		preAuthenticationChecks.check(user);
      	// 其他身份检查，这个是个子类实现的方法。也就是说，如果你还想检查别的，就可以实现 AbstractUserDetailsAuthenticationProvider 重写该方法，提供对其他数据的判断。比如 
      // DaoAuthenticationProvider 实现了对用户密码的校验
      
  		additionalAuthenticationChecks(user,
  					(UsernamePasswordAuthenticationToken) authentication);
  	
  		// 发布身份检查
  		postAuthenticationChecks.check(user);
  		// 创建成功认证
  		return createSuccessAuthentication(principalToReturn, authentication, user);
  	}
  
  	// 创建成功认证
  	protected Authentication createSuccessAuthentication(Object principal,
  			Authentication authentication, UserDetails user) {
          //确保我们返回用户提供的原始凭据，即使使用编码的密码，后续尝试也会成功。 
          //还请确保我们返回原始的getDetails（），以便缓存过期后将来的身份验证事件包含详细信息
  	
          // 设置 用户名 密码 权限 ，并设置 Details
  		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
  				principal, authentication.getCredentials(),
  				authoritiesMapper.mapAuthorities(user.getAuthorities()));
  		result.setDetails(authentication.getDetails());
  
  		return result;
  	}
  ```

  

  

* DaoAuthenticationProvider

  实现了 AbstractUserDetailsAuthenticationProvider 提供了 retrieveUser 的实现

  

  ```java
  	
  protected final UserDetails retrieveUser(String username,
  			UsernamePasswordAuthenticationToken authentication)
  			throws AuthenticationException {
  		// 主要作用，UserDetailsService 根据用户名获取用户详情
  			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
  			return loadedUser;
  	}
  
  	protected void additionalAuthenticationChecks(UserDetails userDetails,
  			UsernamePasswordAuthenticationToken authentication)
  			throws AuthenticationException {
  	
  		// 获取用户输入的密码
  		String presentedPassword = authentication.getCredentials().toString();
  		// 使用 密码编辑器，对输入密码 和 系统中存储的用户密码进行匹配
  		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
  			logger.debug("Authentication failed: password does not match stored value");
  			// 匹配不成功，抛出 BadCredentialsException 异常
  			throw new BadCredentialsException(messages.getMessage(
  					"AbstractUserDetailsAuthenticationProvider.badCredentials",
  					"Bad credentials"));
  		}
  	}
  ```

* UserDetailsService

  用户详细服务，根据用户名查询系统中的用户详情（UserDetails）

  不再阐述，前几篇已经说明过了。 

* FailureHandler/SuccessHandler

  认证成功走认证成功的事件，认证失败走认证失败的事件。

  ```java
  public class WebSecurityConfig extends WebSecurityConfigurerAdapter {  
  
     protected void configure(HttpSecurity http) throws Exception {
    	http ...
    	.successHandler(userAuthenticationSuccessHandler)
    	.failureHandler(userAuthenticationFailureHandler)
     }
  }	
  ```



## 总结

上述的流程，对于我们编写自定义认证流程起到关键作用。我们只需重写相关的方法加以配置，就可以实现一套自定义的认证流程。那么对于登陆图片验证码的实现，就可以考虑使用上述的方法和类。 







