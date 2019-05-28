---
title: Docker环境下前后端分离项目部署与运维(三)
date: 2019-01-20 22:14:43
tags:
- Docker
- Redis
---

为了搭建 **高性能、高负载、高可用** 的系统，继续[本课程](https://coding.imooc.com/class/219.html)的学习。 

# 第5章 搭建Redis集群
## 5.1 Redis主从同步集群介绍

### Redis集群介绍：
* RedisCluster: 官方推荐，没有中心节点 （采用）
* Codis： 中间件产品，存在中心节点
* Twemproxy: 中间件产品，存在中心节点（较老）

### RedisCluster介绍：
* 无中心节点，每个节点都是可读可写，客户端与redis节点直连，不需要中间代理层
* 数据可以被分片存储，每个节点存储的数据不同（与PXC集群不同）；如果一个节点挂掉了，需要有冗余节点继续提供服务（备份）。
* 管理方便，后续可自行增加和摘除节点
 ![redisCluster示意图](http://pl2eyyvre.bkt.clouddn.com/docker5-1.1%20redisCluster%E7%A4%BA%E6%84%8F%E5%9B%BE.png)


 <!--more-->


### RedisCluster的主从同步：
* Redis集群中的数据库赋值是通过主从同步来实现
* 主节点（Master）把数据分发给从节点（slave）
* 优点：高可用，Reids节点有冗余设计

### RedisCluster的**高可用**：
* Redis集群应该包含奇数个Master，至少有3个Master（原因：Redis集群和PXC节点有选举性，当一个节点挂掉，剩下的节点如果超过总数的一半，则通过选举选出中心节点，组成新的集群。只有2个节点则不可用）
* Redis集群中每个Master都应该有Slave：

![每个Master都应该有Slave](http://pl2eyyvre.bkt.clouddn.com/docker5-1.2%20%E4%B8%BB%E4%BB%8E.png)

* 此处不用搭建负载均衡，工程基于Spring,已经实现了负载均衡

## 5.2 配置RedisCluster集群

此处的架构图：

![架构图](http://pl2eyyvre.bkt.clouddn.com/docker5-1.2%20%E4%B8%BB%E4%BB%8E%20%E7%BB%93%E6%9E%84.png)

### 安装Redis镜像
```
[root@bigdata-master ~]# docker network create --subnet=172.19.0.0/16 net2
6c0b633b58abd6bd47381840f94956c0307ce93a1aa6650c41e4034b6f1e5127

[root@bigdata-master ~]# docker pull yyyyttttwwww/redis
[root@bigdata-master ~]# docker tag docker.io/yyyyttttwwww/redis redis
[root@bigdata-master ~]# docker rmi docker.io/yyyyttttwwww/redis

docker run -it -d --name r1 -p 5001:6379 --net=net2 --ip 172.19.0.2 redis bash
```
### 配置redis节点
配置文件路径 /usr/redis/redis.conf，需要修改的重点配置如下：
```
daemonize yes                       #以后台进程运行
cluster-enabled yes                 #开启集群
cluster-config-file nodes.conf      #集群配置文件
cluster-node-timeout 15000          #超时时间
appendonly yes                      #开启AOF模式（日志功能）
```
**注意此处需要：**
```
bind 127.0.0.1 
改为
bind 172.19.0.x(本虚拟机的局域网ip地址，不然后面创建redis-cluster会失败) 
```
注意要删掉127.0.0.1，此处是redis的一个bug, 参照 [stack overflow的解答方案](https://stackoverflow.com/a/53128159/8526099)。

启动redis
```
root@c265772c4715:/# cd /usr/redis/src/
root@c265772c4715:/usr/redis/src# ./redis-server ../redis.conf
```
至此，一个redis节点已经配置并启动完毕，6节点的redis集群，one is done, five to go...
启动剩下的5个节点：
```
[root@bigdata-master ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS                                            NAMES
172dbca59e17        redis               "bash"                   19 minutes ago      Up 19 minutes             0.0.0.0:5006->6379/tcp                           r6
f8b0df9ab628        redis               "bash"                   26 minutes ago      Up 26 minutes             0.0.0.0:5005->6379/tcp                           r5
07767c6d2b22        redis               "bash"                   29 minutes ago      Up 29 minutes             0.0.0.0:5004->6379/tcp                           r4
94778bf771f8        redis               "bash"                   32 minutes ago      Up 32 minutes             0.0.0.0:5003->6379/tcp                           r3
b6eec374ed19        redis               "bash"                   37 minutes ago      Up 37 minutes             0.0.0.0:5002->6379/tcp                           r2
c265772c4715        redis               "bash"                   21 hours ago        Up 21 hours               0.0.0.0:5001->6379/tcp                           r1
f06d6cc4a1d1        pxc                 "/entrypoint.sh "        23 hours ago        Exited (1) 22 hours ago                                                    node1
5c0d6c170ca1        haproxy             "/docker-entrypoin..."   45 hours ago        Up 45 hours               0.0.0.0:4004->3306/tcp, 0.0.0.0:4003->8888/tcp   h2
fe53e56e10c3        haproxy             "/docker-entrypoin..."   3 days ago          Up 3 days                 0.0.0.0:4002->3306/tcp, 0.0.0.0:4001->8888/tcp   h1
58dc77c722ef        docker.io/java      "bash"                   4 days ago          Exited (130) 4 days ago                                                    myjava
[root@bigdata-master ~]#
```

### 安装 redis-trib.rb
根据redis-trib.rb脚本（基于Ruby的redis集群命令行工具）安装
```
#进入任意一个redis目录： docker exec -it r1 bash
root@c265772c4715:/# cd /usr/redis
root@c265772c4715:/# mkdir cluster
root@c265772c4715:/# cp /usr/redis/erc/redis-trib.rb /usr/redis/cluster
root@c265772c4715:/# cd /usr/redis/cluster
root@c265772c4715:cluster/# apt-get install ruby
root@c265772c4715:cluster/# apt-get install rubygems
root@c265772c4715:cluster/# gem install redis
```

创建redis集群
```
root@c265772c4715:cluster/# ./redis-trib.rb create --replicas 1 172.19.0.2:6379 172.19.0.3:6379 172.19.0.4:6379 172.19.0.5:6379 172.19.0.6:6379 172.19.0.7:6379 
```
这里我创建失败了，原因是之前没有在redis的配置文件里面 “bind 172.19.0.x”，修改6个节点的redis配置文件后，到如下步骤。
```
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
172.19.0.2:6379
172.19.0.3:6379
172.19.0.4:6379
Adding replica 172.19.0.5:6379 to 172.19.0.2:6379
Adding replica 172.19.0.6:6379 to 172.19.0.3:6379
Adding replica 172.19.0.7:6379 to 172.19.0.4:6379
M: 976f833b9002bbde8b687a08c3727c5017833bdf 172.19.0.2:6379
   slots:0-5460 (5461 slots) master
M: e832ec72a97273963ea74c950be86158ce0fac91 172.19.0.3:6379
   slots:5461-10922 (5462 slots) master
M: 76f42d918ae5c85d70f0b70724e852b0ef07aae5 172.19.0.4:6379
   slots:10923-16383 (5461 slots) master
S: 5b0f17a4b2cc713cd11854d2a229b2d1b56de223 172.19.0.5:6379
   replicates 976f833b9002bbde8b687a08c3727c5017833bdf
S: 0c4b318aef032deae4f1d832e2b1ae085bdf35a9 172.19.0.6:6379
   replicates e832ec72a97273963ea74c950be86158ce0fac91
S: 2dc444a9321a6668f732c7b58ebf61a79f1e4fe4 172.19.0.7:6379
   replicates 76f42d918ae5c85d70f0b70724e852b0ef07aae5
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join........................................................................
..............................................................................................

```

> 原因：redis集群不仅需要开通redis客户端连接的端口，而且需要开通集群总线端口，集群总线端口为redis客户端连接的端口 + 10000。如redis端口为6379，则集群总线端口为16379。故，所有服务器的点需要开通redis的客户端连接端口和集群总线端口。注意：iptables 放开，如果有安全组，也要放开这两个端口。

在修改完后重新启动之前，需要连上各节点的redis-cli,进行下列清空内容，重启cluster的操作：

```
172.19.0.2:6379> flushall
172.19.0.2:6379> cluster reset
```

最终，创建成功。

```
Waiting for the cluster to join.....
>>> Performing Cluster Check (using node 172.19.0.2:6379)
M: 976f833b9002bbde8b687a08c3727c5017833bdf 172.19.0.2:6379
   slots:0-5460 (5461 slots) master
M: e832ec72a97273963ea74c950be86158ce0fac91 172.19.0.3:6379
   slots:5461-10922 (5462 slots) master
M: 76f42d918ae5c85d70f0b70724e852b0ef07aae5 172.19.0.4:6379
   slots:10923-16383 (5461 slots) master
M: 5b0f17a4b2cc713cd11854d2a229b2d1b56de223 172.19.0.5:6379
   slots: (0 slots) master
   replicates 976f833b9002bbde8b687a08c3727c5017833bdf
M: 0c4b318aef032deae4f1d832e2b1ae085bdf35a9 172.19.0.6:6379
   slots: (0 slots) master
   replicates e832ec72a97273963ea74c950be86158ce0fac91
M: 2dc444a9321a6668f732c7b58ebf61a79f1e4fe4 172.19.0.7:6379
   slots: (0 slots) master
   replicates 76f42d918ae5c85d70f0b70724e852b0ef07aae5
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## 5.3 测试RedisCluster集群

连上r1容器，启动redis客户端，-c：是连接cluster，可存可取，并且是分片存储（Redirected to slot [15495] located at 172.19.0.4:6379，可以看出是分片到了第三个master节点在保存。）

```
[root@bigdata-master ~]# docker exec -it r1 bash
root@c265772c4715:/# cd usr/redis/src/
root@c265772c4715:/usr/redis/src# ./redis-cli -c -h 172.19.0.2
172.19.0.2:6379> set a 10
-> Redirected to slot [15495] located at 172.19.0.4:6379
OK
172.19.0.4:6379>
```

现在进行测试，停止 r1。可以看见，172.19.0.4 fail, 172.19.0.7 升级为master。再次get a, 发现是从172.0.0.7获取到了数据。

```
[root@bigdata-master ~]# docker pause r3

172.19.0.2:6379> cluster nodes
2dc444a9321a6668f732c7b58ebf61a79f1e4fe4 172.19.0.7:6379 master - 0 1548084429158 7 connected 10923-16383
976f833b9002bbde8b687a08c3727c5017833bdf 172.19.0.2:6379 myself,master - 0 0 1 connected 0-5460
5b0f17a4b2cc713cd11854d2a229b2d1b56de223 172.19.0.5:6379 slave 976f833b9002bbde8b687a08c3727c5017833bdf 0 1548084428157 4 connected
76f42d918ae5c85d70f0b70724e852b0ef07aae5 172.19.0.4:6379 master,fail - 1548084279946 1548084275943 3 connected
0c4b318aef032deae4f1d832e2b1ae085bdf35a9 172.19.0.6:6379 slave e832ec72a97273963ea74c950be86158ce0fac91 0 1548084428657 5 connected
e832ec72a97273963ea74c950be86158ce0fac91 172.19.0.3:6379 master - 0 1548084430159 2 connected 5461-10922

172.19.0.2:6379> get a
-> Redirected to slot [15495] located at 172.19.0.7:6379
"10"

```

下面恢复r3节点，查看cluster状态，发现r3节点重新上线，但是成为了slave节点。

```
[root@bigdata-master ~]# docker unpause r3
r3

172.19.0.7:6379> cluster nodes
0c4b318aef032deae4f1d832e2b1ae085bdf35a9 172.19.0.6:6379 slave e832ec72a97273963ea74c950be86158ce0fac91 0 1548084694915 5 connected
76f42d918ae5c85d70f0b70724e852b0ef07aae5 172.19.0.4:6379 slave 2dc444a9321a6668f732c7b58ebf61a79f1e4fe4 0 1548084693915 7 connected
e832ec72a97273963ea74c950be86158ce0fac91 172.19.0.3:6379 master - 0 1548084695916 2 connected 5461-10922
5b0f17a4b2cc713cd11854d2a229b2d1b56de223 172.19.0.5:6379 slave 976f833b9002bbde8b687a08c3727c5017833bdf 0 1548084697919 4 connected
2dc444a9321a6668f732c7b58ebf61a79f1e4fe4 172.19.0.7:6379 myself,master - 0 0 7 connected 10923-16383
976f833b9002bbde8b687a08c3727c5017833bdf 172.19.0.2:6379 master - 0 1548084696918 1 connected 0-5460
```

以上测试表明了redis-cluster具有一定的 **高可用性**。