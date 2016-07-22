---
layout: post
title: 搭建Docker Swarm集群
tags: docker,swarm,shipyard
tagline: null
keywords: Docker
categories: [Docker, Swarm]
group: archive
icon: leaf
---

{% include codepiano/setup %}


官网上给出了几种Docker Swarm集群的部署方法，分别是基于[Docker ToolBox][1] 以及 [Amazon AWS][2] 。

[Shipyard][3] 是一个非常牛逼的docker集群管理系统，而且有着非常友好的Web界面。更为变态的是Shipyard有一个比较牛逼的一键自动部署脚本，不过里面的服务发现与注册使用的是 etcd 而不是 consul 的，但是 consul 是 docker 官网推荐的，本文主要介绍使用shipyard以及 consul 来手工部署 Swarm 集群。


# 环境准备

实验环境为三台服务器，各个服务器的角色、配置如下表所示：

| IP |操作系统 | 角色              |
| ------------ | ----------------- | ------------------- |
| 192.168.0.47 | Ubuntu 14.04 内核3.16.0 | consul 节点          |
| 192.168.0.56 | Ubuntu 14.04 内核3.16.0 | swarm manager节点 |
| 192.168.0.57 | Ubuntu 14.04 内核3.16.0 | swarm agent 节点  |


# 安装Docker

官方提供了各个平台的docker[安装手册][4]， 读者可以参考 。为了偷懒方便，在这里我们使用了Docker公司提供的一键安装脚本来进行docker的安装

    curl -sSL https://get.docker.io/ubuntu/ | sudo sh

安装完毕，我们还需要修改 docker 的默认启动参数， 修改下面的文件:

    vi /etc/default/docker

添加这么一行:

     DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"

然后重新启动Docker服务

    service start docker


# 部署Consul节点

在192.168.0.47也就是consul节点上，运行如下命令来部署服务发现模块

    docker run -d \
        -p 8300:8300 \
        -p 8301:8301 \
        -p 8301:8301/udp \
        -p 8302:8302 \
        -p 8302:8302/udp \
        -p 8400:8400 \
        -p 8500:8500 \
        -p 8600:53 \
        -p 8600:53/udp \
        --restart=always \
        --name=consul progrium/consul -server -bootstrap -ui-dir=/ui -advertise 192.168.0.47 -client 0.0.0.0  

解释下各个参数:

 *   -d 容器在后台运行, detached mode
 *   --restart=always 重启模式, always 表示永远
 *  -p 8400:8400 映射 consul的 rpc 端口8400
 *  -p 8500:8500 映射到公共 IP 这样方便我们使用 UI 界面
 *  -p 53:53/udp 绑定udp 端口53(默认 DNS端口)在 docker0 bridge 地址上
 *  -advertise 192.168.0.47 consul服务对外公布的 IP, 这里特意设置为0.47, 否则 service 会显示为内部的容器的 IP 地址, 这样就访问不到了
 *  -client 0.0.0.0 consul 监听的地址


# 部署Swarm Manager节点

## Step 1
先在192.168.0.56即swarm manager节点上安装 rethinkdb 数据库

    docker run -d --restart=always --name shipyard-rethinkdb rethinkdb

## Step 2
然后继续安装 swarm manager，**需要注意的是将192.168.0.47替换为实际上的 consul 节点的 ip地址**

    docker run -d -p 3375:3375 --restart=always --name shipyard-swarm-manager swarm:latest manage --host tcp://0.0.0.0:3375 consul://192.168.0.47:8500
    
## Step 3
接着在manager节点上安装 swarm agent，这里需要注意的是192.168.0.47是consul节点的IP地址，192.168.0.56是manager的IP地址，将两者替换为你实际环境中的节点地址

    docker run -d --restart=always --name shipyard-swarm-agent swarm:latest join --addr 192.168.0.56:2375 consul://192.168.0.47:8500

## Step 4
最后部署 shipyard 管理模块

    docker run -d --restart=always --name shipyard-controller --link shipyard-rethinkdb:rethinkdb --link shipyard-swarm-manager:swarm -p 9090:8080 shipyard/shipyard:latest    server -d tcp://swarm:3375

部署完毕后可以在浏览器访问http://192.168.0.56:9090 , 就能看到 shipyard 的登录页面， 默认账户是 admin，密码shipyard。


# 部署Swam Agent节点
节点agent的部署比较简单，只需要参考manager节点上的步骤3

    docker run -d --restart=always --name shipyard-swarm-agent swarm:latest join --addr 192.168.0.56:2375 consul://192.168.0.47:8500

**类似的，192.168.0.47是consul节点的IP地址，192.168.0.57是agent节点的IP地址，将两者替换为你实际环境中的节点地址**

# 部署Overlay网络

为了能够在Docker Swarm集群上使用跨主机Overlay网络，还需要做一些配置修改。在manager节点和agent节点上，

    vi /etc/default/docker

将这一行

     DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"

修改为

    DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=consul://192.168.0.47:8500 --cluster-advertise=eth0:2375"
    
这里192.168.0.47为consul节点的IP地址，eth0为manager/agent节点上192.168.0.56/57所在的网卡名称。然后重新启动Docker服务

    service start docker
    
下面创建一个overlay网络

    docker -H 192.168.0.56:3375 network create -d overlay test-net
 
 查看swarm集群中所有的网络
 
     $ docker -H 192.168.0.56:3375 network ls
     NETWORK ID          NAME                    DRIVER
     1abc3888c68f        node1/bridge            bridge              
     7c21108c289f        node1/docker_gwbridge   bridge              
     fd7873463f44        node1/host              host                
     61808ed752b3        node1/none              null                
     3239dd516cb5        node2/bridge            bridge              
     049b996b6ec0        node2/docker_gwbridge   bridge              
     045824d9569c        node2/host              host                
     2a0ec5e15ce8        node2/none              null                
     33df13f850c5        test-net                overlay

可以看到test-net网络同时在56和57上创建，下面我们在此网络上创建容器

    $ docker -H 192.168.0.56:3375 run -itd --net test-net --name test4 ubuntu:14.04 
    db7113bae6d06922a133cef92f01701c0f35156b374ae14e31e13ac54500e979
    $ docker -H 192.168.0.56:3375 run -itd --net test-net --name test5 ubuntu:14.04 
    dbea32bd623ce225af426470f1f30f5f76d5f5148a63893e2f9f84b4628cb171

看到两个容器分别分布在两个节点上

    docker -H 192.168.0.56:3375 ps -a
    CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                   NAMES
    dbea32bd623c        ubuntu:14.04               "/bin/bash"              2 minutes ago       Up About a minute                                           node1/test5
    db7113bae6d0        ubuntu:14.04               "/bin/bash"              2 minutes ago       Up 2 minutes                                                node2/test4
    da671da259ce        swarm:latest               "/swarm join --addr 1"   2 hours ago         Up 2 hours          2375/tcp                                node2/shipyard-swarm-agent
    4d9725a9f0de        shipyard/shipyard:latest   "/bin/controller serv"   2 hours ago         Up 2 hours          192.168.0.56:9090->8080/tcp             node1/shipyard-controller
    d994169e39c7        rethinkdb                  "rethinkdb --bind all"   2 hours ago         Up 2 hours          8080/tcp, 28015/tcp, 29015/tcp          node1/shipyard-rethinkdb,node1/shipyard-controller/rethinkdb
    5f5b793a358d        swarm:latest               "/swarm join --addr 1"   2 hours ago         Up 2 hours          2375/tcp                                node1/shipyard-swarm-agent
    7f20411fc504        swarm:latest               "/swarm manage --host"   2 hours ago         Up 2 hours          2375/tcp, 192.168.0.56:3375->3375/tcp   node1/shipyard-swarm-manager,node1/shipyard-controller/swarm

下面我们测试跨主机网络的连通性，我们通过attach到test4容器上ping test5来验证分散在不同节点上的容器的网络连通性

    $ docker -H 192.168.0.56:3375 attach test4
    root@db7113bae6d0:/# 
    root@db7113bae6d0:/# ping -c 4 test5
    PING test5 (10.0.0.8) 56(84) bytes of data.
    64 bytes from test5.test-net (10.0.0.8): icmp_seq=1 ttl=64 time=0.283 ms
    64 bytes from test5.test-net (10.0.0.8): icmp_seq=2 ttl=64 time=0.285 ms
    64 bytes from test5.test-net (10.0.0.8): icmp_seq=3 ttl=64 time=0.295 ms
    64 bytes from test5.test-net (10.0.0.8): icmp_seq=4 ttl=64 time=0.221 ms

    --- test5 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 2998ms
    rtt min/avg/max/mdev = 0.221/0.271/0.295/0.029 ms

----
最后欣赏shipyard两张截图

![image](/assets/post-images/shipyard.png)

![image](/assets/post-images/shipyard-2.png)

  [1]: https://docs.docker.com/swarm/install-w-machine
  [2]: https://docs.docker.com/swarm/install-manual/
  [3]: http://www.shipyard-project.com
  [4]: https://docs.docker.com/engine/installation/linux/
