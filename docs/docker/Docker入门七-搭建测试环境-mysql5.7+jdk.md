[TOC]

## 目标

* 搭建测试环境和配置

* 上传创建的容器到docker hub

  参考：[docker在使用JAVA生产、测试、开发环境的部署流程](https://blog.csdn.net/bbwangj/article/details/79550933)

  参考：[[如何通过Docker快速搭建测试环境？](https://www.cnblogs.com/hg-super-man/p/10908216.html)]

  参考：[docker构建Java Web + Mysql运行环境](https://blog.csdn.net/sinat_27629035/article/details/60125097)

  [[docker部署Javaweb项目（jdk+tomcat+mysql）](https://www.cnblogs.com/Crysta1/p/10943704.html)](https://www.cnblogs.com/Crysta1/p/10943704.html)

  [Docker搭建mysql+jdk1.8+tomcat运行容器](https://blog.csdn.net/weixin_40322495/article/details/82621105)

  [[利用Docker搭建java项目开发环境](https://www.cnblogs.com/wxjnew/p/7118000.html)](https://www.cnblogs.com/wxjnew/p/7118000.html)

  [使用Docker快速搭建生产环境](https://blog.csdn.net/u012206617/article/details/84561800)

## 搭建测试环境和配置

### 在宿主机上安装docker

参考 ：[Ubuntu Docker 安装](https://www.runoob.com/docker/ubuntu-docker-install.html)

参考：[Windows Docker 安装](https://www.runoob.com/docker/windows-docker-install.html)

以下所有操作都在 ubuntu 环境中：

### 运行docker

```shell
# 检查docker 版本,看docker 是否安装成功
$ docker version
# 若输出了 Docker 的版本号，则说明安装成功了，可通过以下命令启动 Docker 服务 (ubuntu)：
# 一旦 Docker 服务启动完毕，就可以开始使用 Docker 了。
$ service docker start
```

### pull 基础镜像 ubuntu

```shell
# 下载ubuntu镜像
$ docker pull ubuntu
 
Using default tag: latest
latest: Pulling from library/ubuntu
Digest: sha256:bec5a2727be7fff3d308193cfde3491f8fba1a2ba392b7546b43a051853a341d
Status: Image is up to date for ubuntu:latest
 
# 查看镜像
$ docker images
 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              4e5021d210f6        2 weeks ago         64.2MB
 
 
```

### 启动容器

容器是在镜像的基础上来运行的，一旦容器启动了，我们就可以进入到容器中，安装自己所需的软件或应用程序。

本例中，所有安装程序都放在了宿主机的/mnt/software目录下，现在需要将其挂载到容器的/mnt/software/目录下。

```shell
# 在/mnt/software 下我们上传了
apache-tomcat-8.5.47.tar.gz
jdk-8u181-linux-x64.tar.gz
mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz
nginx-1.11.3.tar.gz
redis-4.0.8.tar.gz
# 启动ubuntu镜像的容器
$ docker run -i -t -v /mnt/software/:/mnt/software  ubuntu /bin/bash
 
-v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>
 
# 安装软件 jdk  和 tomcat 以及其他软件
# 现在的目标是 安装jdk 和 tomcat 在这个 ubuntu 的镜像中，然后创建成为一个新的镜像。其他的mysql
# nginx redis 等直接使用docker pull 安装使用 （2020年4月6日20:31:39）
 
```

###  安装jdk 和 tomcat

```shell
# 安装jdk，解压 JDK 程序包
$ tar -zxf /mnt/software/jdk-8u181-linux-x64.tar.gz
# 移动 JDK 目录
$ mv jdk1.8.0_181/  /opt/jdk/
# 安装Tomcat，解压Tomcat程序包
$ tar -zxf /mnt/software/apache-tomcat-8.5.47.tar.gz
# 移动Tomcat目录
$ mv apache-tomcat-8.5.47/ /opt/tomcat/
# 编写运行脚本，编写一个运行脚本，当启动容器时，运行该脚本，启动 Tomcat。
$ touch /root/run.sh
# 可能需要安装vim ， apt-get update 更新后，安装vim apt-get install -y vim
$ vi /root/run.sh 
# 输入以下内容
#!/bin/bash
export JAVA_HOME=/opt/jdk/
export PATH=$JAVA_HOME/bin:$PATH
sh /opt/tomcat/bin/catalina.sh run
# 为运行脚本添加执行权限：
$ chmod u+x /root/run.sh
# 退出容器，这时容器会停止
$ exit
 
```

### 保存成一个新的镜像

```shell
# 查看之前运行的容器
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                                         NAMES
b46376fca8de        ubuntu              "/bin/bash"              23 minutes ago      Exited (0) 7 seconds ago                                                  cocky_merkle
 
# 创建新的镜像
$ docker commit b46376fca8de gengzi:tomcat8
sha256:5d7b5992bbbecacd4413e6a6c29e9b7a2bc25faddac152eb9036e2ffd8c888f0
 
# 查看现在的镜像
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
gengzi              tomcat8             5d7b5992bbbe        28 seconds ago      549MB
 
现在我们可以使用 docker run gengzi:tomcat8 来启动容器
 
```

### 将web资源设置到容器的tomcat 的 webapps 下

```shell
# 新建 /mnt/webapps 目录，把我们的web资源放在这个目录中 比如 war 或者静态网站
$ mkdir -p /mnt/webapps
 
```

### 启动tomcat测试环境

```shell
# 启动gengzi:tomcat8 镜像容器
$ docker run -it -d   -p 8080:8080  -v /mnt/webapps:/opt/root/webapps/ --name tomcat8 gengzi:tomcat8 /root/run.sh
 
-d：表示以“守护模式”执行/root/run.sh脚本，此时 Tomcat 控制台不会出现在输出终端上。
-p：表示宿主机与容器的端口映射，此时将容器内部的 8080 端口映射为宿主机的 8080 端口，这样就向外界暴露了 8080 端口，可通过 Docker 网桥来访问容器内部的 8080 端口了。
-v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>
--name：表示容器名称，用一个有意义的名称命名即可。
 
# 查看当前运行的容器
$ docker ps
CONTAINER ID    IMAGE           COMMAND       CREATED           STATUS            PORTS        NAMES
8488e7e558d4   gengzi:tomcat8  "/root/run.sh" 14 seconds ago  Up 12 seconds       0.0.0.0:8080->8080/tcp                        tomcat8
 
# 测试使用curl  或者使用浏览器访问 http://服务器ip:8080
$ curl http://127.0.0.1:8080
```

### 停止tomcat容器

```shell
# 停止容器
$ docker stop tomcat8  # 或者使用容器id
# 如果不想要这个容器，可以删除容器
$ docker rm  容器id
 
```

 

 

 

 

##  特定的镜像安装使用

### 安装mysql 5.7

参考：[[如何通过Docker快速搭建测试环境？](https://www.cnblogs.com/hg-super-man/p/10908216.html)](https://www.cnblogs.com/hg-super-man/p/10908216.html)

[Docker 安装 MySQL](https://www.runoob.com/docker/docker-install-mysql.html)

[如何修改 docker 容器中 mysql 的端口号](https://hacpai.com/article/1575269751601)

[更改docker下的mysql容器的配置文件](https://blog.csdn.net/qq_40389276/article/details/102773081)

[linux下使用docker创建并进入mysql的容器](https://blog.csdn.net/qq_33359219/article/details/80980783)

[**在Docker中安装JDK**](http://www.zuidaima.com/blog/4656110816087040.htm)

```shell
# 使用 docker pull 下载镜像   https://hub.docker.com/_/mysql?tab=tags
$ docker pull mysql:5.7
# 查看镜像
$ $ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               5.7                 413be204e9c3        6 days ago          456MB
 
# 运行mysql容器
$ docker run -itd --name mysql-server1 -p 3307:3307 -e MYSQL_ROOT_PASSWORD=111 mysql:5.7
 
-p 3307:3307 ：映射容器服务的 3307 端口到宿主机的 3307 端口，外部主机可以直接通过 宿主机ip:3307 访问到 MySQL 的服务。
MYSQL_ROOT_PASSWORD=111：设置 MySQL 服务 root 用户的密码。
 
# 使用 docker ps 查看现在运行的容器信息
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
15c50777c383        mysql:5.7           "docker-entrypoint.s…"   5 seconds ago       Up 3 seconds        3306/tcp, 33060/tcp, 0.0.0.0:3307->3307/tcp   mysql-server1
 
# 将容器中的目录文件复制到宿主机中
mysql 配置文件；
数据存储目录，以便挂载(PS: 若不挂载到宿主机，每次启动容器数据都会丢失)
mysql 的日志logs
 
# 将容器中的 mysql 配置文件复制到宿主机中指定路径下，路径你可以根据需要，自行修改
$ mkdir -p /usr/local/docker/mysql/config
$ docker cp mysql-server1:/etc/mysql/mysql.conf.d/mysqld.cnf /usr/local/docker/mysql/config
# 将容器中的 mysql 存储目录复制到宿主机中
$ mkdir -p /usr/local/docker/mysql/data
$ docker cp mysql-server1:/var/lib/mysql/ /usr/local/docker/mysql/data
 
# 将容器中的 mysql log复制到宿主机中
$ mkdir -p /usr/local/docker/mysql/logs
$ docker cp mysql-server1:/logs /usr/local/docker/mysql/logs
 
 
# 将刚才的容器删除 PS: mysql-server 是我们运行容器时，指定的名称，当然，你也可以先执行 docker ps, 通过容器 ID 来删除。
$ docker rm -f mysql-server1
 
# 修改mysql 配置
$ vi /usr/local/docker/mysql/config/mysqld.cnf
# 修改端口号 设置在 mysqld 项目下新增 port=3307 配置
[mysqld]
port=3307
 
# 重新运行mysql 容器
$ docker run -itd --name mysql-server1 -p 3307:3307 -v /usr/local/docker/mysql/config/mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf -v /usr/local/docker/mysql/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=111 mysql:5.7
 
其他不变，额外添加了两个挂载子命令：
 
-v/usr/local/docker/mysql/config/mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf: 将容器中 /etc/mysql/mysql.conf.d/mysqld.cnf 配置文件挂载到宿主机的 /usr/local/docker/mysql/config/mysqld.cnf 文件上；
 
-v/usr/local/docker/mysql/data:/var/lib/mysql: 将容器中 /var/lib/mysql 数据目录挂载到宿主机的 /usr/local/docker/mysql/data 目录下；
 
-d: 后台运行容器
-p 将容器的端口映射到本机的端口
-v 将主机目录挂载到容器的目录
-e 设置参数
 
 
# 检查容器是否启动
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
15c50777c383        mysql:5.7           "docker-entrypoint.s…"   5 seconds ago       Up 3 seconds        3306/tcp, 33060/tcp, 0.0.0.0:3307->3307/tcp   mysql-server1
 
# mysql容器运行时，进入mysql
$ docker exec -it mysql-server1 bash
或者
docker exec -it 容器ID /bin/bash
 
# 如果修改配置文件
$ vim /etc/mysql/mysql.conf.d/mysqld.cnf
 
# 保存后，重新启动mysql容器
$ docker restart mysql-server1
或者 docker restart 容器id
 
# 进入mysql 容器的命令行
$ docker exec -it mysql-server1 bash
# 连接mysql
$ mysql -uroot -p
剩下的操作，就是mysql 提供的语法了
 
# 退出容器，不关闭容器
ctrl+p+q
 
例子 ： root@15c50777c383:/# mysql -uroot -p111
 
 
```

远程测试：

![1586169077885](../../../img/1586169077885.png)

 

 

 

 