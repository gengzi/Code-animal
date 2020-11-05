[TOC]

## 目标

* 了解docker安装和部署jenkins

  参考：[Ubuntu怎么开启/关闭防火墙](https://blog.csdn.net/smileui/article/details/87909393)

  [jenkins 更改插件下载镜像地址为国内镜像下载地址](https://www.jianshu.com/p/ac23d7028427)

  [ubuntu 下 maven安装](https://developer.aliyun.com/mirror/maven)

  [Jenkins学习笔记（一）：Docker安装Jenkins及自动部署Maven项目](https://blog.csdn.net/weixin_44249490/article/details/103687307)

  [Maven 镜像](https://developer.aliyun.com/mirror/maven)

  [Jenkins配置maven](https://www.cnblogs.com/xiao987334176/p/11433636.html)

  [docker-compose部署配置jenkins的详细教程](https://www.jb51.net/article/190632.htm) 可以参考

  [#!/bin/bash表示什么意思](https://blog.csdn.net/loryliu/article/details/23817975)

  [jenkins+github+docker+maven自动化构建部署](https://zhuanlan.zhihu.com/p/64946155)

  [Docker下安装Jenkins并配置Maven](https://www.reinforce.cn/t/658.html)

  [linux怎么将一个文件夹链接到另一个文件夹上](https://blog.csdn.net/CSDN_LSD/article/details/78761323)

## Jenkins docker 安装

jenkins 官方地址：https://www.jenkins.io/zh

安装环境：ubuntu 18.04.4 LTS

注意： docker 的安装对linux版本有要求，建议参考官方文档

建议使用大的云厂商的云服务器，因为对于某些软件的下载，他们的节点快很多。自己的服务器，要改好多国内的镜像地址，否则下载东西要慢死。

安装步骤：

* 安装docker

  可以参考：https://docs.docker.com/engine/install/

  jenkins docker 镜像地址：https://hub.docker.com/r/jenkins/jenkins

  略，建议参考官方文档

* 安装docker-compose

  可以参考： https://www.runoob.com/docker/docker-compose.html

  具体命令：来源于官方文档

```shell
## 安装docker-compose


#运行以下命令以下载Docker Compose的当前稳定版本：

sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 将可执行权限应用于二进制文件：

sudo chmod +x /usr/local/bin/docker-compose

# 如果命令docker-compose在安装后失败，请检查您的路径。您还可以创建指向/usr/bin或路径中任何其他目录的符号链接。

sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# 测试安装

$ docker-compose --version
docker-compose version 1.27.4, build 1110ad01

```

* 编写docker-compose.yml 文件

  在目录下新建 docker-compose.yml  内容如下：

  ```yml
  # docker引擎对应所支持的docker-compose文本格式
  version: '2'
  
  # jenkins 服务
  services:
  jenkins:
    # 使用的镜像 这里不使用 docker hub 官方提供的，因为那个已经好长时间没有更新了
    # 使用 jenkins 官方提供的 jenkins/jenkins 或者  jenkinsci/blueocean
    image: jenkinsci/blueocean
    volumes:
      - /data/jenkins_home:/var/jenkins_home    # 将容器中的 /var/jenkins_home 映射到宿主机的 /data/jenkins/
      - /var/run/docker.sock:/var/run/docker.sock  # 允许容器与容器与Docker守护进程通信 ，也就是使用docker 命令 ， 比如 容器中需要实例化其他Docker容器
      - /usr/bin/docker:/usr/bin/docker
      - /usr/lib/x86_64-linux-gnu/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7
    ports:    #端口映射  宿主机:容器
      - "8029:8080"
    expose:  #暴露端口，但不映射到宿主机，只被连接的服务访问。仅可以指定内部端口为参数
      - "8080"
      - "50000"
    privileged: true  # 使用该参数，container内的root拥有真正的root权限，否则，container内的root只是外部的一个普通用户权限。
    user: root  # 指定用户是 root 用户，来解决没有权限的问题
    restart: always    #是否随docker服务启动重启
    container_name: jenkins  # 容器实例后的名称
    environment: # environment 和 Dockerfile 中的 ENV 指令一样会把变量一直保存在镜像、容器中，类似 docker run -e 的效果。设置容器的环境变量
      - JAVA_OPTS:'-Djava.util.logging.config.file=/var/jenkins_home/log.properties'
      - TZ=Asia/Shanghai #这里设置容器的时区为亚洲上海
  ```

  可以手动修改一下，docker 容器要映射的宿主机的端口

  注意：使用xshell 直接粘贴会出现注释覆盖，可以先创建一个不带后缀名的文件，粘贴后，再更正名称。

* 启动jenkins

  注意，安装jenkins 的主机，对配置是有要求的

  - 256MB可用内存
  - 1GB可用磁盘空间(作为一个[Docker](https://www.jenkins.io/zh/doc/book/installing/#docker)容器运行jenkins的话推荐10GB)

  反正越高越好，否则在之后的构建过程中，可能会出现内存溢出等等情况

```shell
# 在docker-compose.yml 的目录下执行
docker-compose up

# 等待启动，在启动完成的结尾会有一个初始化密码
jenkins    | Please use the following password to proceed to installation:
jenkins    |
jenkins    | 1acc6f86385f4a9aa9de3051105d3286
```

  jenkins 的访问地址：http://宿主机ip:8029/

  访问失败，请检查如下：

  防火墙是否关闭(ubuntu 关闭防火墙 sudo ufw disable)

  端口是否正确

  docker 容器是否正常启动

## Jenkins 的使用

[Jenkins详细教程](https://www.jianshu.com/p/5f671aca2b5a)

[Jenkins与Docker的自动化CI/CD实战](https://blog.51cto.com/lizhenliang/2159817)

[Jenkins自动化部署入门详细教程](https://www.cnblogs.com/wfd360/p/11314697.html) 建议参考，写的很详细

[Jenkins学习笔记（一）：Docker安装Jenkins及自动部署Maven项目](https://blog.csdn.net/weixin_44249490/article/details/103687307) 可以参考

图示和流程不再阐述，具体看上面的

注意：**对于jenkins 全局配置，jdk maven git 等配置，建议手动下载好，因为网络原因，对某些资源下载速度过于缓慢**，如果你使用的是 docker 启动的jenkins，请先进入jenkins容器中，再下载这些资源并配置。

### 组件安装

* 第一种

  对于 jdk maven组件可以直接使用宿主机的，在构建jenkins 镜像时，将jdk 的路径和maven 的路径挂载到容器上。在此篇演示中，我将maven的安装资源，放在了/data/jenkins_home/install/maven 下，那么对应到jenkins中的路径就是 /var/jenkins_home/install/maven ,在jenkins 的系统配置中，配置对应的路径即可。

* 第二种

  在容器内安装jdk和maven ，应该也是可以的。

### jenkins 中jdk配置

如果项目编译的jdk版本运行jenkins容器的java版本一致，也可以直接使用运行jenkins容器的jdk，可以在 系统管理-系统信息 中的 java 中看到，jdk的版本和路径 /opt/java/openjdk/jre 把jre 删掉就是jdk的路径。

jdk 安装过程：

```shell
# 宿主机
## 下载jdk，拷贝到 /data/jenkins_home/install 目录下
# 解压jdk 的压缩包
tar -zxvf jdk1.7.tar.gz
##-----可以不执行以下操作，jenkins 不会读取这些配置 ，只有宿主机可以用--------------------
# 设置环境变量
vi /etc/profile
在profile文件末尾添加：
export JAVA_HOME=/data/jenkins_home/install/jdk1.7
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
export PATH=$PATH:${JAVA_PATH}
# 使环境变量生效
source /etc/profile
# 测试jdk是否安装成功
java -version
echo $JAVA_HOME
```

在jenkins 的系统配置-全局工具配置-jdk 配置

![BWYGgP.png](https://s1.ax1x.com/2020/11/05/BWYGgP.png)

### jenkins 中 maven 安装和配置

```shell
# 宿主机
# 下载maven安装包，拷贝到 /data/jenkins_home/install 目录下
wget http://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
# 解压压缩包
tar -zxvf apache-maven-3.3.9-bin.tar.gz
# 删除压缩包
rm apache-maven-3.3.9-bin.tar.gz
# 修改maven 包名称
mv ./apache-maven-3.3.9/ ./maven
##------------可以不执行以下操作，jenkins 不会读取这些配置 ，只有宿主机可以用--------------------
# 配置maven环境变量
vi /etc/profile
    在文件的最下面添加：
# set Maven environment
export MAVEN_HOME=/data/jenkins_home/install/maven
export PATH=$MAVEN_HOME/bin:$PATH
# 生效profile文件
source /etc/profile
# 检查maven是否配置成功（显示maven home、java home等信息则配置成功）
mvn -v
```

修改maven 镜像为国内

进入maven/conf  修改 settings.xml 文件,在`<mirrors></mirrors>`标签中添加 mirror 子节点:

```xml
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>*</mirrorOf>
    <name>aliyun</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

在jenkins 的系统配置-全局工具配置-Maven 配置

![BWYcuT.png](https://s1.ax1x.com/2020/11/05/BWYcuT.png)

![BWYfUJ.png](https://s1.ax1x.com/2020/11/05/BWYfUJ.png)

### jenkins 的git安装

直接使用配置中的自动下载。。没啥问题。

 

### 其他命令：

```shell
# 进入容器
docker exec -it jenkins /bin/bash
 
# 我曾经尝试使用linux链接方式，将 maven 资源文件，链接到 /data/jenkins_home/ 目录下，但是在jenkins 容器中无法进入maven 目录，所以不能使用链接的方式，只能将需要的资源文件，挂载到容器的目录上
ln -s 源文件位置 目标文件位置
 
 
```

 

## 出现的问题以及解决方法

1.  在构建某个工程，出现卡死情况，关闭也关闭不了

   参考：[jenkins 的关闭与启动](https://www.jb51.net/article/98554.htm)

   可以尝试重启jenkins，重启方式很简单

   http://你的jenkins地址/restart 访问该路径，出现是否重启的页面，点击重启即可。

2.  Build step 'Invoke top-level Maven targets' marked build as failure Finished 构建报错

   参考：[Build step 'Invoke top-level Maven targets' marked build as failure Finished ](https://blog.csdn.net/shenjuntao520/article/details/103048287)

   常见情况就是，内存溢出，调大运行内存

3. 高版本Jenkins无法关闭跨站请求伪造保护（CSRF）

   不关这个，github 的webhook 回调jenkins 的地址，会出现 403 错误。

   在高版本的jenkins，把这个跨站脚本攻击的配置注释掉了。不能再手动配置。

   网上大概有几种方式：

   * 在启动命令设置关闭 csrf

     https://www.cnblogs.com/kazihuo/p/12937071.html

   * 在jenkins 配置文件设置，关闭csrf

     https://blog.csdn.net/qq_39507330/article/details/106494621

   * 全局配置github的 hookurl

     不过，可以在全局配置中配置 github，配置 Hook URL

     具体参考：[Jenkins与Github集成](https://www.cnblogs.com/weschen/p/6867885.html)

     但是这种好像也不太行，一个工程push，会连锁所有的构建item

   * 创建一个公共的web工程

     创建一个web工程，用于接收github 的回调请求，当接收到，再访问内网jenkins的构建路径，让jenkins进行构建（http://jenkins的地址/job/codecopy-two/build?token=TOKEN_NAME）

4. [Docker](https://testerhome.com/topics/node48) docker 启动 jenkins，将 jdk、mven、git 挂载在容器中，构建时提示找不到命令

   参考 https://testerhome.com/topics/18799 https://testerhome.com/topics/23906

5. 其他问题

   [docker重新进入容器时“/etc/profile”中环境变量失效问题的解决](https://blog.csdn.net/dongdong9223/article/details/81094657)

### 小技巧

遇到一些问题后，可以优先排查，路径是否正确，防火墙是否关闭，执行目录是否有权限，修改的配置是否生效，可能需要手动重新加载，避免在浪费大量时间后，才发现是个很简单的问题导致的。

 

 

 

 

 

 

 

 

     

 

 

 

 

 

 