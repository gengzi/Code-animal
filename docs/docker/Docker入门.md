

[TOC]

## 目标

* 了解docker

* docker 包含什么

  参考：[Docker 教程](https://www.runoob.com/docker/docker-tutorial.html)

  [docker视频](https://www.php.cn/code/8787.html)

## 了解docker

* 什么是容器？

  一种虚拟化的方案

  操作系统级别的虚拟化

  docker 的容器技术是虚拟化的一种。

  跟虚拟机进行对比，docker容器不会包含操作系统

![JIDB4I.png](https://s1.ax1x.com/2020/04/28/JIDB4I.png)


* 什么是docker

  将应用程序自动部署到容器

* 目标

  提供简单轻量的建模方式

  职责的逻辑分离

  快速高效的开发生命周期 ，开发测试生产 都是用一套环境

  鼓励使用面向服务的架构

* 使用场景

  使用docker容器开发、测试、部署服务

  创建隔离的运行环境

  搭建测试环境

  在服务型环境中部署和调整数据库或其他的后台应用

  从头编译或者扩展现有的 OpenShift 或 Cloud Foundry 平台来搭建自己的 PaaS 环境

## docker 基本组成

* docker client 客户端

* docker Daemon 守护进程

      c/s 架构

* Docker Image 镜像

  容器的基石

  层叠的只读文件系统

  [![JIrEad.png](https://s1.ax1x.com/2020/04/28/JIrEad.png)](https://imgchr.com/i/JIrEad)



  联合加载：一次加载多个文件系统

* Docker Container 容器

  容器通过镜像来启动

  程序在可读写层加载进行

  写时复制

[![JIrNR0.png](https://s1.ax1x.com/2020/04/28/JIrNR0.png)](https://imgchr.com/i/JIrNR0)

* Docker Registry 仓库

  公有 ： Docker hub

  私有 ： 自己搭建的仓库

## docker 操作流程

[![JIrdMT.png](https://s1.ax1x.com/2020/04/28/JIrdMT.png)](https://imgchr.com/i/JIrdMT)




Docker client 客户端通过命令行或者其他工具使用 Docker SDK (https://docs.docker.com/develop/sdk/) 与 Docker 的守护进程通信，发起一些命令来操作docker contalner 容器。

Docker Host 是一个物理或者虚拟的机器用于执行 Docker 守护进程和容器。
Docker Daemon 是docker 的守护进程

Docker Registry 是 Docker 仓库用来保存镜像，可以理解为代码控制中的代码仓库。

* Docker Hub([https://hub.docker.com](https://hub.docker.com/)) 提供了庞大的镜像集合供使用。

  一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。

  通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 **<仓库名>:<标签>** 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 **latest** 作为默认标签。

  如  ubuntu:14.0.4   指定ubuntu的14.0.4 这个版本

 

Docker Images 是 docker 镜像用于创建 Docker 容器的模板，比如 Ubuntu 系统。
