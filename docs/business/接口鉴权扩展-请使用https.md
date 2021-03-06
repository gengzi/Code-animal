 


[TOC]

## 目标

* 了解https协议概念

* 实践配置https协议-tomcat和nginx

  参考：[【网络安全】HTTPS为什么比较安全](https://www.cnblogs.com/54chensongxia/archive/2019/11/04/11772752.html)

  [数字签名是什么？](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)  推荐阅读

  [CA认证](https://baike.baidu.com/item/CA%E8%AE%A4%E8%AF%81/6471579?fr=aladdin)

该篇是理论篇

## HTTPS 相关概念

概念：[https百度百科](https://baike.baidu.com/item/https/285356?fr=aladdin) 建议阅读一遍

### SSL协议的主要功能

- 加密数据以防止数据中途被窃取；
- 认证用户和服务器，确保数据发送到正确的客户机和服务器；
- 验证数据的完整性，确保数据在传输过程中不被篡改。

### SSL 原理

先说下简单流程，ssl 的一个功能是加密数据以防止数据中途被窃取，那么加密数据用的对称加密算法，比如 AES ,DES 等等，已知对称加密算法只有一个密钥，一旦密钥被泄露，传输的数据也就可以被解密。我们也知道，非对称加密RSA公钥加密可以防止数据被泄露，所以客户端会使用服务端的公钥加密对称加密算法的密钥，发往服务端，服务端通过自己的私钥解密，就拿到了对称加密的密钥。

从大致的流程中，我们发现为了加密数据，使用了对称加密算法，为了保证对称加密的密钥不被泄露，使用了非对称加密算法，那么怎么保证服务端非对称加密的算法公钥，是真的服务端的而不是冒充的？

引入了受信任的第三方，CA机构。

## CA机构-数字签名和数字证书

先看一篇大佬的文章，理解一下 [数字签名是什么？](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)    非常推荐阅读

证书颁发机构（CA, Certificate Authority）即颁发数字证书的机构。是负责发放和管理数字证书的权威机构，并作为电子商务交易中受信任的第三方，承担公钥体系中公钥的合法性检验的责任。

CA中心为每个使用[公开密钥](https://baike.baidu.com/item/公开密钥)的用户发放一个数字证书，数字证书的作用是证明证书中列出的用户合法拥有证书中列出的公开密钥。CA机构的[数字签名](https://baike.baidu.com/item/数字签名)使得攻击者不能伪造和篡改证书。

证书实际是由证书签证机关（CA）签发的对用户的公钥的认证。

上面的介绍来源百度百科-  [CA认证]([https://baike.baidu.com/item/CA%E8%AE%A4%E8%AF%81/6471579?fr=aladdin](https://baike.baidu.com/item/CA认证/6471579?fr=aladdin))  建议阅读

简单的理解：CA机构也有一套RSA公钥私钥，用自己的私钥，对客户端或者服务端的公钥和一些相关信息一起加密，生成"数字证书"（Digital Certificate），在客户端（服务端）需要获取服务端（客户端）公钥时，使用CA机构的公钥解密，比对摘要信息，如果一致，就通过。不一致，说明**通信被劫持**了。

综上所述，可知使用Https 需要三方参与，客户端（发送请求方）、服务端（接受请求方）、CA机构（第三方受信任的证书颁发机构）。

## SSL单向认证和双向认证

参考：[https双向认证](https://www.wosign.com/tag/httpsshuangxiangrenzheng.htm)

[图解 https 单向认证和双向认证！](https://www.cnblogs.com/kabi/p/11629603.html)

### 单向认证

![d2M4dP.png](https://s1.ax1x.com/2020/08/25/d2M4dP.png)

1、客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。

2、服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务器端的证书，即公钥证书

3、客户端使用服务端返回的信息验证服务器的合法性，包括：

证书是否过期

发型服务器证书的CA是否可靠

返回的公钥是否能正确解开返回证书中的数字签名

服务器证书上的域名是否和服务器的实际域名相匹配

验证通过后，将继续进行通信，否则，终止通信

4、客户端向服务端发送自己所能支持的对称加密方案，供服务器端进行选择

5、服务器端在客户端提供的加密方案中选择加密程度最高的加密方式。

6、服务器将选择好的加密方案通过明文方式返回给客户端

7、客户端接收到服务端返回的加密方式后，使用该加密方式生成产生随机码，用作通信过程中对称加密的密钥，使用服务端返回的公钥进行加密，将加密后的随机码发送至服务器

8、服务器收到客户端返回的加密信息后，使用自己的私钥进行解密，获取对称加密密钥。 在接下来的会话中，服务器和客户端将会使用该密码进行对称加密，保证通信过程中信息的安全。

### 双向认证

双向认证和单向认证原理基本差不多，只是除了客户端需要认证服务端以外，增加了服务端对客户端的认证，具体过程如下：

![d2MjZq.png](https://s1.ax1x.com/2020/08/25/d2MjZq.png)

1、客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。

2、服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务器端的证书，即公钥证书

3、客户端使用服务端返回的信息验证服务器的合法性，包括：

证书是否过期

发型服务器证书的CA是否可靠

返回的公钥是否能正确解开返回证书中的数字签名

服务器证书上的域名是否和服务器的实际域名相匹配

验证通过后，将继续进行通信，否则，终止通信

4、服务端要求客户端发送客户端的证书，客户端会将自己的证书发送至服务端

5、验证客户端的证书，通过验证后，会获得客户端的公钥

6、客户端向服务端发送自己所能支持的对称加密方案，供服务器端进行选择

7、服务器端在客户端提供的加密方案中选择加密程度最高的加密方式

8、将加密方案通过使用之前获取到的公钥进行加密，返回给客户端

9、客户端收到服务端返回的加密方案密文后，使用自己的私钥进行解密，获取具体加密方式，而后，产生该加密方式的随机码，用作加密过程中的密钥，使用之前从服务端证书中获取到的公钥进行加密后，发送给服务端

10、服务端收到客户端发送的消息后，使用自己的私钥进行解密，获取对称加密的密钥，在接下来的会话中，服务器和客户端将会使用该密码进行对称加密，保证通信过程中信息的安全。

### SSL单向认证和双向认证的区别

1、单向认证只要求站点部署了ssl证书就行，任何用户都可以去访问（IP被限制除外等），只是服务端提供了身份认证。而双向认证则是需要是服务端需要客户端提供身份认证，只能是服务端允许的客户能去访问，安全性相对于要高一些

2、双向认证SSL 协议的具体通讯过程，这种情况要求服务器和客户端双方都有证书。

3、单向认证SSL 协议不需要客户端拥有CA证书，以及在协商对称密码方案，对称通话密钥时，服务器发送给客户端的是没有加过密的（这并不影响SSL过程的安全性）密码方案。

4、如果有第三方攻击，获得的只是加密的数据，第三方要获得有用的信息，就需要对加密的数据进行解密，这时候的安全就依赖于密码方案的安全。而幸运的是，目前所用的密码方案，只要通讯密钥长度足够的长，就足够的安全。这也是我们强调要求使用128位加密通讯的原因。

5、**一般Web应用都是采用单向认证的**，原因很简单，用户数目广泛，且无需做在通讯层做用户身份验证，一般都在应用逻辑层来保证用户的合法登入。**但如果是企业应用对接，情况就不一样，可能会要求对客户端（相对而言）做身份验证。这时就需要做双向认证。**

参考：[SSL单向认证和双向认证的区别](https://www.wosign.com/FAQ/faq_2018102901.htm)

