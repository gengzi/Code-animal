[TOC]

## 目标

* 了解 tomcat 优化配置

## 指定tomcat日志按天输出，保存日志天数为15天

参考：[ubuntu手动安装deb文件](https://help.ubuntu.com/kubuntu/desktopguide/zh_CN/manual-install.html)

[deb安装包](https://pkgs.org/search/?q=cronolog)

[日志分隔工具Cronolog的使用](https://blog.csdn.net/weixin_38860565/article/details/81633234)

[tomcat的catalina.out日志按天生成](https://www.jianshu.com/p/1abc5ae3251c)

日志的按天输出方便我们查看日志，也方便拷贝和查找。

* 下载安装

```shell
# 下载 cronolog
$ wget http://archive.ubuntu.com/ubuntu/pool/universe/c/cronolog/cronolog_1.6.2+rpk-1ubuntu1_amd64.deb
# 安装 cronolog
$ dpkg -i ./cronolog_1.6.2+rpk-1ubuntu1_amd64.deb
# 查看是否安装成功
$ which cronolog
 
# 修改tomcat的 bin/catalina.sh
$ vi ./catalina.sh
 
 
```

* 修改日志输出路径

```shell
修改：
if [ -z "$CATALINA_OUT" ] ; then
      CATALINA_OUT="$CATALINA_BASE"/logs/catalina.out
fi
为：
    if [ -z "$CATALINA_OUT" ] ; then
      CATALINA_OUT="$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out
fi
```

* 删除生成日志文件

```shell
注释：
touch "$CATALINA_OUT"
  为：
#touch "$CATALINA_OUT"
```

* 修改启动脚本参数（有2处）

```shell
修改：
      org.apache.catalina.startup.Bootstrap "$@" start \
      >> "$CATALINA_OUT" 2>&1 "&"
    为：
      org.apache.catalina.startup.Bootstrap "$@" start 2>&1 \
      | /usr/local/sbin/cronolog "$CATALINA_OUT" >> /dev/null &
 
 
      org.apache.catalina.startup.Bootstrap "$@" start \
      >> "$CATALINA_OUT" 2>&1 "&"
    为：
      org.apache.catalina.startup.Bootstrap "$@" start 2>&1 \
      | /usr/local/sbin/cronolog "$CATALINA_OUT" >> /dev/null &
 
```

* 测试

  重启tomcat ，查看 logs 下的日志文件。是否有 catalina.2020-05-06.out 类型

* 设置保存只保存15天的日志

  参考：[Linux使用定时任务每周定时清理45天以前日志](https://www.jb51.net/article/95094.htm)

  [Linux下定时切割Tomcat日志并删除指定天数前的日志记录](https://www.jb51.net/article/121346.htm)

  在阿里巴巴的java编码规范中指出：

  日志规约第二条：
  【强制】所有日志文件至少保存 15 天，因为有些异常具备以“周”为频次发生的特点。网络
  运行状态、安全相关信息、系统监测、管理后台操作、用户敏感操作需要留存相关的网络日
  志不少于 6 个月。

  太多的日志也会对磁盘有一定的压力，有些工程的日志高达几个G，更有甚者多达十几个G。

  所以可以设置日志范围在15天之内的就可以了。

  使用定时任务（crontab -e）  ，定时删除15前的日志

  ```shell
  # 打开linux定时任务
  $ crontab -e
  # 添加一个定时任务，将 日志位置 换成 真实的日志路径
  # 每周六 5点半 执行, +15 15天之前的
  30 5 * * 6 /bin/find 日志位置/logs/ -mtime +15 -type f -name "catalina.*.out" -exec /bin/rm -f {} \;
 
  ```

## 修改tomcat 加载指定目录下的web 工程

参考：[Catalina\localhost下的xml文件配置](https://blog.csdn.net/weixin_34238642/article/details/91683653)

[tomcat配置虚拟目录](https://www.cnblogs.com/kevinq/p/4822091.html)

在 /apache-tomcat-7.0.85/conf/Catalina/localhost 下创建 ROOT.xml 文件，将下面内容写入

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context path="" docBase="/home/ubuntu/webproject/GsjBlog" reloadable="true"/>
 
```

参数解释：

一般 docBase 放编译后web工程，path 访问这个web工程的跟路径

```
docBase
 
指定web应用程序的文档根目录或者war文件的路径名，你可以指定目录或war文件的绝对路径名，也可以指定相对于Host元素的appBase目录的相对路径名。
 
path
 
web应用的上下文路径，通过匹配URI来运行适当的web应用。一个Host中的上下文路径必须是唯一的。如果指定一个上下文路径为空字符串("")，则定义了这个Host的默认web应用，会被用来处理所有没有被分配给其他web应用的请求(即如果没有找到相应的web应用，则执行这个默认的web应用)
 
reloadable
 
如果设置为true，则tomcat服务器在运行时，会监视WEB-INF/classes和WEB-INF/lib目录下类的改变，如果发现有类被更新，tomcat服务器将自动重新加载该web应用程序。这个特性在应用程序的开发阶段非常有用，但是它需要额外的运行时开销，所以在产品法布时不建议使用。该属性的默认值是false
```

 

## 创建tomcat 启动脚本

创建 tomcat-restart.sh 文件 , 将cd 的目录改为真实的 tomcat 目录

```shell
#!/bin/sh
cd /home/ubuntu/tomcat/tomcat-two/apache-tomcat-7.0.85/bin
sh shutdown.sh
pid=$(ps -ef | grep tomcat | grep -v grep | awk '{print $2}')
 
if [ -n "$pid" ]; then
        kill -9 $pid
fi
sleep 5s
sh startup.sh
 
```

执行：

```shell
sh   tomcat-restart.sh
```

 

## 指定tomcat运行web项目的jvm配置

阿里巴巴java编程规范指出：

【推荐】在线上生产环境， JVM 的 Xms 和 Xmx 设置一样大小的内存容量，避免在 GC 后调整
堆大小带来的压力。

```
修改tomcat 配置： 修改 /bin/catlina.sh  在第一行加入
--------------------------------------------------------------------------------------------
JAVA_OPTS="-Djava.awt.headless=true -Dfile.encoding=UTF-8    -server -Xms1024m -Xmx1024m    -XX:NewSize=512m -XX:MaxNewSize=512m -XX:PermSize=512m   -XX:MaxPermSize=512m -XX:+DisableExplicitGC"
---------------------------------------------------------------------------------------------
-Xms – 指定初始化时化的内存
-Xmx – 指定最大可用内存
 
 
```

 

## 设置tomcat 热部署

参考：[tomcat热部署和热加载（转载）](https://www.iteye.com/blog/langgufu-2035505)

[Tomcat热部署三种方式的详细说明](https://blog.csdn.net/troub_cy/article/details/90052283)