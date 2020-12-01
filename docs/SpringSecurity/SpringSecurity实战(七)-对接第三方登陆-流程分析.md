[TOC]

## 目标

* 了解第三方登陆流程

  参考：[如何在Markdown中画流程图](https://www.jianshu.com/p/b421cc723da5)
  [关于第三方登录，你应该知道的](http://www.woshipm.com/pd/437357.html)

  Spring Security 实战 书籍

  [JustAuth与用户系统整合](https://justauth.wiki/#/ext/justauth_integrated_with_the_existing_account_system)

本文分三篇来介绍第三方登陆的实现。

## 第三方登陆流程

许多人应该对第三方登陆不陌生，当你登陆某网站，会发现在登陆选项中，允许使用其他平台账户登陆。比如微信，QQ，微博等方式。

### 登陆流程

以Gitee 这个网站的第三方登陆示例：我们希望通过Github账户，登陆Gitee网站。

<div align="center"> 
    	<img src="https://s3.ax1x.com/2020/12/01/D4FySx.png" height="295" width="225" />
        <img src="https://s3.ax1x.com/2020/12/01/D4Fc6K.png" height="295" width="225" />
        <img src="https://s3.ax1x.com/2020/12/01/D4F6l6.png" height="295" width="225" />
</div>



大致分为三步：

1，点击github登陆，跳转到Github授权页面（如果已经绑定了该Github账户，会直接登陆成功，跳转到gitee的主页）

2，点击授权按钮，允许Gitee获取当前用户的信息。

3，跳转到Gitee的登陆绑定页面，绑定账户，实现登陆。（当然也可以不绑定本地账户，直接登陆成功。看业务定制需求）

### 必要概念了解

参考 ：[开箱即用的整合第三方登录的开源组件](https://justauth.wiki/#/) 建议将必读部分阅读完成 或者阅读 Spring Secuirty 第三部分内容

建议先阅读上述内容。

第三方登陆的实现基于OAuth2 协议来实现：

* OAuth 开发授权（Open Authorization）

  OAuth 解决了在用户不提供密码给第三方应用的情况下，让第三方应用有权获取用户数据以及基本信息的难题。OAuth 分为OAuth 1.0 和 OAuth 2.0 ，OAuth1.0已经基本退出历史舞台，所以了解OAuth 2.0即可。

  OAuth2 的授权流程大致分为四种角色：

  - 资源所有者（Resource Owner）：通常指用户自己，例如每一个Github用户

  - 客户端（Client）：指需要获取用户资源的网站，上述的 Gitee 网站就是客户端。
  - 资源服务器（Resource  Server）：指存储了用户资源的服务器，上述的Github网站就存储了你的用户信息
  - 认证服务器（Authorization Server）: 验证资源所有者（用户信息），并在验证成功后发放相关访问令牌（Access Token）给客户端。
  
  资源服务器和认证服务器可以是同一个。
  
  
  
  OAuth2 提供了四种授权机制：
  
  1. Authorization Code 授权码模式
  
     授权码模式是功能最完整、流程最严密的授权模式，它将用户引导到授权服务器进行身份验证，
     授权服务器将发放的访问令牌传递给客户端。
  
  2. 隐式授权模式（Implicit）
  
     结合移动应用或 Web App 使用
  
  3. 密码授权模式（Password Credentials）
  
     适用于受信任客户端应用，例如同个组织的内部或外部应用
  
  4. 客户端授权模式（Client Credentials）
  
     适用于客户端调用主服务API型应用（比如百度API Store）



#### 授权码模式(Authorization Code)

着重介绍一下这个模式流程，接下来的代码实现，都基于这种模式：(下图不是原图，修改了部分)

```markdown
 	+----------+
     | Resource |
     |   Owner  |
     |   用户   |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     | 用户代理  |                                 |   认证服务器   |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |         |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     |  Client |       (w/ Optional Refresh Token)
     |  客户端  |
     |         |                                    +---------------+
     |         |----(F)------ Access Token -------->|    Resource   |
     |         |                                    |     Server    |
     |         |<---(G)-- Protected Resource -------|   资源服务器   |
     +---------+                                    +---------------+
    
          
     
```

参见：[授权码](https://tools.ietf.org/html/rfc6749#section-1.2)

用户代理（User-Agent），可以理解为web浏览器。这里依然以 Gitee网站使用Github账户登录举例。

[Github 授权OAuth应用 文档](https://docs.github.com/en/free-pro-team@latest/developers/apps/authorizing-oauth-apps)

 A：用户发起第三方登陆请求，客户端通过浏览器请求认证服务器地址，并携带了客户端id（Client Identifier）和回调地址（Redirection URI）参数，认证服务器在浏览器端显示，授权页面。（图示第二个图）
 Github 的请求用户身份地址,携带了client_id和redirect_uri的参数：

```
 GET https://github.com/login/oauth/authoriz?client_id=e1015c0a10dc5e11b718&redirect_uri=http://localhost:8081/api/v1/oauth/callback/github
```

B：用户选择是否授予访问权限

C：点页面上的同意，认证服务器会回调客户端Redirection URI这个地址，并携带code 参数返回

Github 回调客户端地址：

```
 GET http://localhost:8081/api/v1/oauth/callback/github?code=28ba84fe6fbbb895f1e6&state=a885aad30100738a7d26018f9ec52c17
```

D： 客户端获得认证服务器返回的code 参数，然后再发送获取令牌请求到认证服务器，并携带code参数

Github 获取token的请求：需要三个参数

```
POST https://github.com/login/oauth/access_token

client_id	string	需要。您从GitHub收到的GitHub App的客户端ID。
client_secret	string	需要。您从GitHub收到的GitHub App的客户密码。
code	string	需要。您收到的作为对步骤1的响应的代码。
```

E:  认证服务器根据code等参数，返回 access_token

F：客户端拿着 access_token 去请求资源服务器

Github 获取用户信息请求：

```
Authorization: token OAUTH-TOKEN
GET https://api.github.com/user
```

G：资源服务器返回受保护的资源，也就是目标用户信息

 参数解释：

```
clientId 客户端身份标识符（应用id），一般在申请完Oauth应用后，由第三方平台颁发，唯一
clientSecret 客户端密钥，一般在申请完Oauth应用后，由第三方平台颁发
redirectUri 开发者项目中的有效api地址。用户在确认第三方平台授权（登录）后，第三方平台会重定向到该地址，并携带code等参数
```



## 实现第三方登陆框架

* Spring  Social

  Spring  Social是一个专门用于连接社交平台，实现OAuth服务共享的框架。

  github地址：https://github.com/spring-projects/spring-social

  **被spring官方抛弃，不建议使用，已经很长时间没有更新了。**

  [spring官方为什么放弃spring social项目及替代方案](https://www.cnblogs.com/gzulmc/p/12692939.html)

* JustAuth

  JustAuth，如你所见，它仅仅是一个**第三方授权登录**的**工具类库**，它可以让我们脱离繁琐的第三方登录 SDK，让登录变得**So easy!**

  github地址：https://github.com/justauth/JustAuth

  国人开发，不断更新中，不依赖于Spring 框架，支持国内大多数平台的登陆，使用还是很简单的，如果不对接Spring Security框架**推荐使用**。

* Spring Security OAuth2

  默认提供了对Goole，Github，Facebook，Okta 这些平台的第三方登陆支持，如果需要定制其他第三方登陆增加代码实现即可。

  github地址：https://github.com/spring-projects/spring-security

  提供了Oauth2的解决方案，如果用于对接Spring Security **建议使用**。

  文档：[spring -security官方Oauth2文档示例](https://github.com/spring-projects/spring-security/tree/5.4.1/samples/boot/oauth2login#github-login)

