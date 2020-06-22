[TOC]

## 目标

* docker 的基本使用命令

* docker 部署一个静态网站

* 守护式容器

  参考：[Docker 容器使用](https://www.runoob.com/docker/docker-container-usage.html)

## Docker 的基本使用命令

* 运行一个docker 容器

  ```shell
  ## 检查docker 版本
  $ docker version
 
  Client:   # 客户端
   Version:           18.09.7               #版本
   API version:       1.39
   Go version:        go1.10.4               # go语言版本
   Git commit:        2d0083d
   Built:             Fri Aug 16 14:19:38 2019
   OS/Arch:           linux/amd64
   Experimental:      false
 
  Server:  # 服务端
   Engine:
    Version:          18.09.7
    API version:      1.39 (minimum version 1.12)
    Go version:       go1.10.4
    Git commit:       2d0083d
    Built:            Thu Aug 15 15:12:41 2019
    OS/Arch:          linux/amd64
    Experimental:     false
 
    ## 运行一个应用程序
    $ docker run ubuntu:15.10 /bin/echo "Hello world"
    ## 输出
    Hello world
 
      - docker: Docker 的二进制执行文件。
      - run: 与前面的 docker 组合来运行一个容器。
      -  ubuntu:15.10 指定要运行的镜像，Docker 首先从本地主机上查找镜像是否存在，如果不存        在，  Docker 就会从镜像仓库 Docker Hub 下载公共镜像。
      - /bin/echo "Hello world": 在启动的容器里执行的命令
  ```

   注意： 首次运行可能会提示 找不到这个镜像 ubuntu:15.10 (Unable to find image 'ubuntu:15.10' locally)等待下载即可

 

  这个容器启动后，就不能再接受你的命令操作了。

* 启动交互式的容器

  我们通过 docker 的两个参数 -i -t，让 docker 运行的容器实现**"对话"**的能力：

  ```shell
  $ docker run -i -t ubuntu:15.10 /bin/bash
  root@0123ce188bd8:/#
 
  ## 或者连写  -it
 
  -t: 在新容器内指定一个伪终端或终端。
  -i: 允许你对容器内的标准输入 (STDIN) 进行交互。
 
  注意第二行 root@0123ce188bd8:/#，此时我们已进入一个 ubuntu15.10 系统的容器
 
  ## 输入 exit  或者 ctrl+d 来退出容器
  root@0123ce188bd8:/#  exit
  exit
  root@runoob:~#     表示已经退出了
  ```

* 启动后台模式的容器

  ```shell
  $ docker run -d ubuntu:15.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"
  2b1b7a428627c51ab8810d541d759f072b4fc75487eed05812646b8534a2fe63
 
  在输出中，我们没有看到期望的 "hello world"，而是一串长字符
  2b1b7a428627c51ab8810d541d759f072b4fc75487eed05812646b8534a2fe63
  这个长字符串叫做容器 ID，对每个容器来说都是唯一的，我们可以通过容器 ID 来查看对应的容器发生了什么。
 
  $ docker ps  查看运行中的容器
  root@VM-56-19-ubuntu:/home/ubuntu# docker ps
  CONTAINER ID        IMAGE               COMMAND             CREATED          
  STATUS              PORTS                   NAMES
  a15c4b7fdbef        ubuntu              "/bin/bash"         2 days ago          Up 47 hours         0.0.0.0:32769->80/tcp   web
 
  输出详情介绍：
 
  CONTAINER ID: 容器 ID。
  IMAGE: 使用的镜像。
  COMMAND: 启动容器时运行的命令。
  CREATED: 容器的创建时间。
  STATUS: 容器状态。
 
  状态有7种：
  created（已创建）
  restarting（重启中）
  running（运行中）
  removing（迁移中）
  paused（暂停）
  exited（停止）
  dead（死亡）
 
  PORTS: 容器的端口信息和使用的连接类型（tcp\udp）。
  NAMES: 自动分配的容器名称。
 
  # 查看容器内的标准输出
  $ docker logs 容器名称 或者 容器id
  例：
  $ docker logs a15c4b7fdbef
 
  # 停止容器
  $ docker stop 容器名称 或者 容器id
  ```

 ### 其他命令

  ```shell
 # 获取镜像,当我们本地没有的时候
$ docker pull ubuntu
# 查看所有的容器
$ dcoker  ps -a 
# 启动一个停止的容器
$ docker start 容器名称 或者 容器id
# 后台运行 ,加一个 -d 后台运行
$ docker run -d 
# 停止一个容器
$ docker stop 容器名称 或者 容器id
# 重新启动一个容器
$ docker  restart 容器名称 或者 容器id
# 在使用 -d 参数时，容器启动后会进入后台。此时想要进入容器，可以通过以下指令进入： 推荐大家使用 docker exec 命令，因为此退出容器终端，不会导致容器的停止。
$ docker  exec 容器名称 或者 容器id
# 删除容器
$ docker rm -f 容器名称 或者 容器id
# 下面的命令可以清理掉所有处于终止状态的容器
$ docker container prune
 
# 设置容器的端口映射
通过run命令的两个方式 一个是  [-P] 大写的P   [-p] 小写的p
 
-P 大写的P ，会将容器的所有端口进行映射
 
-p 小写的p ，可以指定容器那些端口可以被映射
四种方式：
    containerPort：
    只指定容器端口，这种情况下，宿主机的端口是随机映射的
    docker run -p 80 -i -t ubuntu /bin/bash
    hostPort:containerPort
    同时指定容器端口和宿主机的端口，这种情况下，宿主机的端口和容器端口是对应的
    docker run -p 8080:80 -i -t ubuntu /bin/bash
    ip::containerPort
    指定ip和容器的端口
    docker run  -p 0.0.0.0:80  -i -t ubuntu /bin/bash
    up:hostPort:containerPort
    指定ip和宿主机的端口和容器的端口
    docker run -p 0.0.0.0:8080:80    -i -t ubuntu /bin/bash
 
 
 
 
  ```

## 使用docker 部署一个静态网站

我们想在容器中启动一个nginx来跳转到我们的一个静态页面上

需要的镜像 ： ubuntu

步骤：

* 启动一个交互式的容器, 并且指定容器的端口映射为80
* 安装nginx
* 安装vim
* 创建静态页页面
* 修改nginx的配置文件
* 运行nginx
* 验证网站访问

```shell
# 启动一个交互式的容器, 并且指定容器的端口映射为80
# 这里使用了 -p 来指定容器映射 80 端口，但是宿主机的端口是随机的
# --name 指定这个容器的名称是 web1
$ docker run -p 80 --name web1 -it  ubuntu /bin/bash
# 进入交互式界面，使用 apt-get 命令安装 nginx
$ apt-get  update  #先更新下apt索引
$ apt-get install -y nginx  #安装nginx
$ apt-get install -y vim  #安装vim
# 创建一个静态的html页面
$ mkdir  -p /var/www/html 
$ cd /var/www/html 
$ vim ./index.html
插入一下内容
<!DOCTYPE html>
<html lang="en">
  <head>
  </head>
  <body>
    <h1>hello world！！！</h1>
  </body>
</html>
# 修改nginx的配置文件
$ whereis nginx   #查找一下nginx
$ vim /etc/nginx/sites-enabled/default
## 将 root 的目录修改为  /var/www/html
## 运行nginx
$ nginx
$ ps -ef  #查看nginx 是否运行
## 使用ctrl+q 来实现退出  交互式命令行
# 使用docker ps 查看容器
$ docker ps
root@VM-56-19-ubuntu:/home/ubuntu# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                   NAMES
cdba03583728        ubuntu              "/bin/bash"         29 minutes ago      Up 29 minutes       0.0.0.0:32770->80/tcp   web1
## 我们会发现容器的 80 端口映射到宿主机的32770端口
## 查看网站是否访问了
$ curl http://127.0.0.1:32770
## 也可以使用容器的ip 来访问
# 使用 docker inspect 命令查看容器的信息
$ docker inspect  web1
出现：
"IPAddress": "172.17.0.2",
 
## 使用容器的ip来访问
$ curl http://172.17.0.2:80
 
```
