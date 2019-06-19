---

layout: post
title: 分布式配置系统
category: 技术
tags: Distribute
keywords: 分布式配置系统 zookeeper etcd Consul

---

## 简介


Zookeeper is a **distributed storage** that provides the following guarantees

1. Sequential Consistency - Updates from a client will be applied in the order that they were sent.
2. Atomicity - Updates either succeed or fail. No partial results.
3. Single System Image - A client will see the same view of the service regardless of the server that it connects to.
4. Reliability - Once an update has been applied, it will persist from that time forward until a client overwrites the update.
5. Timeliness - The clients view of the system is guaranteed to be up-to-date within a certain time bound.

You can use these to implement different **recipes** that are required for cluster management like locks, leader election etc.

从这段描述中，我们就可以知道，什么是本质（distributed storage + guarantees），什么是recipes。

2017.11.23 更新

我们都以为用zookeeper做一致性工具天经地义，[来自京东、唯品会对微服务编排、API网关、持续集成的实践分享（上）](https://my.oschina.net/u/1777263/blog/827661)却用db做一致性，一切看场景。就注册中心的功能来说，[Netflix/eureka](https://github.com/Netflix/eureka)也比zookeeper更好些。换个方式考虑，注册中心本质为了服务调用方和提供方的解耦，存储服务注册信息。也就是能存数据的都可以用来做注册中心，但从可用性上考虑，zookeeper因为副本因素可靠性高些。一致性 <== 副本 <== 高可用性存储，这或许才是zookeeper等一致性工具的本质，其它的才是kv存储、通知机制等枝节。 

## 现有产品

分布式配置系统一般有zookeeper,etcd,Consul等

分布式配置中心一般有以下特点：

1. 集中管理外部依赖的服务配置和服务内部配置
2. 提供web管理平台进行配置和查询
3. 支持服务注册与发现
4. 支持客户端拉取配置
5. 支持订阅与发布，配置变更主动通知到client，实时变更配置

    这条跟消息队列很像，不过两者的目的完全不同。
    
6. 分布式配置中心本身具有非常高的可靠性

    因此一般以集群状态运行，集群节点的增加和减少不影响服务的提供。


我个人觉得比较直观的一条便是：

- 使用之前，分布式系统各主机间交互，需要跨网络访问
- 使用之后，分布式系统的daemon服务只需与配置中心daemon交互即可。省事，尤其是一个消息要通知好几个主机时。

    - 两台电脑互通，需要一根网线
    - 三台电脑互通，需要三根网线
    - 四台电脑互通，需要六根网线
    - 四台电脑互通，如果使用集线器，则需要四根网线即可

**其作用类似于集线器，不同主机的各个进程连上它，就不用彼此之间费事通信了，当然，通信的数据一般是对连接上配置中心的所有进程都有用的**。如果是交互的数据量太大，或数据只与某两个进程相关，还是各主机自己动手或使用消息队列。

## 通过安装过程来体会工具的异同

### etcd

以ubuntu15.04 为例，用Systemd管理etcd，针对每个主机

1. 下载文件  `https://github.com/coreos/etcd/releases`
2. 拷贝可执行文件etcd,etcdctl到path目录
3. 准备`/etc/systemd/system/etcd.service`文件

        [Unit]
        Description=etcd shared configuration and service discovery daemon
        
        [Service]
        Type=notify
        EnvironmentFile=/etc/default/etcd
        ExecStart=/usr/local/bin/etcd $ETCD_OPTS
        Restart=on-failure
        RestartSec=5
        
        [Install]
        WantedBy=multi-user.target
        
4. 准备配置文件`/etc/default/etcd`

        ETCD_OPTS=-addr=server_ip:4001 -peer-addr=server_ip:7001 -data-dir=/var/lib/etcd/
        
5. `systemctl enable etcd`使配置文件生效，`systemctl start etcd`启动etcd，`systemctl status etcd`查看etcd运行状态。

几个端口的作用

- 4001，客户端（比如使用etcd的应用程序）通过它访问etcd数据
- 7001，Etcd节点通过7001端口在集群各节点间同步Raft状态和数据

etcd启动时，有三种模式`static`,`etcd Discovery`和`DNS Discovery`三种模式来确定哪些节点是“自己人”，参见`https://coreos.com/etcd/docs/latest/clustering.html`

存储结构，键值对，键以文件夹的形式组织，例如

    root@docker1:~# etcdctl ls /network/docker/nodes
    /network/docker/nodes/192.168.56.101:2375
    /network/docker/nodes/192.168.56.102:2375

### zookeeper

针对每个主机

1. 将文件解压到特定位置，`tar -xzvf zookeeper-x.x.x.tar.gz -C xxx`
2. 根据样本文件创建配置文件，`cp zookeeper-x.x.x/zoo_sample.cfg zookeeper-x.x.x/zoo.cfg`
3. 更改配置文件

        dataDir=/var/lib/zookeeper
        clientPort=2181
        server.1=server_ip1:2888:3888
        server.2=server_ip2:2888:3888
        server.3=server_ip3:2888:3888
        
4. 编辑myid文件，`${dataDir}/myid`，不同的节点写个不同的数字即可
5. 启动zookeeper，`zookeeper-x.x.x/bin/zkServer.sh start`

几个端口的作用：

- 端口2181由 ZooKeeper客户端（比如访问zookeeper数据的应用程序）使用，用于连接到 ZooKeeper 服务器；
- 端口2888由对等 ZooKeeper 服务器使用，用于互相通信；
- 而端口3888用于领导者选举。

集群配置时，集群有哪些节点，已在所有节点的配置文件中讲明，比如这里的`server.1,server.2,server.3`


## 数据模型

### zookeeper

ZooKeeper的数据结构, 与普通的文件系统类似，每个节点称为一个znode. 每个znode由3部分组成:

1. stat. 此为状态信息, 描述该znode的版本, 权限等信息.
2. data. 与该znode关联的数据.
3. children. 该znode下的子节点.

## 小结

分布式配置系统一般有以下不同：

1. 对外提供的数据格式**未完待续**
2. 一致性原理

有以下相同的地方

1. 都会将部分数据存储在本地（可指定存储目录）
2. 都是master-slave机制，当master挂掉时，集群立即协商新的master，并以master为准同步数据。
3. 分布式配置中心为提高可靠性，会以集群的形态存在，那么集群在启动时，如何判定哪些节点是“自己人”。

其它，随着版本的发展，etcd和zookeeper等配置项和端口的意义会有些变化，此处不再赘述。

## 典型应用场景

1. 假设一个系统有多个后端，后端的状态在不断变化（后端可能会宕机，也可能有新的后端加入）。那么每个节点就需要感知到这种变化，并作出反应。
2. 动态配置中心。假设系统的运行受一个参数的影响，那么可以在etcd等应用中更改这个参数，并将参数变化推送到各个节点，各节点据此更改内存中的状态信息，无需重启。
3. 分布式锁。对于同一主机共享资源的访问，可以用锁来协调同一主机的多个进程（或线程）。对于跨主机共享资源的访问，etcd等工具提供相应的工具。
4. 一个系统包括多个组件，组件间不是通过API直接通信，而是通过Watch ETCD变化来通信，从而减少组件间的耦合。

## 引用

[ZooKeeper 基础知识、部署和应用程序][]

[ZooKeeper 基础知识、部署和应用程序]: http://www.ibm.com/developerworks/cn/data/library/bd-zookeeper/