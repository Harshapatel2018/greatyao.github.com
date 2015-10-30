---
layout: post
title: 使用DevStack搭建OpenStack Liberty版本的Magnum
tagline: null
category: null
tags: []
published: true

---
10月15日，OpenStack发布了Liberty版本，其中一个变化就是容器服务Magnum正式推出1.0版本，只是各大Linux发行版还没有发布magnum的安装包，因此需要从源码直接构建。下面给出自己的基于Devstack的方式的完整安装实践过程。

环境准备 

在VMware Workstations建台虚拟机，Ubuntu 14.04 LTS，CPU两个核以上，内存至少4G以上，硬盘100G，两个网卡（一个主机模式，另一个NAT模式）。

vi /etc/network/interfaces
##主机模式，用于openstack内部模块通信 
auto eth0
iface eth0 inet static
        address 20.0.0.11
        gateway 20.0.0.1
        netmask 255.255.255.0

#外部网络，虚拟机访问外网的出口
auto eth1
iface eth1 inet static
        address 172.24.54.222
        gateway 172.24.54.2
        netmask 255.255.255.0

#将缺省路由设置为网卡2
up route add default gw 172.24.54.2 dev eth1

部署过程
1.安装git
$ sudo su
# apt-get install git

2.下载devstack源码
# cd /home
# git clone https://github.com/openstack-dev/devstack 

3.创建stack用户运行
# cd /home/devstack/tools/
# ./create-stack-user.sh

4.修改devstack目录权限,让stack用户可以运行
# chown -R stack:stack /home/devstack

5.切换到stack用户，进入devstack目录下，创建localrc文件
# su stack
$ cd /home/devstack
$ cat > localrc << END
[[local|localrc]]
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_TOKEN=password
SERVICE_PASSWORD=password
ADMIN_PASSWORD=password
PUBLIC_INTERFACE=eth1
enable_plugin magnum https://git.openstack.org/openstack/magnum
enable_plugin barbican https://git.openstack.org/openstack/barbican
VOLUME_BACKING_FILE_SIZE=20G
HOST_IP=20.0.0.11
END


6. 运行Devstack
$ ./stack.sh

注意：使用的是stack用户运行。在安装过程中，可能会提示apt-get下载源错误，重复执行上述安装命令；其他情况，可以再次执行安装命令。

$  ./unstack.sh && ./stack.sh
