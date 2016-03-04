---
layout: post
title: OpenStack Neutron性能测试
tagline: null
category: null
tags: []
published: true

---
# 环境
OpenStack硬件环境为：一台控制节点、一台网络节点、四台计算节点（总计104核 、326G内存、4.8T硬盘）。所有节点通过千兆网路由连接，通过iperf可以检查两个节点间的速率，最高约为120MB/s。

OpenStack网络模块为neutron+ovs。

# 测试方法
本次测试并不是采用Rally测试Neutron Api的并发度，而是测在大量虚拟机并发环境下东西流量和南北流量对网络的负载情况。网络负载性能测试所用的工具为iperf2和iperf3，以虚拟机两两之间的带宽速率作为评价指标。

本次测试环境创建的虚拟机操作系统为ubuntu14.04 x86_64，规格配置为cpu1核，内存1G，硬盘10G。

##	场景一 （东西流量，一对一） 
一台iperf server(172.16.100.4)，另一台跑iperf client(172.16.100.x)，两者组成一个vpc，每一组的client都往自己组里面的server打iperf东西向的流量。
![image](/assets/post-images/2016-03-04-90ab162b-ae4d-4433-e7c9-60e4b6c5abd4.jpg)

下表为不同的VPC数与带宽之间的关系。



|  | 1VPC | 5VPC| 9VPC|20VPC|30VPC|50VPC|75VPC|100VPC|
| ------------- |-------:| -----:| -----:|-----:|-----:|-----:|-----:|-----:|
|带宽mbps     | 914   | 180 |150|70~150|60~150|50~150|35~100|15~80|
|带宽mbps(同一节点)     | <font color=#ff0000>11000</font>  | - |-|<font color=#ff0000>12000</font>|<font color=#ff0000>12000</font>|<font color=#ff0000>6000</font>|<font color=#ff0000>5500</font>|<font color=#ff0000>4700</font>|


**注：其中红色的表示两台虚拟机处于同一计算节点，这样两者通信不需要走网络节点，故而带宽比较高。**

##	场景二 （南北流量，多对一）
一台固定iperf server（通过路由器连接到外网，浮动IP地址192.168.1.140），另外若干台iperf client挂在内网通过路由接到外网。每个client都往server（192.168.1.140）上打iperf南北向流量。

![image](/assets/post-images/2016-03-04-046227fc-4812-4f8b-ef38-9d3ce6bb6415.jpg)


下表为不同的client数与带宽之间的关系。



**注：同场景一类似，client和server两者处于同一节点上时的带宽也相对较两者处于不同节点上时的带宽要快。**
  


##	场景三 （南北流量，多对多）
5台固定iperf server挂在内网10.200.0.0/24，通过路由连接到外网（浮动IP分别为192.168.1.140/142/145/146/147），另外有5台iperf client挂在内网10.100.200.0/24，这5个client组成一个VPC，也通过路由连接到外网。在VPC内部，5个client分别往5个server打iperf南北向流量。

![image](/assets/post-images/2016-03-04-1f563dc9-ef94-4928-c3f9-4ef74409aa93.jpg)


 

|  | 1VPC | 5VPC| 20VPC|30VPC|
| ------------- |-------:| -----:| -----:| -----:|
|Server1     |11000| 1770 |230|55|
|Server2     |12000| 1610 |264|19|
|Server3     |470| 1780 |175|96|
|Server4     |465| 1970 |282|91|
|Server5     |470| 1720 |282|84|


**注：类似的，当client和server同处于同一计算节点上，其两者通信的带宽会比较大（从1VPC的12G左右，到20VPC仍然有几百兆）；而两者处于不同计算节点上，两者通信的带宽相对较小（从1VPC的400-500M下降到20VPC只有10-50M左右）。**

