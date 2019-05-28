---
title: Docker环境下前后端分离项目部署与运维(二)
date: 2019-01-17 12:35:07
tags:
- Docker
- MySql
---



为了搭建 **高性能、高负载、高可用** 的系统，继续[本课程](https://coding.imooc.com/class/219.html)的学习。 

# 第四章 Mysql集群的搭建

## 1. 介绍
![db集群介绍](http://pl2eyyvre.bkt.clouddn.com/docker4-1.1db%E9%9B%86%E7%BE%A4%E4%BB%8B%E7%BB%8D.png)
* PXC (Percona XtraDB Cluster)：数据同步双向、强一致性（同步复制，不能同步就不会执行）
* Replication： 数据同步单向、弱一致性（异步复制）

## 2. 利用docker搭建5个PXC容器的数据库集群

 <!--more-->

### 2.1 下载docker的PXC镜像
```
# 下载镜像
docker pull percona/percona-xtradb-cluster
# 改个名字
docker tag percona/percona-xtradb-cluster pxc
```
### 2.2 创建内部网段
处于安全考虑，需要给PXC集群实例创建Docker内部网络
```
# 创建网段
docker network create net1
# 查看网段
docker network inspect net1
# 删除网段
docker network rm net1
```
### 2.3 创建Docker卷
前言： Docker使用原则，业务数据存储在宿主机上，而不是Docker容器里 ；手段是目录映射；但是pcx技术无法直接使用映射目录。
```
# 创建数据卷
docker volume create --name v1
# 删除数据卷
docker volume rm v1
```
```
# 本次创建5个数据卷
docker volume create --name v1
docker volume create --name v2
docker volume create --name v3
docker volume create --name v4
docker volume create --name v5
```
注：创建数据卷会在宿主机的目录上创建一个空间

### 2.4 创建PXC容器
向PXC镜像传入运行参数，以创建出PXC容器
```
# 创建PXC容器：
# -d:后台运行; -p:端口映射; -v:目录映射; -e:启动参数; --privileged:最高权限; 
# --name:名字; --net:内部网段; --ip:内部网段的ip地址; pxc:镜像的名字。

# 第一个pxc容器
docker run -d -p 3307:3306 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 --privileged --name=node1 --net=net1 --ip 172.18.0.2 pxc

# 第二个pxc容器： 数据卷要不同; 端口要错开; -e CLUSTER_JOIN=node1(和第一个节点同步); ip地址要不同
docker run -d -p 3308:3306 -v v2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 --privileged --name=node2 --net=net1 --ip 172.18.0.3 pxc

# 第三个pxc容器：
docker run -d -p 3309:3306 -v v3:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 --privileged --name=node3 --net=net1 --ip 172.18.0.4 pxc

# 第四个pxc容器： 
docker run -d -p 3310:3306 -v v4:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 --privileged --name=node4 --net=net1 --ip 172.18.0.5 pxc

# 第五个pxc容器：
docker run -d -p 3311:3306 -v v5:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 --privileged --name=node5 --net=net1 --ip 172.18.0.6 pxc
```
注意：创建PXC较快，但是初始化db需要时间。后一个db要在前一个db之前初始化完后（远程数据库工具能连上后）才能继续创建。

创建结果
```
docker ps
```
![db集群介绍](http://pl2eyyvre.bkt.clouddn.com/docker4-2.2db%E9%9B%86%E7%BE%A4%E7%9A%84%E5%88%9B%E5%BB%BA%E7%BB%93%E6%9E%9C.png)

### 2.5 利用远程数据库工具访问（这里选择datagrip）
注意端口号应该是：3307-3311(配置的是宿主机的端口号，而不是容器的端口号，容器的端口号是3306)
![datagrip](http://pl2eyyvre.bkt.clouddn.com/docker4-2.3db%E9%9B%86%E7%BE%A4%E7%9A%84%E5%88%9B%E5%BB%BA%E7%BB%93%E6%9E%9C.png)

可以看见，有5个数据库连接的实例，分别来自5个不同的pxc节点

* 在Node1上创建test库，并且建表test_table1,添加字段name等，会自动同步到其他4个库中。
* 在Node2上的test库，建表test_table2,添加字段age，会自动同步到其他4个库中。

利用docker搭建5节点的PXC集群，告一段落。


## 3. 数据库的负载均衡
### 3.1 负载均衡的意义
单节点处理多有请求，负载高，性能差。变成负载均衡之后，请求被均匀的发给每个节点，单点负载低，性能好。
![负载均衡](http://pl2eyyvre.bkt.clouddn.com/docker4-3.1pxc%20%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.png)

### 3.2 通过 Haproxy 搭建5节点的数据库负载均衡
下载haproxy的镜像
```
docker pull haproxy
```
创建Haproxy配置文件, 比较麻烦，详细参照[这个网站](https://zhangge.net/5125.html)。这里是我的haproxy的[配置文件](http://pl2eyyvre.bkt.clouddn.com/dockerhaproxy.cfg)

```
touch /home/soft/haproxy.cfg
```

创建Haproxy容器
```
# 容器的端口8888：后台管理界面，映射到宿主机的4001; 容器的3306端口：容器的数据库端口，映射到宿主机的4001端口; /home/soft/haproxy宿主机目录，/usr/local/etc/haproxy 容器目录; 名字：h1; 网段：net1(和数据库实例在同一个网段)

docker run -it -d -p 4001:8888 -p 4002:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy --name h1 --privileged --net=net1 haproxy
```

进入后台运行的容器
```
docker exec -it h1 bash
```
设置配置文件
```
haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```
配置文件完成后，先在MySQL数据库里创建一个名叫haproxy的用户，没有初始密码。sql语句为：
``` sql
CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
```

通过管理界面查看数据库状态，地址为： <i>http://your_server_ip:4001/dbs</i>
![管理界面1](http://pl2eyyvre.bkt.clouddn.com/docker4-3.2%205%E8%8A%82%E7%82%B9%E6%95%B0%E6%8D%AE%E5%BA%93%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.png)

测试：挂掉一个节点
```
docker stop node1
```
![管理界面2](http://pl2eyyvre.bkt.clouddn.com/docker4-3.2%204%E8%8A%82%E7%82%B9%E6%95%B0%E6%8D%AE%E5%BA%93%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.png)


## 4. 负载均衡的 **高可用** 方案

### 4.1 单节点的 Haproxy 不具有高可用，需要冗余设计。 
![单节点](http://pl2eyyvre.bkt.clouddn.com/docker4-4.1.png)

### 4.2 虚拟IP技术： 
![虚拟ip](http://pl2eyyvre.bkt.clouddn.com/docker4-4.2%20%E8%99%9A%E6%8B%9Fip.png)

### 4.3 双机热备份：
![双机热备份](http://pl2eyyvre.bkt.clouddn.com/docker4-4.3%20%E5%8F%8C%E6%9C%BA%E7%83%AD%E5%A4%87.png)

双机热备份方案（非常重要，这是后面内容的整体架构图）：
![双机热备份](http://pl2eyyvre.bkt.clouddn.com/docker4-4.4%20%E5%8F%8C%E6%9C%BA%E7%83%AD%E5%A4%87%E4%BB%BD.png)

步骤：
 Step1. 在h1容器内安装Keepalived, 在Haproxy所在的容器（ubuntu）之内：
```
apt-get update
# 安装keepalived
apt-get insatll keepalived
# 安装VIM
apt-get install vim
# 编辑Keepalived配置文件
vim /etc/keepalived/keepalived.conf
```
配置文件keepalived.conf内容如下：
```
vrrp_instance  VI_1 {
    state  MASTER
    # 注意1：此处的网卡的名称要和容器h1的网卡名称保持一致，“ip a” 查看
    interface  eth0
    virtual_router_id  51
    priority  100
    advert_int  1
    authentication {
        auth_type  PASS
        auth_pass  123456
    }
    # 注意2：配置的虚拟网络的ip要和h1在同一个网段下，还记得之前5个pxc的node的ip分别是：172.18.0.x
    virtual_ipaddress {
        172.18.0.201
    }
}
```

启动 keepalived
```
# 启动
service keepalived start
# 测试
ping 172.18.0.201
```
![keepalived在haproxy1里面运行成功](http://pl2eyyvre.bkt.clouddn.com/docker4-4.5%20keepalived%E9%85%8D%E7%BD%AE%E6%88%90%E5%8A%9F.png)

keepalived已经在h1里成功运行

Step2. 创建h2,并且安装keepalived
```
# 创建第2个Haproxy负载均衡服务器
docker run -it -d -p 4003:8888 -p 4004:3306 -v /home/dockerproject/soft/haproxy:/usr/local/etc/haproxy --name h2 --privileged --net=net1 --ip 172.18.0.201 haproxy
# 进入h2容器，启动Haproxy
docker exec -it h2 bash
haproxy -f /usr/local/etc/haproxy/haproxy.cfg
# 进入h2容器
docker exec -it h2 bash
# 更新软件包
apt-get update
# 安装VIM
apt-get install vim
# 安装Keepalived
apt-get install keepalived
# 编辑Keepalived配置文件（和h1相同）
vim /etc/keepalived/keepalived.conf
# 启动Keepalived
service keepalived start
# 退出h2
exit
```
Step3. 本地宿主机安装keepalived进行外网路由：
```
# 宿主机操作：
 # 安装keepalived:
yum install -y keepalived
 
安装完后,配置文件有default的配置文件，/etc/keepalived文件夹中，通过ftp替换掉这个配置文件
```

配置文件如下：
```
vrrp_instance VI_1 {
    state MASTER
#注意3： 这里是宿主机的网卡，可以通过ip a查看当前自己电脑上用的网卡名是哪个
    interface ens192
    virtual_router_id 100
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
# 注意4： 这里是指定的一个宿主机上的虚拟ip，一定要和宿主机网卡在同一个网段，我的宿主机网卡ip是10.103.238.xxx，所以指定虚拟ip是给的200
       	10.103.238.200
    }
}
 
# 接受监听数据来源的端口，网页入口使用
# 注意5：此处是虚拟ip映射，端口号应确保没有被其他程序占用
virtual_server 10.103.238.200 8877 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
#把接受到的数据转发给docker服务的网段及端口，由于是发给docker服务，所以和docker服务数据要一致
    real_server 172.18.0.201 8888 {
        weight 1
    }
}
 
#接受数据库数据端口，宿主机数据库端口是3306，所以这里也要和宿主机数据接受端口一致
virtual_server 10.103.238.200 3306 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
#同理转发数据库给服务的端口和ip要求和docker服务中的数据一致
    real_server 172.18.0.201 3306 {
        weight 1
    }
}
```

6个注意，成功：
![成功](http://pl2eyyvre.bkt.clouddn.com/docker4-4.6%20success.png)

本节内容参考 [docker下用keepalived+Haproxy实现高可用负载均衡集群](https://blog.csdn.net/qq_21108311/article/details/82973763) 和 [高可用MySQL集群搭建](https://supercym.github.io/2019/01/15/Docker-Front-End-Sep-02/)

## 5. 热备份数据
### 5.1 冷备份与热备份
 * 冷备份： 关闭数据库时候的备份方式，通常做法是拷贝数据文件。是一种简单安全的备份方式。大型网站无法做到关闭业务备份数据，但是可以停掉某个pxc节点来进行备份,备份完后再让节点上线同步数据。
 * 热备份： 在系统运行状态下备份数据；MySQL常见的热备份有 LVM 和 XtraBackup两种。LVM 是 linux自带的技术，对某一分区进行快照来备份数据，缺点是需要对数据库加锁。XtraBackup方案则不需要锁表。

 ### 5.2 XtraBackup介绍
 * 基于InnoDB的在线热备工具，备份过程不锁表
 * 不会打断正在执行的事务
 * 能够基于压缩等功能节约磁盘的空间和流量，分为 全量备份 和 增量备份。

 全量备份和增量备份
 * 全量备份： 备份全部数据，时间长，空间大，首次需要；
 * 增量备份： 只备份变化的那部分数据，时间短，空间小。

### 5.3 PXC全量备份步骤
```
# 创建数据卷
docker volume create backup
# 重新生成node1节点（因为要 映射backup这个目录）
docker stop node1
docker rm node1
# 注意：CLUSTER_JOIN=node2,将node1和其他节点同步
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -v v1:/var/lib/mysql -v backup:/data --privileged -e CLUSTER_JOIN=node2 --name=node1 --net=net1 --ip 172.18.0.2 pxc
```
```
# 进入docker的node1容器
docker exec -it --user root node1 bash
# 安装 percona-xtrabackup-24
apt-get update
apt-get install percona-xtrabackup-24
# 开始全量备份
innobackupex --user=root --password=abc123456 /data/backup/full
```
备份成功：
![备份成功](http://pl2eyyvre.bkt.clouddn.com/docker4-5.1%E7%83%AD%E5%A4%87%E4%BB%BD%E5%AE%8C%E6%88%90.png)
```
# 进入容器目录查看
# 进入容器外的目录查看,退出docker容器（ctrl+d）
docker inspect backup
cd /var/lib/docker/volumes/backup/_data
```

### 5.3 PXC全量恢复步骤
数据库备份分为热备份和冷备份，但是还原方式只有冷还原。对于PXC集群，还原的方式比较复杂，为了避免在恢复的过程中数据同步问题，先删除各节点，采用空白的MySQL还原数据，然后重新建立PXC集群，同步数据。还原数据前要将未提交的事务回滚，还原数据之后重启MySQL。
```
# 删除mysql内部数据
rm -rf /var/lib/mysql/*
# 将没有进行完的数据回滚
innobackupex --user=root --password=abc123456 --apply-back /data/backup/full/2019-01-20_12-52-53/
# 还原操作
innobackupex --user=root --password=abc123456 --copy-back /data/backup/full/2019-01-20_12-52-53/
```
具体操作：
```
[root@bigdata-master ~]# docker stop node1 node2 node3 node4 node5
node1
node2
node3
node4
node5
[root@bigdata-master ~]# docker rm node1 node2 node3 node4 node5
node1
node2
node3
node4
node5
[root@bigdata-master ~]# docker volume rm v1 v2 v3 v4 v5
v1
v2
v3
v4
v5
[root@bigdata-master ~]# docker run -d -p 3307:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACK_PASSWORD=abc123456 -v v1:/var/lib/mysql -v backup:/data --privileged --name=node1 --net=net1 --ip 172.18.0.2 pxc
f06d6cc4a1d1ba72b6fc0594dfecfb5e8fc205ebd3556c6617bb69713b72de1b
[root@bigdata-master ~]# docker exec -it node1 bash
[root@bigdata-master ~]# innobackupex --user=root --password=abc123456 --apply-back /data/backup/full/2019-01-20_12-52-53/
[root@bigdata-master ~]# innobackupex --user=root --password=abc123456 --copy-back /data/backup/full/2019-01-20_12-52-53/
root@f06d6cc4a1d1:/# exit
[root@bigdata-master ~]# docker stop node1
node1
[root@bigdata-master ~]# docker start node1
node1
```