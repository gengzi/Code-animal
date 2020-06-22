[TOC]


## 目标

* docker  在ubuntu 安装

* docker  在windos 安装

 

## ubuntu 安装docker

参考 ：[Ubuntu Docker 安装](https://www.runoob.com/docker/ubuntu-docker-install.html)
http://get.daocloud.io/#install-docker-for-mac-windows  国内下载地址

https://www.jianshu.com/p/ae728f8d7636

**特别说明**：

windows中的提供的linux子系统(WLS)，最好不要安装docker环境，因为WLS,还并不能算真正的linux。安装后可能会导致一系列的问题。

（别问我怎么知道的，我花了一天的时间，在本地的WLS上安装并运行docker环境，出现了很多问题。）

[官方的docker ubuntu系统安装文档](https://docs.docker.com/engine/install/ubuntu/)

ubuntu 安装注意几点：

- 查看 ubuntu 的系统版本是否能支持 docker

  ```
  # 官方文档有说明
  Ubuntu Eoan 19.10
  Ubuntu Bionic 18.04（LTS）
  Ubuntu Xenial 16.04（LTS）
  ```

- 使用apt-get 下载软件包，最好修改成国内的镜像源

  [修改ubuntu18.04 apt源为阿里云(aliyun)镜像](https://www.cnblogs.com/Eric-Shenblog/p/10862027.html)

- 下载docker 的镜像源，最好也修改成国内的

  ```shell
  # 添加docker 镜像源
  $ sudo add-apt-repository deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable
  ```

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

[![JIrrdJ.png](https://s1.ax1x.com/2020/04/28/JIrrdJ.png)](https://imgchr.com/i/JIrrdJ)



