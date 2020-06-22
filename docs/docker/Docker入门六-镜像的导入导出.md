[TOC]

## 目标

* 了解docker镜像导入导出

  参考：[Docker镜像的导入导出](https://blog.csdn.net/ncdx111/article/details/79878098)

  [[Docker 本地导入镜像/保存镜像/载入镜像/删除镜像](https://www.cnblogs.com/linjiqin/p/8604756.html)](cnblogs.com/linjiqin/p/8604756.html)

## docker 镜像导出和导入（export import）

```shell
# 容器导出
$ docker export [options] container
$ docker export -o 目标文件.tar 容器名称
其中-o表示输出到文件
$ docker export -o  gengzitomcat.tar tomcat8
 
例子:gengzitomcat.tar为目标文件，tomcat8是源容器名（name）
在当前目录下，可以看到
-rw-------  1 root   root   553198592 Apr  6 21:25 gengzitomcat.tar
 
# 容器镜像导入
$ docker import [options] file|URL|- [REPOSITORY[:TAG]]
 
例子:
$ docker import gengzitomcat.tar gengzitomcat:v1
或
$ cat gengzitomcat.tar | docker import - gengzitomcat:v1
 
# 查看镜像
$ docker images
 
可以看到导入完成后，docker为我们生成了一个镜像ID，使用docker images也可以看到我们刚刚从本地导入的镜像。
 
```

## docker 镜像导出和导入（save  load ）

```shell
# 导出
$ docker save [options] images [images...]
$ docker save -o nginx.tar nginx:latest
或
$ docker save > nginx.tar nginx:latest
其中-o和>表示输出到文件，nginx.tar为目标文件，nginx:latest是源镜像名（name:tag）
 
# 导入
$ docker load [options]
示例
$ docker load -i nginx.tar
或
$ docker load < nginx.tar
其中-i和<表示从文件输入。会成功导入镜像及相关元数据，包括tag信息
 
 
```

## 区别：

* export命令导出的tar文件略小于save命令导出的

* export命令是从容器（container）中导出tar文件，而save命令则是从镜像（images）中导出

* 基于第二点，export导出的文件再import回去时，无法保留镜像所有历史（即每一层layer信息，不熟悉的可以去看Dockerfile），不能进行回滚操作；而save是依据镜像来的，所以导入时可以完整保留下每一层layer信息。如下图所示，nginx:latest是save导出load导入的，nginx:imp是export导出import导入的。

 

## 建议

可以依据具体使用场景来选择命令

- 若是只想备份images，使用save、load即可
- 若是在启动容器后，容器内容有变化，需要备份，则使用export、import