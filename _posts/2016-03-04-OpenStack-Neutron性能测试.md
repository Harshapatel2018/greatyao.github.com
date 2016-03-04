---
layout: post
title: OpenStack Neutron性能测试
tagline: null
category: null
tags: []
published: true

---
# 环境
OpenStack硬件环境包括：一台控制节点、一台网络节点、四台计算节点（总计104核 、326G内存、4.8T硬盘）。所有的网络环境都为1G。

OpenStack网络模块为neutron+ovs，本次测试主要测在大量虚拟机并发环境下东西流量和南北流量对网络的负载情况，性能测试所用的工具为iperf2和iperf3，以虚拟机两两之间的带宽速率作为评价指标。

本次测试环境创建的虚拟机操作系统为ubuntu14.04 x86_64，规格配置为cpu1核，内存1G，硬盘10G。

#	场景一 （东西流量，一对一） 
一台iperf server(172.16.100.4)，另一台跑iperf client(172.16.100.x)，内网打iperf
 
下表为不同的VPC数与带宽之间的关系
|             | 1VPC           | Cool  |
| ------------|:-------------:| -----:|
| 带宽(Mbps)  | right-aligned | $1600 |
| col 2 is    | centered      |   $12 |
| zebra stripes | are neat      |    $1 |


	1组	4组	5组	9组	20组	30组	50组 	75 组	100组 
BandWidth
Mbps	914 
11G 	230
-	180
-	150
-	70M~150M
12G	60M~150M
12G	50-150M
6G	35~100
5.5G	15~80M
4.7G
注：其中红色的表示两台虚拟机处于同一计算节点，这样两者通信不需要走网络节点，故而带宽比较高

#	场景二 （南北流量，多对一）
一台固定iperf server（挂在外网,192.168.1.140） 另外若干台iperf client 挂在内网通过路由接到外网
 

	1组	5组	10组	20组
BandWidth
Mbps	950M/12G 	2.8G	1.1G	420M
注：同场景一类似，client和server两者处于同一节点上时的带宽也相对较两者处于不同节点上时的带宽要快。
  
5client各自带宽 
 
20client各自带宽

#	场景三 （南北流量，多对多）
5台固定iperf server挂在内网10.200.0.0/24，通过路由链接到外网（浮动IP分别为192.168.1.140/142/145/146/147），另外有5台iperf client挂在内网10.100.200.0/24，也通过路由连接到外网，这5个client组成一个VPC，在这VPC分别往5个server打iperf流量
 

	1VPC	5VPC	20vpc	30VPC
Server1	11G	1.77G	230M	55M
Server2	12G	1.61G	264M	19M
Server3	470M	1.78G	175M	96M
Server4	465M	1.97G	282M	91M
Server5	470M	1.72G	282M	84M
平均	4.88G	1.77G	246M	69M
注:类似的，当client和server同处于同一计算节点上，其两者通信的带宽会比较大（从1VPC的12G左右，到20VPC仍然有几百兆）；而两者处于不同计算节点上，两者通信的带宽相对较小（从1VPC的400-500M下降到20VPC只有10-50M左右）

 
 
 
 
 
5组vpc时 5个server各自的带宽情况

