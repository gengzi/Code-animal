[TOC]

## 目标

* 镜像基础命令

* 构建镜像的方式

 

## 镜像基础命令

```shell
# 列出镜像列表
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              14.04               90d5884b1ee0        5 days ago          188 MB
 
标签解释：
REPOSITORY  镜像的仓库源
TAG  镜像的标签
IMAGE ID  镜像的Id
CREATED  镜像的创建时间
SIZE  镜像大小
 
同一仓库源可以有多个 TAG，代表这个仓库源的不同个版本，如 ubuntu 仓库源里，有 15.10、14.04 等多个不同的版本，我们使用 REPOSITORY:TAG 来定义不同的镜像。
如果你不指定一个镜像的版本标签，例如你只使用 ubuntu，docker 将默认使用 ubuntu:latest 镜像。
 
比如指定使用 ubuntu:15.10
$ docker run -i -t ubuntu:15.10 /bin/bash
 
# 获取一个新的镜像
$ docker pull ubuntu:15.10
 
# 查找镜像 通过search 指令 可以查询 Docker Hub 上面的镜像 ， 可以返回25 条信息
$ docker search httpd
NAME  DESCRIPTION         STARS     OFFICIAL     AUTOMATED
httpd The Apache..         2941      [OK]       centos/httpd-24-centos7                
 
NAME: 镜像仓库源的名称
DESCRIPTION: 镜像的描述
OFFICIAL: 是否 docker 官方发布
stars: 类似 Github 里面的 star，表示点赞、喜欢的意思
AUTOMATED: 自动构建
 
# 删除镜像
$ docker rmi 镜像名称
 
```

## 创建镜像

当我们从 docker 镜像仓库中下载的镜像不能满足我们的需求时，我们可以通过以下两种方式对镜像进行更改。

* 从已经创建的容器中更新镜像，并且提交这个镜像
* 使用 Dockerfile 指令来创建一个新的镜像

### 从已经创建的容器更新镜像，并提交这个镜像

```shell
# 准备好一个容器
$ docker run  --name web1 -i -t   ubuntu /bin/bash
# 更新镜像
$ apt-get update
# 退出
$ exit
# 提交镜像
docker commit -m="update images" -a="gengzi" e218edb10161  gengzi/ubuntu:v1
 
参数解释：
-m 提交的描述信息
-a 作者信息
e218edb10161  容器的id
gengzi/ubuntu:v1  创建的镜像名称和标签（tag）
 
# 查看当前的镜像
$ docker images
 
# 启动新镜像
$ docker run -i -t gengzi/ubuntu:v1  /bin/bash
 
 
```

 

### 使用 Dockerfile 指令来创建一个新的镜像

我们使用命令 **docker build** ， 从零开始来创建一个新的镜像。为此，我们需要创建一个 Dockerfile 文件，其中包含一组指令来告诉 Docker 如何构建我们的镜像。

```shell
# 创建Dockerfile  文件
mkdir -p /home/dockerfile
cd /home/dockerfile
vi Dockerfile
输入
FROM    centos:6.7
MAINTAINER      Fisher "gengzi@qq.com"
RUN     /bin/echo  "hello world"
 
每一个指令都会在镜像上创建一个新的层，每一个指令的前缀都必须是大写的。
第一条FROM，指定使用哪个镜像源
RUN 指令告诉docker 在镜像内执行命令，安装了什么。。。
然后，我们使用 Dockerfile 文件，通过 docker build 命令来构建一个镜像。


 
# 使用docker build 构建镜像
$ docker build -t gengzi/centos:6.7 .
 
----------------- 执行过程 ---------------------------------
Sending build context to Docker daemon 3.584 kB
Step 1 : FROM centos:6.7
6.7: Pulling from library/centos
 
cbddbc0189a0: Pull complete
Digest: sha256:4c952fc7d30ed134109c769387313ab864711d1bd8b4660017f9d27243622df1
Status: Downloaded newer image for centos:6.7
---> 9f1de3c6ad53
Step 2 : MAINTAINER Fisher "gengzi@qq.com"
---> Running in b9f024efcf6d
---> d38b2abac6a2
Removing intermediate container b9f024efcf6d
Step 3 : RUN /bin/echo  "hello world"
---> Running in 0111b30d6693
hello world
---> e962a020a72c
Removing intermediate container 0111b30d6693
Successfully built e962a020a72c
SECURITY WARNING: You are building a Docker image from Windows against a non-Windows Docker host. All files and directories added to build context will have '-rwxr-xr-x' permissions. It is recommended to double check and reset permissions for sensitive files and directories.
 
------------------------------------------------------------------------------------
 
-t ：指定要创建的目标镜像名
. ：Dockerfile 文件所在目录，可以指定Dockerfile 的绝对路径
 
# 查看现在的镜像
$ docker images
 
# 为镜像设置一个新的标签
$ docker tag 860c279d2fec gengzi/centos:dev
 
docker tag 镜像ID，这里是 860c279d2fec ,用户名称、镜像源名(repository name)和新的标签名(tag)。
使用 docker images 命令可以看到，ID为860c279d2fec的镜像多一个标签。
 
 
```

 
