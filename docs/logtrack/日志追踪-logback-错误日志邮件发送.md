[TOC]

## 目标

* 了解logback配置错误日志邮件发送

  参考：[Appenders](http://www.logback.cn/04%E7%AC%AC%E5%9B%9B%E7%AB%A0Appenders.html)

  [logback发送告警邮件](https://cloud.tencent.com/developer/article/1424740)

  [logback 发送邮件和自定义发送邮件；java类发送邮件](https://www.cnblogs.com/zhouyantong/p/7682941.html)

Appender 最基本的责任是将日志事件进行输出。比如最常见的在logback.xm配置的 ConsoleAppender （控制台）、RollingFileAppender（滚动文件）等等，下面介绍的SMTPAppender，就是用来支持邮件日志事件。

## SMTPAppender

* SMTP [百度百科-SMTP](https://baike.baidu.com/item/SMTP/175887?fr=aladdin)

  SMTP是一种提供可靠且有效的[电子邮件传输](https://baike.baidu.com/item/电子邮件传输/22035911)的协议。SMTP是一个相对简单的基于[文本](https://baike.baidu.com/item/文本)的[协议](https://baike.baidu.com/item/协议)。在其之上指定了一条[消息](https://baike.baidu.com/item/消息)的一个或多个接收者（在大多数情况下被确认是存在的），然后消息文本会被传输。

以下内容大多引用官方文档 Appenders 一节，只是增加某些细节的说明

### 介绍

[`SMTPAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SMTPAppender.html) 收集日志事件到一个或多个固定大小的缓冲区，当用户指定的事件发生时，将从缓冲区中取出适当的内容进行发送。SMTP 邮件是**异步发送**的。默认情况下，当**日志的级别为 ERROR **时，邮件发送将会**被触发**。而且默认的情况下，所有事件都使用同一个缓冲区。

### 基本配置

请关注加粗部分的内容

| 属性名              | 类型                                                         | 描述                                                         |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| smtpHost            | String                                                       | SMTP 服务器的主机名。强制性的。 **常见的163邮箱：smtp.163.com，QQ邮箱：smtp.qq.com** |
| smtpPort            | int                                                          | SMPT 服务监听的端口。默认为 25.**（未启用SSL情况下），启用SSL情况下，需要查询下端口** |
| to                  | String                                                       | 接收者的邮件地址。触发事件发送给接收者。多个收件人可以使用逗号(,)分隔，或者使用多个 `<to>` 元素来指定。 |
| from                | String                                                       | `SMTPAppender` 使用的发件人，格式遵循[邮件通用格式](http://en.wikipedia.org/wiki/Email_address)，如果你想要包含发送者的名字，使用这种格式 " Adam Smith <smith@moral.org>"，那么邮件将会显示收件人为 " Adam Smith \[smith@moral.org\](mailto:smith@moral.org\)"  **注意，这里发件人，是需要指定邮箱的，不能随便指定** |
| subject             | String                                                       | 邮件的主题。它可以是通过 [PatternLayout](https://logback.qos.ch/manual/layouts.html#ClassicPatternLayout) 转换后的有效值。关于 Layout 将在接下来的章节讨论。 邮件应该有一个主题行，对应触发的邮件信息。 假设 `subject` 的值为："Log: %Logger - %msg"，触发事件的 logger 名为 "com.foo.Bar"，并且日志信息为 "Hello world"。那么发出的邮件信息将会有一个名为 "Log: com.foo.Bar - Hello World" 的主题行。 默认情况下，这个属性的值为 "%logger{20} - %m" |
| discriminator       | [Discriminator](https://logback.qos.ch/xref/ch/qos/logback/core/sift/Discriminator.html) | 在 Discriminator 的帮助下，`SMTPAppender` 根据 discriminator 返回的值可以将不同日志事件分散到不同的缓冲区中。默认的 discriminator 将返回同一个值，所以所有的事件都使用同一个缓冲区。 |
| evaluator           | [IEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/IEvaluator.html) | 通过创建一个新的 元素来声明此选项。通过 `class` 属性指定 class 的名字表示用户希望通过哪个类来满足 `SMTPAppender` 的 `Evaluator` 的需要。 如果没有指定此选项，当触发一个大于等于 *ERROR* 级别的事件时，`SMTPAppender` 将会被分配一个 [OnErrorEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnErrorEvaluator.html) 的实例。 logback 配备了几个其它的 evaluator，分别叫 [`OnMarkerEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnMarkerEvaluator.html) （将在下面讨论），一个相对强大的 evaluator 叫 [`JaninoEventEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/JaninoEventEvaluator.html)（在[其它章节](https://logback.qos.ch/manual/filters.html#evalutatorFilter)讨论） 以及最近版本才有的一个更加强大的 evaluator 叫 [`GEventEvaluator`](https://logback.qos.ch/manual/filters.html#GEventEvaluator)。 |
| cyclicBufferTracker | [`CyclicBufferTracker`](https://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) | 从名字可以看出，是一个 `CyclicBufferTracker` 的实例追踪循环缓冲区。它基于 discriminator 返回的 key （见上）。 如果你不想指定一个 cyclicBufferTracker，那么将会自动创建一个 [CyclicBufferTracker](https://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) 的实例。默认的，这个实例用来保留事件的循环缓冲区的大小为 256。你需要改变 `bufferSize` 选项的大小（见下面） |
| username            | String                                                       | 默认为 null **用户名**                                       |
| password            | String                                                       | 默认为 null **一般是去对应的邮箱网站开通SMTP后的授权码**     |
| STARTTLS            | boolean                                                      | 如果为 true，那么 appender 将会发送 STARTTLS 命令（如果服务器支持）将连接变成 SSL 连接。注意，连接初始的时候是为加密的。默认为 false。 |
| SSL                 | boolean                                                      | 如果为 true，将通过 SSL 连接服务器。**默认为 false。**       |
| charsetEncoding     | String                                                       | 邮件信息将会通过 [charset](http://www.logback.cn/(https:/docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html)]https://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html) 进行编码。默认编码为 "UTF-8" |
| localhost           | String                                                       | 一旦 SMTP 客户端的主机名没有配置正确，例如客户端的 hostname 不是全限定的，那么服务端会拒绝客户端发送的 HELO/EHLO 命令。为了解决这个问题，你可以将 `localhost` 的值设置为客户端主机的全限定名。详情见 [com.sun.mail.smtp](http://javamail.kenai.com/nonav/javadocs/com/sun/mail/smtp/package-summary.html) 包文档中的 "mail.smtp.localhost" 属性。(这个网站已经关闭了...) |
| asynchronousSending | boolean                                                      | **决定邮件传输是否是异步进行。默认为 'true'。**但是，在某些特定的情况下，异步发送不怎么合适。例如，当发生一个严重错误时，你的应用使用 `SMTPAppender` 去发送一个警告，然后退出。但是相关线程可能没有时间去发送警告邮件。在这种情况下，你可以设置该属性的值为 'false'。 |
| includeCallerData   | boolean                                                      | 默认为 false。如果 `asynchronousSending` 的值为 true，并且你希望在日志中看到调用者的信息，你可以设置该属性的值为 true |
| sessionViaJNDI      | boolean                                                      | `SMTPAppender` 基于 `javax.mail.Session` 来发送邮件信息。默认情况下，该属性的值为 false，所以需要用户指定相关属性通过 `SMTPAppender` 来构建 `javax.mail.Session` 实例。如果设置为 true，`javax.mail.Session` 实例将会通过 JNDI 来获取。参见 `jndiLocation` 属性。 通过 JNDI 获取 `Session` 实例可以减少需要配置的数量，使你的应用减少重复([dryer](http://en.wikipedia.org/wiki/Don't_repeat_yourself))的工作。更多关于在 Tomcat 配置 JNDI 的信息请参考 [JNDI Resources How-to](http://tomcat.apache.org/tomcat-6.0-doc/jndi-resources-howto.html#JavaMail_Sessions)。 `注意`：通过 JNDI 获取 `Session` 的时候请移除 web 应用下 *WEB-INF/lib* 文件夹下的 *mail.jar* 与 *activation.jar*。 |
| jndiLocation        | String                                                       | JNDI 中放置 javax.mail.Session 的地方。默认为：" java:comp/env/mail/Session " |

配置示例：

```xml
<configuration>
    <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
        <smtpHost>SMTP 服务器的地址</smtpHost>
        <to>收件人1</to>
        <to>收件人2</to>
        <from>发件人</from>
        <subject>TESTING: %logger{20} - %m</subject>
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%date %-5level %logger{35} - %message%n</pattern>
        </layout>
    </appender>
 
    <root level="DEBUG">
        <appender-ref ref="EMAIL" />
    </root>
</configuration>
```

 

### 格式化日志

从上述的配置中，layout 标签的配置，用于格式化日志。收件者收到的邮件是经过 `PatternLayout` 格式化后的日志

类似的格式如下：

![smtpAppender1](http://www.logback.cn/images/smtpAppender1.jpg)

如果采用 HTMLLayout 配置，将日志格式化为 HTML 表格。你可以更改行与列的顺序，以及表格的 CSS 样式。查看 [HTMLLayout](https://logback.qos.ch/manual/layouts.html#ClassicHTMLLayout) 文档更详细的信息。

```xml
        <layout class="ch.qos.logback.classic.html.HTMLLayout">
            <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %X{bl-traceid}  %thread %logger Line:%-3L - %msg%n</Pattern>
        </layout>
```

类似的效果：

![1610859215316](https://s3.ax1x.com/2021/02/23/yLwhEd.png)

### 定制缓冲区大小

默认情况下，`SMTPAppender` 会输出最新的 256 条日志信息。下面一个例子设置了不同缓冲区大小。

```xml
<configuration>
    <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
        <smtpHost>${smtpHost}</smtpHost>
        <to>${to}</to>
        <from>${from}</from>
        <subject>%logger{20} - %m</subject>
        <layout class="ch.qos.logback.classic.html.HTMLLayout" />
 
        <cyclicBufferTracker
            class="ch.qos.logback.core.spi.CyclicBufferTracker">
            <!-- 每封邮件只包含一条日志 -->
            <bufferSize>1</bufferSize>
        </cyclicBufferTracker>
    </appender>
 
    <root level="DEBUG">
        <appender-ref ref="EMAIL" />
    </root>
</configuration>
```

 

### 定制触发事件

直接看官方文档吧。

### STARTTLS/SSL 认证

`SMTPAppender` 支持通过用户名/密码以及 STARTTLS，SSL 协议进行认证。STARTTLS 跟 SSL 的不同之处在于，STARTTLS 初始化连接不是加密的，仅仅只有在客户端发出 STARTTLS 命令的时候将连接变为 SSL。在 SSL 模式下，连接在一开始就是被加密的。

配置也直接看官方文档吧，主要配置SSL 或者 STARTTLS 为 true， smtpPort 修改为对应的即可

## 实践

### 邮箱开启SMTP

[logback发送告警邮件](https://cloud.tencent.com/developer/article/1424740) 这篇有163 邮箱的截图

以QQ邮箱示例：

[QQ邮箱的POP3与SMTP服务器是什么？](https://service.mail.qq.com/cgi-bin/help?id=28&no=167&subtype=1)

[如何打开POP3/SMTP/IMAP功能？](https://service.mail.qq.com/cgi-bin/help?subtype=1&no=166&id=28)

开启后，记住授权码即可。（设置-账户）

![1610860122049](https://s3.ax1x.com/2021/02/23/yLw54I.png)

### 依赖

开发环境： Spring boot 2.3.7.RELEASE

`SMTPAppender` 基于 JavaMail API,只需要添加额外的`javax.mail`，如果不是spring boot就还需要添加logback-classic,因为 SMTPAppender 是logback-classic下提供的。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
         <!--发送邮件的依赖-->
        <dependency>
            <groupId>javax.mail</groupId>
            <artifactId>mail</artifactId>
            <version>1.4.7</version>
        </dependency>
        <!--引用 Janino 来提高日志输出-->
        <dependency>
            <groupId>org.codehaus.janino</groupId>
            <artifactId>janino</artifactId>
            <version>3.0.7</version>
        </dependency>
```

 

### logback.log 配置

```xml
<!-- scan 配置文件如果发生改变，将会被重新加载  scanPeriod 检测间隔时间 -->
<configuration scan="true" scanPeriod="60 seconds" debug="false">
 
<!--    如果配置文件的配置有问题，logback 会检测到这个错误并且在控制台打印它的内部状态。但是，如果配置文件没有被找到，logback 不会打印它的内部状态信息，因为没有检测到错误。通过编码方式调用 StatusPrinter.print() 方法会在任何情况下都打印状态信息。
强制输出状态信息：在缺乏状态信息的情况下，要找一个有问题的配置文件很难，特别是在生产环境下。为了能够更好的定位到有问题的配置文件，可以通过系统属性 "logback.statusListenerClass" 来设置 StatusListener 强制输出状态信息。系统属性 "logback.statusListenerClass" 也可以用来在遇到错误的情况下进行输出。-->
   <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />
 
    <!--  邮件 -->
    <!-- SMTP server的地址，必需指定。如网易的SMTP服务器地址是： smtp.163.com -->
    <property name="smtpHost" value="smtp.qq.com"/><!--填入要发送邮件的smtp服务器地址-->
    <!-- SMTP server的端口地址。默认值：25 -->
    <property name="smtpPort" value="465"/>
    <!-- 发送邮件账号，默认为null -->
    <property name="username" value="1164014750@qq.com"/><!--发件人账号-->
    <!-- 发送邮件密码，默认为null -->
    <property name="password" value="xxxx"/><!--发件人密码-->
    <!-- 如果设置为true，appender将会使用SSL连接到日志服务器。默认值：false -->
    <property name="SSL" value="true"/>
    <!-- 指定发送到那个邮箱，可设置多个<to>属性，指定多个目的邮箱 -->
    <property name="email_to" value="332235918@qq.com"/><!--收件人账号多个可以逗号隔开-->
    <!-- 指定发件人名称。如果设置成“&lt;ADMIN&gt; ”，则邮件发件人将会是“<ADMIN> ” -->
    <!--    SMTPAppender 使用的发件人，格式遵循邮件通用格式，如果你想要包含发送者的名字，使用这种格式 " Adam Smith <smith@moral.org>"，那么邮件将会显示收件人为 " Adam Smith \smith@moral.org\"-->
    <property name="email_from" value="1164014750@qq.com"/>
    <!-- 指定emial的标题，它需要满足PatternLayout中的格式要求。如果设置成“Log: %logger - %msg ”，就案例来讲，则发送邮件时，标题为“【Error】: com.foo.Bar - Hello World ”。 默认值："%logger{20} - %m". -->
    <property name="email_subject" value="【Error】: %logger"/>
    <!-- ERROR邮件发送 -->
    <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
        <smtpHost>${smtpHost}</smtpHost>
        <smtpPort>${smtpPort}</smtpPort>
        <username>${username}</username>
        <password>${password}</password>
        <!--决定邮件传输是否是异步进行。默认为 'true'。但是，在某些特定的情况下，异步发送不怎么合适。例如，当发生一个严重错误时，你的应用使用 SMTPAppender 去发送一个警告，然后退出。但是相关线程可能没有时间去发送警告邮件。在这种情况下，你可以设置该属性的值为 'false'。-->
        <asynchronousSending>false</asynchronousSending>
        <SSL>${SSL}</SSL>
        <to>${email_to}</to>
        <from>${email_from}</from>
        <subject>${email_subject}</subject>
        　　　　 <!-- html格式-->
        <layout class="ch.qos.logback.classic.html.HTMLLayout">
            <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %X{bl-traceid}  %thread %logger Line:%-3L - %msg%n</Pattern>
        </layout>
        　　　　 <!-- 这里采用等级过滤器 指定等级相符才发送 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <cyclicBufferTracker class="ch.qos.logback.core.spi.CyclicBufferTracker">
            <!-- 每个电子邮件只发送一个日志条目 -->
            <bufferSize>1</bufferSize>
        </cyclicBufferTracker>
    </appender>
 
 
    <root level="INFO" additivity="false">
        <appender-ref ref="EMAIL"/>
    </root>
</configuration>
```

注意：在上述配置中

* 最好配置 scan ，便于测试。修改logback.xml文件后，会自动在配置时间内刷新配置。

* 配置OnConsoleStatusListener，便于强制输出logback的一些问题。在发送邮件时，可能会打印一些错误，便于我们查找问题。（之前一直没有打印此日志，测试时一直发送邮件不成功，不知道什么原因。。。跟踪了一下源码方法，才看到打印的异常信息）

  ```java
  // 在logback的源码SMTPAppenderBase 中，发送邮件的方法中 sendBuffer()      
  try{
          // ....
  } catch (Exception e) {
      // 发送电子邮件时发生的错误
     addError("Error occurred while sending e-mail notification.", e);
  }
  ```

* 配置 from 标签的的发件人，格式遵循邮件通用格式，建议写邮箱即可。如果不符合格式可能发送不成功

* 使用的QQ邮箱，应该是默认开启了SSL，所以需要更换了STMP的端口并设置SSL为true

* 在上述配置中，配置发送邮件为同步策略便于测试查看，建议实际使用还是异步发送邮件，不影响真正的主流程。

 

### 代码示例

```java
       // 很简单，打印error日志即可。
       log.error("测试发送邮件");
```

 

 