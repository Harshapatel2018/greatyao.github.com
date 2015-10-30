---
layout: post
title: OpenStack Liberty版本Magnum模块的极速体验
tagline: null
keywords: OpenStack
categories: [OpenStack]
tags: [OpenStack, Magnum, DevStack]
group: archive
icon: leaf
published: true
---

{% include codepiano/setup %}

---
10月15日，OpenStack发布了Liberty版本，其中一个变化就是容器即服务模块Magnum正式推出1.0版本，只是各大Linux发行厂商还没有发布magnum的安装包，因此需要从源码直接构建。下面给出自己基于Devstack方式的完整安装实践过程。

# 环境准备 

在VMware Workstations建台虚拟机，Ubuntu 14.04 LTS，CPU两个核以上，内存至少4G以上，硬盘100G，两个网卡（一个主机模式，另一个NAT模式）。

    $ vi /etc/network/interfaces
    #仅主机模式，用于openstack内部模块通信 
    auto eth0
    iface eth0 inet static
        address 20.0.0.11
        gateway 20.0.0.1
        netmask 255.255.255.0

    #NAT模式，用于虚拟机访问外网的外部网络
    auto eth1
    iface eth1 inet static
        address 172.24.54.222
        gateway 172.24.54.2
        netmask 255.255.255.0

    #将缺省路由设置为网卡2
    up route add default gw 172.24.54.2 dev eth1

# 部署过程

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

6.运行Devstack
    
    $ ./stack.sh

注意：使用的是stack用户运行。在安装过程中，可能会提示apt-get下载源错误，重复执行上述安装命令；其他情况，可以再次执行安装命令。

    $ ./unstack.sh && ./stack.sh
    
7.验证

整个安装过程耗时一个小时左右，取决于你的网络状况。在浏览器中打开http://20.0.0.11/即可访问horizon。默认Devstack创建admin和demo两个用户，密码是第5步中localrc中设置的password。

或者可以使用OpenStack的命令行工具来验证

    $ cd /home/devstack
    $ source openrc admin admin
    $ glance image-list
    +--------------------------------------+---------------------------------+
    | ID                                   | Name                            |
    +--------------------------------------+---------------------------------+
    | 148f3031-13cd-4a93-85fe-03b42b2117c6 | cirros-0.3.4-x86_64-uec         |
    | c291cb0c-6265-4be9-8eb2-d2988bda203c | cirros-0.3.4-x86_64-uec-kernel  |
    | 502aca3d-4e41-4e41-ad50-baa82c0a6cde | cirros-0.3.4-x86_64-uec-ramdisk |
    | b89a7a9f-3b2f-4291-8327-60c735495b09 | fedora-21-atomic-5              |
    +--------------------------------------+---------------------------------+

# 窥探Magnum

Magnum一般有两个子模块，magnum-api和magnum-conductor,为了验证conductor服务是否健康运行

    $ magnum service-list
    +----+-----------+------------------+-------+
    | id | host      | binary           | state |
    +----+-----------+------------------+-------+
    | 1  | localhost | magnum-conductor | up    |
    +----+-----------+------------------+-------+


1.创建密钥对

    $ test -f ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
    $ nova keypair-add --pub-key ~/.ssh/id_rsa.pub testkey
    
2.创建BayModel对象

    $ magnum baymodel-create --name k8sbaymodel \
       --image-id fedora-21-atomic-5 \
       --keypair-id testkey \
       --external-network-id public \
       --dns-nameserver 8.8.8.8 \
       --flavor-id m1.small \
       --docker-volume-size 5 \
       --network-driver flannel \
       --coe kubernetes
                       
3.创建Bay对象

    $ magnum bay-create --name k8sbay --baymodel k8sbaymodel --node-count 1

Bays对象初始化为CREATE_IN_PROGRESS状态，当创建完成后会更新为CREATE_COMPLETE状态. 此步骤耗时10多分钟，可以通过以下命令查看已有的Bays对象

    $ magnum bay-list
    +--------------------------------------+--------+------------+--------------+-----------------+
    | uuid                                 | name   | node_count | master_count | status          |
    +--------------------------------------+--------+------------+--------------+-----------------+
    | f0ce1f2d-a1a9-4a47-977f-15244625463d | k8sbay | 1          | 1            | CREATE_COMPLETE |
    +--------------------------------------+--------+------------+--------------+-----------------+
      
      
上述的Bay对象将会创建一个Kubernetes master节点和一个minion节点:  

    $ nova list
    +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+------------------------------+
    | ID                                   | Name                                                  | Status | Task State | Power State | Networks                     |
    +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+------------------------------+
    | 4656ff13-9210-47b5-9c27-8c26309d5e04 | k8-fpoxo3qagq-0-bax3fvtb5pf2-kube_master-riv2fk732dbd | ACTIVE | -          | Running     | private=10.0.0.5, 172.24.4.5 |
    | 45b7c2af-29de-4f59-b5ba-b784e0ef503b | k8-hdx2ewllqx-0-5vptn5l4ahtt-kube-minion-i6jyfvsvxkzu | ACTIVE | -          | Running     | private=10.0.0.6, 172.24.4.6 |
    +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+------------------------------+
    
更直观的，可以在Horizon的编配里面查看相关栈或Bay的详情
![image](/assets/post-images/2015-10-30-70479d1c-db71-4383-dd15-8f5df12bb94c.jpg)

4.下载Kubernetes源码

    $ wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.1/kubernetes.tar.gz
    $ tar -xvzf kubernetes.tar.gz

5.部署Redis容器集群

    $ cd kubernetes/examples/redis
    $ magnum pod-create --manifest ./redis-master.yaml --bay k8sbay
    
    $ magnum coe-service-create --manifest ./redis-sentinel-service.yaml --bay k8sbay
    
    $ sed -i 's/\(replicas: \)1/\1 2/' redis-controller.yaml
    $ magnum rc-create --manifest ./redis-controller.yaml --bay k8sbay
    
    $ sed -i 's/\(replicas: \)1/\1 2/' redis-sentinel-controller.yaml
    $ magnum rc-create --manifest ./redis-sentinel-controller.yaml --bay k8sbay

这样我们就创建了一个redis的ReplicationController，由这个Controller来调度和管理redis容器，通过magnum命令可以查看IP与状态。

    $ magnum bay-show k8sbay
    +--------------------+------------------------------------------------------------+
    | Property           | Value                                                      |
    +--------------------+------------------------------------------------------------+
    | status             | CREATE_COMPLETE                                            |
    | uuid               | f0ce1f2d-a1a9-4a47-977f-15244625463d                       |
    | status_reason      | Stack CREATE completed successfully                        |
    | created_at         | 2015-10-20T12:56:07+00:00                                  |
    | updated_at         | 2015-10-20T13:13:41+00:00                                  |
    | bay_create_timeout | 0                                                          |
    | api_address        | https://172.24.4.4:6443                                    |
    | baymodel_id        | 33d8e94d-f867-437f-80b4-10d5d6885c8a                       |
    | node_count         | 1                                                          |
    | node_addresses     | [u'172.24.4.6']                                            |
    | master_count       | 1                                                          |
    | discovery_url      | https://discovery.etcd.io/6533dd3e28c0ff7e410cff633fad90ad |
    | name               | k8sbay                                                     |
    +--------------------+------------------------------------------------------------+
    
*注意：使用Kubernetes部署Pod时会从Google container registry下载基础的Pause镜像，但众所周知由于国内特殊的网络环境，会导致容器无法部署成功。所以咱最好还是在主机上拨个VPN。*


6.验证

剩下来就是ssh到相应的虚拟机中，通过docker命令或redis客户端来控制和访问容器了。

    $ ssh minion@172.24.4.6
    [minion@k8-hdx2ewllqx-0-5vptn5l4ahtt-kube-minion-i6jyfvsvxkzu ~]$ sudo su
    bash-4.3# docker ps
    CONTAINER ID        IMAGE                                  COMMAND             CREATED             STATUS              PORTS               NAMES
    6522a94e8c2a        gcr.io/google_containers/pause:0.8.0   "/pause"            58 minutes ago      Up 55 minutes                           k8s_POD.49eee8c2_redis-bldhr_default_8f5a1e81-7739-11e5-8d21-fa163ec6c383_cb8d1102
    277bbcfca7df        gcr.io/google_containers/pause:0.8.0   "/pause"            58 minutes ago      Up 56 minutes                           k8s_POD.39750b55_redis-master_default_690a053d-7739-11e5-8d21-fa163ec6c383_1da0a23d
    fbc0bf70079e        gcr.io/google_containers/pause:0.8.0   "/pause"            58 minutes ago      Up 56 minutes                           k8s_POD.ecf0e8f4_redis-sentinel-ii2go_default_9ca36eec-7739-11e5-8d21-fa163ec6c383_9a95d2d0

