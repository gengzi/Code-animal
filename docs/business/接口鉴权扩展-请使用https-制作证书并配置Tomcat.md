[TOC]

## 目标

* 了解制作证书步骤并配置Tomca

  参考：[openssl 自签名证书 - 制作证书（二）](https://www.jianshu.com/p/37ded4da1095)

  [Nginx https 双向认证](https://www.cnblogs.com/yelao/p/9486882.html)

  [openssl制作证书全过程及https实现](https://www.cnblogs.com/poziiey/p/12491214.html)

  [常见证书格式和转换](https://blog.csdn.net/justinjing0612/article/details/7770301)

  [常见证书格式及相互转换](https://www.cnblogs.com/lzjsky/archive/2010/11/14/1877143.html)

  [SpringBoot配置https ](https://www.jianshu.com/p/3c90e336f794)

  [https双向认证](https://www.wosign.com/tag/httpsshuangxiangrenzheng.htm)

该篇自己模拟CA机构，制作证书，来演示模拟HTTPS 的简单使用，对于商业使用，需要去服务供应商申请ssl证书，有免费的也有收费的，申请后配置一下即可。

## 制作证书

工具： Open SSL                 windows 下载地址：[Win32 / Win64 OpenSSL](http://slproweb.com/products/Win32OpenSSL.html)

安装后，在 bin 目录可以使用 openssl.exe 来制作证书。

制作证书前，先了解一下证书的一些格式和之间的转换。

* 证书格式

  pem、key：私钥文件，对数据进行加密解密,pem如果只含私钥的话，一般用.key扩展名，而且可以有密码保护

  csr：证书签名请求文件，将其提交给证书颁发机构（ca、CA）对证书签名

  crt：由证书颁发机构（ca、CA）签名后的证书或者自签名证书，该证书包含证书持有人的信息、持有人的公钥以及签署者的签名等信息

  PKCS12(P12)：带有私钥的证书，包含了公钥和私钥的二进制格式的证书形式，通常有保护密码,以 pfx 作为证书文件后缀名。

  cer ：二进制编码的证书（DER），证书中没有私钥，DER 编码二进制格式的证书文件，以 .cer 作为证书文件后缀名。Base64 编码的证书（PEM）：证书中没有私钥，Base64 编码格式的证书文件，也是以 .cer 作为证书文件后缀名。der,cer文件一般是二进制格式的，只放证书，不含私钥。

* 证书格式转换

  参考：[pem转换为crt和key 简单方法](https://www.fujieace.com/jingyan/pem-crt-key.html)

  [pem文件转cer文件](https://www.cnblogs.com/StephenWu/p/6791714.html)

  pem 转 crt key 可以直接重名名

  转 cer 好像也可以直接重名名

  等需要转换的时候，再查询一下吧

### Open SSL 的使用

在上篇中介绍了 SSL 的单向认证和双向认证。

单向认证只需要 CA 证书和 服务端证书，双向认证需要 客户端证书，CA 证书 和 服务端证书。

证书制作流程：

1. 生成私钥
2. 生成证书请求文件
3. 使用CA 根证书签发 证书请求文件

双向认证制作证书演示：

自己来扮演 ca机构，对客户端和服务端进行认证。

参考：[openssl制作证书全过程及https实现](https://www.cnblogs.com/poziiey/p/12491214.html)

[openssl 自签名证书 - 制作证书（二）](https://www.jianshu.com/p/37ded4da1095)

先在open ssl bin目录中创建 ca、client、server 文件夹，用于存放生成的文件目录

* 制作CA 根证书

  在 bin 目录下执行

  ```shell
  # 创建私钥
  genrsa -out ca/ca-key.pem 1024
  # 创建证书请求
  req -new -out ca/ca-req.csr -key ca/ca-key.pem
 
  ## ----------------- 这里要输入一些信息
  Country Name (2 letter code) [AU]:cn 【国家代码两个字母可为空，ca、server、client要一致】
  State or Province Name (full name) [Some-State]:【省份，ca、server、client要一致】
  Locality Name (eg, city) []:【城市】
  Organization Name (eg, company) [Internet Widgits Pty Ltd]:【公司名，ca、server、client要一致】
  Organizational Unit Name (eg, section) []:【组织名】
  Common Name (e.g. server FQDN or YOUR name) []:【不可为空，全限定域名或名字】
  Email Address []:【邮箱】
 
  Please enter the following 'extra' attributes
  to be sent with your certificate request
  A challenge password []:【输入密码】
  An optional company name []:【可选的公司名】
 
  上面输入的ca请求信息在后续申请使用该ca证书签名的请求证书要保证一致。
  ## ----------------------
 
  # 自签署证书，自己作为ca机构签发根证书（自签发证书）
  x509 -req -in ca/ca-req.csr -out ca/ca-cert.pem -signkey ca/ca-key.pem -days 3650
  # 转换证书格式
  pkcs12 -export -clcerts -in ca/ca-cert.pem -inkey ca/ca-key.pem -out ca/ca.p12
 
    ## 参数说明
    genrsa：使用RSA算法生成私钥
    -out：输出文件路径
    1024 ：私钥长度
    req：执行证书签发命令
    -new：新的证书签发请求
    -key：指定私钥文件的路径
    x509：用于自签名证书，生成x509格式的证书
    -req：请求签名
    -days：证书有效期
    -signkey：证书签发的私钥
    -in：证书请求文件，有效的文件路径
    -out：ca签名后的证书输出路径
  ```

***

* 制作 server 证书

  步骤与上面基本一致

  ```shell
    # 创建私钥
    genrsa -out server/server-key.pem 1024
    # 创建证书请求
    req -new -out server/server-req.csr -key server/server-key.pem
    ## 这里再次输入与上面一致的内容 Common Name 要输入域名或者服务器ip地址
 
    # 签署证书，这里的签署证书，使用的是 ca 的证书 和 私钥 来签署的
    x509 -req -in server/server-req.csr -out server/server-cert.pem -signkey server/server-key.pem -CA ca/ca-cert.pem -CAkey ca/ca-key.pem -CAcreateserial -days 3650
    # 转换证书格式
    pkcs12 -export -clcerts -in server/server-cert.pem -inkey server/server-key.pem -out server/server.p12
  ```

 

* 制作 client 证书

  步骤与上面基本一致

  ```shell
  # 创建私钥
  genrsa -out client/client-key.pem 1024
  # 创建证书请求
  req -new -out client/client-req.csr -key client/client-key.pem
  ## 这里再次输入与上面一致的内容
 
  # 签署证书，这里的签署证书，使用的是 ca 的证书 和 私钥 来签署的
  x509 -req -in client/client-req.csr -out client/client-cert.pem -signkey client/client-key.pem -CA ca/ca-cert.pem -CAkey ca/ca-key.pem -CAcreateserial -days 3650
  # 转换证书格式
  pkcs12 -export -clcerts -in client/client-cert.pem -inkey client/client-key.pem -out client/client.p12
  ```

执行完上述步骤，你会在  ca、client、server 这些文件夹，看到生成的证书文件和私钥文件等

大致说下： p12 转换后证书文件，client-cert.pem 证书文件 ，client-key.pem 私钥文件，client-req.csr 证书请求文件，其他类似。在配置tomcat 或者 nginx 配置时，需要转换一下文件格式。

## Tomcat 配置验证https

配置tomcat主要将证书文件引入，这样在tomcat提供web服务的时候，就可以使用证书进行ssl的认证了

* 将ca证书生成jks文件

  在jdk 的bin 目录下执行命令,  需要更换一下目录，这里是用我本地的路径

  ```shell
  keytool -keystore D:\bcinstall\OpenSSL-Win64\bin\jks\truststore.jks -keypass 222222 -storepass 222222 -alias ca -import -trustcacerts -file D:\bcinstall\OpenSSL-Win64\bin\ca\ca-cert.pem
  # -keypass  -storepass   密码
  ```

配置tomcat

修改tomcat的配置文件（conf/server.xml）添加一个新的connector，内容如下（tips：注：keystoreFile对应server端的jks文件，keystorePass对应其密码）

```shell
<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
    maxThreads="150" scheme="https" secure="true"
    clientAuth="true" sslProtocol="TLS"
    keystoreFile="C:\OpenSSL-Win64\bin\server\server.p12" keystorePass="test" keystoreType="PKCS12"
    truststoreFile="C:\OpenSSL-Win64\bin\jks\truststore.jks" truststorePass="222222" truststoreType="JKS"/>
```

浏览器导入证书

将ca.p12，client.p12分别导入到IE中去（打开IE->Internet选项->内容->证书）。
ca.p12导入至受信任的根证书颁发机构，client.p12导入至个人。

启动tomcat 验证,在浏览器地址前，出现一个小锁，说明成功。

https://127.0.0.1:8443

## spring boot 配置 https 访问

可参考：[codecopy](https://github.com/gengzi/codecopy/tree/master/src/main/resources) 下的 certificate 目录和 server.p12 truststore.jks 文件

将服务端证书文件和jks 文件拷贝至resources 目录下

在application.yml 文件中增加以下配置

```yaml
# 开启tomcat 的https
server:
  ssl:
    key-store: classpath:server.p12
    key-store-type: PKCS12
    trust-store: classpath:truststore.jks
    trust-store-password: 222222
    trust-store-type: JKS
    key-store-password: test
    enabled: true
```

 