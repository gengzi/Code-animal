[TOC]


## 目标

* 了解docker 容器连接

* 实现tomcat容器，连接mysql容器

 

## Docker 容器连接

###  容器和宿主机的网络映射

```shell
# 容器启动后，外部需要访问宿主机访问到启动的容器
# 这就需要容器的端口映射到宿主机的端口上
 
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
 
例子：
# 随机映射
$ docker run -p 80 --name web1 -it  ubuntu /bin/bash
# 查看宿主机的端口
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                   NAMES
cdba03583728        ubuntu              "/bin/bash"         29 minutes ago      Up 29 minutes       0.0.0.0:32770->80/tcp   web1
 
# 宿主机的端口为 32770 ，所以外部访问这个容器服务的端口，就是 32770
 
# 默认都是绑定 tcp 端口，如果要绑定 UDP 端口，可以在端口后面加上 /udp
$ docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py
 
# 查看端口绑定情况
$ docker port  容器id  或者 容器名称
 
```

## Docker 容器互联

端口映射并不是唯一把 docker 连接到另一个容器的方法。

docker 有一个连接系统允许将多个容器连接在一起，共享连接信息。

docker 连接会创建一个父子关系，其中父容器可以看到子容器的信息。
###  Docker 容器的网络基础

```shell
# 使用ifconfig 查看现在的网络信息
docker0   Link encap:Ethernet  HWaddr 02:42:15:07:29:29
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:664113 errors:0 dropped:0 overruns:0 frame:0
          TX packets:610550 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:302568088 (302.5 MB)  TX bytes:753824854 (753.8 MB)
# docker0 由Docker 的守护进程创建，docker 0 是 linux 的虚拟网桥
# 网桥是数据链路层的设备，linux 的虚拟网桥可以设置ip地址，相当于拥有一个隐藏的虚拟网卡
 
# docker0 的地址划分：
ip 172.17.0.1   子网掩码：255.255.0.0
Mac 02:42:15:07:29:29   到 02:42:15:07:ff:ff
总共提供了 65534 个地址
 
# 查看网桥
$ apt-get install -y  bridge-utils
# 查看现在的网络设备
$ brctl show
 
bridge name    bridge id        STP enabled    interfaces
br-85365d777273        8000.0242111e2a33    no    
docker0        8000.024215072929    no        veth0daa9f8
                            veth1d53574
                            veth5cb2085
                            vetha01fa23
                            vethb9abf04
                            vethd74715b
# 自定义docker0 设置它为我们希望使用的网段
$ ifconfig docker0 192.168.200.1 netmask 255.255.0.0
# 重启docker 服务
$ service docker restart
 
# 有时候我们不希望使用docker0 这个虚拟网桥
# 添加虚拟网桥
$ brctl addbr br0
$ ifconfig  br0 192.168.100.1 netmask 255.255.0.0
# 更改docker 守护进程的启动配置
$ vi /etc/default/docker
# 添加 DOCKER_OPS 的值
-b=br0
 
```

### Docker 容器互联 - 允许所有容器互联

```shell
# 在默认情况下，docker 允许所有容器的互联(--icc=true)，但是在重启容器后，ip地址是会发生变化的。
# 我们会发现容器提供的ip地址是不可靠的，为了避免这种情况，docker 使用 --link 指定连接到那个容器
 
# -- link
docker run --link=[CONTAINER_NAME]:[ALIAS] [IMAGE] [COMMAND]
参数解释：
CONTAINER_NAME: 需要连接的容器名字
ALIAS: 在容器中连接的代号
 
# 例子：
docker -run -it --link=container01:webtest nginx
ping webtest
 
查看在容器中产生的哪些影响
 
$ env
查看环境变量，可以看到大量以WEBTEST*开头的环境变量，这些环境变量是在容器启动时，由docker添加的。我们还可以查看在/ect/host文件，这里添加了webtest的地址映射。当docker重启启动容器时 /ect/host所对应的ip地址发生了变化。也就是说，针对于指定了link选项的容器，在启动时docker会自动修改ip地址和我们指定的别名之间的映射。环境变量也会做出相应的改变。
 
```

### Docker  容器互联 - 拒绝所有容器间互联

```shell
Docker守护进程的启动选项
--icc=false
修改vim /etc/default/docker，在末尾添加配置 DOCKER_OPTS="--icc=false"。
需要重启docker的服务 sudo service docker restart.即使是link也ping不通。
 
```

### Docker  容器互联 -允许特定容器间的连接

```shell
Docker守护进程的启动选项
--icc=false --iptables=true
--link 在容器启动时添加link
docker利用iptables中的机制，在icc=false时，阻断所有的docker容器间的访问，仅仅运行利用link选项配置的容器进行相互的访问。
注: 如果出现ping不通的情况，可能为iptables的问题(DROP规则在docker之前了)。
# 现将 iptables 清除后，再运行docker 服务
sudo iptables -L -n    查看iptables规则的情况
sudo iptables -F    清空iptables规则设置
sudo service docker restart 重新启用docker的服务
sudo iptables -L -n 再来查看iptables的设置，docker的规则链已经在第一位
 
```

 

## Docker 容器与外部网络连接

- ip_forward

　　　　--ip-forward=true

```
sysctl net.ipv4.conf.all.forwarding
```

ip_forward本身是Linux系统中的一个变量，它的值决定了系统是否会转发流量。在Docker守护进程的默认参数中也有ip_forward选项，默认值是true.

- iptables

iptables是与linux内核集成的包过滤防火墙系统，几乎所有的linux发行版本都会包含iptables的功能。

![img](https://images2015.cnblogs.com/blog/638965/201706/638965-20170615151148618-455735692.png)

每一个大的方块中，就是iptables中的一个链(chain)，每一个链实际上就是数据处理中的一个环节，而在每个环节中又包含了不同的操作。

 

- 允许端口映射访问

- 限制IP访问容器

实质都是通过iptables的规则来控制的。

 

 

  