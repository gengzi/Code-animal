[TOC]


## 目标

* docker  在ubuntu 安装

* docker  在windos 安装

 

## ubuntu 安装docker

参考 ：[Ubuntu Docker 安装](https://www.runoob.com/docker/ubuntu-docker-install.html)
http://get.daocloud.io/#install-docker-for-mac-windows  国内下载地址

https://www.jianshu.com/p/ae728f8d7636


```shell
sudo apt-get update  # 更新apt索引
```

## windows 安装 docker

参考：[Windows Docker 安装](https://www.runoob.com/docker/windows-docker-install.html)


## 修改镜像源
参考: [Docker镜像源修改]( https://blog.csdn.net/jixuju/article/details/80158493 )
由于国内网络原因，我们可以修改镜像源来提升下载镜像速度
```shell
# linux 修改或新增 /etc/docker/daemon.json
$ vi /etc/docker/daemon.json
{
"registry-mirrors": ["http://hub-mirror.c.163.com"]
}
# 重启docker 服务
$ service docker restart
 
# windos 如下图
 
# Docker国内源说明：
Docker 官方中国区
https://registry.docker-cn.com
网易
http://hub-mirror.c.163.com
中国科技大学
https://docker.mirrors.ustc.edu.cn
阿里云
https://pee6w651.mirror.aliyuncs.com
 
```

![1587394323098](index_files/1587394323098.png)
