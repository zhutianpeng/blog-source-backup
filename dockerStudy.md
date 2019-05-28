---
title: Docker环境下前后端分离项目部署与运维(一)
date: 2019-01-16 10:33:55
tags:
- Docker
- Linux
---



为了搭建 **高性能、高负载、高可用** 的系统，开始了[本课程](https://coding.imooc.com/class/219.html)的学习。 

# 第一章 项目架构、软硬件开发环境

## 1. 整体架构
![整体架构](http://pl2eyyvre.bkt.clouddn.com/docker1.%201%E5%AD%A6%E4%B9%A0%E7%9B%AE%E7%9A%84.png )

## 2. 简化部署结构
![简化部署结构](http://pl2eyyvre.bkt.clouddn.com/docker2.1%E7%AE%80%E5%8C%96%E9%83%A8%E7%BD%B2%E7%BB%93%E6%9E%84.png )

 <!--more-->

## 3. 软件环境 (自底向上)
* Wmware虚拟机（或者CentOS机器）
* Docker虚拟机（舱壁隔离）
* JDK、MySQL、Redis（高速缓存）、Nginx（负载均衡）、Node.js（编译前台工程）

## 4. 部署方案
![部署方案](http://pl2eyyvre.bkt.clouddn.com/docker3.1%20%E5%89%8D%E5%90%8E%E5%8F%B0%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2.png )


# 第二章 renren-fast项目介绍
## 1. renren-fast后台项目
* 技术选型
![renren-fast后台项目](http://pl2eyyvre.bkt.clouddn.com/docker5.1renren-fast%E5%90%8E%E5%8F%B0%E9%A1%B9%E7%9B%AE%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B.png )

## 2. renren-fast前端项目
* 技术选型
![renren-fast前端项目](http://pl2eyyvre.bkt.clouddn.com/docker6.1%20renren-fast%E5%89%8D%E7%AB%AF%E9%A1%B9%E7%9B%AE%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B.png )

* 本地前端项目的运行
```
# 克隆项目
git clone https://github.com/daxiongYang/renren-fast-vue.git
# 安装依赖
npm install
# 启动服务
npm run dev
```

# 第三章（上） 基础知识-Linux
## 1. Linux 的优势
* 跨平台的硬件支持
* 丰富的软件支持
* 多用户多任务（权限控制）
* 可靠的安全性（权限管理比较完善，病毒难以获得较高权限）
* 良好的稳定性（安装程序时不需要重启系统）
* 完善的网络功能（自带网络防火墙）

## 2. Linux 目录结构
![Linux目录结构](  http://pl2eyyvre.bkt.clouddn.com/docker3-1.1%20Linux%E7%9A%84%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png )

## 3. Linux 目录与文件管理
``` 
# 创建文件夹
mkdir newproject
# 创建文件
touch hello.txt
# 给文件写入内容(简单的写入)
echo Thanks>hello.txt
# 文件编辑
vi hello.txt
# 复制文件
cp hello.txt new.txt
# 复制文件夹（-r是递归操作）
cp -r myproject newproject 
# 删除文件
rm hello.txt
# 删除文件夹(r：递归删除 f:强制删除)【谨慎操作】
rm -rf myproject
# 移动文件(-f 强制覆盖已存在的目录或文件)
mv -f newproject /home
```
## 4. Linux 文件属性及权限
```
# 查看文件属性
ls -l [可以写具体的文件或者目录，不写即使列出该目录下全部]
```

### 4.1 Linux文件属性
![Linux文件属性](  http://pl2eyyvre.bkt.clouddn.com/docker3-2.1%20Linux%E6%96%87%E4%BB%B6%E6%9D%83%E9%99%90.png )

* 文件类型  (第一位)
* 属主权限 （后三位为一组）（创建改文件的用户 具有的权限）
* 属组权限 （后三位为一组）（创建该文件的用户所在用户组 具有的权限）
* 其他用户权限（最后三位为一组）（创建该文件的用户所不在的用户组 具有的权限）

### 4.2 Linux文件详细目录
 ![Linux文件详细信息](   http://pl2eyyvre.bkt.clouddn.com/docker3-2.2%20Linux%E6%96%87%E4%BB%B6%E8%AF%A6%E7%BB%86%E4%BF%A1%E6%81%AF.png )

### 4.3 Linux修改文件权限
```
chmod 700 hello.txt
```
解释：700的含义，各个权限相加等于一位数字。
* r : 4
* w : 2
* x : 1

所以上述700的含义是： 将该文件的权限改为：属主可读可写可执行；属组无权限；其他用户组无权限。

## 5. Linux 防火墙的管理
### 5.1 防火墙意义
 ![Linux防火墙意义](   http://pl2eyyvre.bkt.clouddn.com/docker3-3.1Linux%E9%98%B2%E7%81%AB%E5%A2%99%E7%9A%84%E6%84%8F%E4%B9%89.png )

### 5.2 相关指令
```
# 查看状态
firewall-cmd --state
# 启动
service firewall start
# 关闭
service firewall stop
# 重启
service firewall restart
```
### 5.3 管理防火墙的 网络端口
```
# 添加端口（开放一批端口）
firewall-cmd --permanent --add-port=8080-8085/tcp
# 加载最新设置
firewall-cmd --reload
# 删除端口 (要和添加端口的时候保持一致)
firewall-cmd --permanent --remove-port=8080-8085/tcp
# 查看防火墙开放了哪些端口【看ports】
firewall-cmd --permanent --list-ports
# 查看防火墙通过上述端口开始了哪些服务【看services】
firewall-cmd --permanent --list-services
```


# 第三章（下） 基础知识-Docker虚拟机
## 1. Docker虚拟机架构
Docker创建的所有虚拟实例共用同一个Linux内核，对硬件占用比较小，属于轻量级虚拟机。
 ![Docker虚拟机架构](http://pl2eyyvre.bkt.clouddn.com/docker3-4.1docker%E6%9E%B6%E6%9E%84.png)

## 2.Docker虚拟机与云计算
 ![Docker虚拟机与云计算](http://pl2eyyvre.bkt.clouddn.com/docker3-4.2docker%E4%B8%8E%E4%BA%91%E8%AE%A1%E7%AE%97.png)
 
 * iaas : Infrastructure-as-a-Service (Docker做虚拟主机)
 * paas : Platform-as-a-Service (Docker的容器可以安装相应的中间件，并生成镜像文件)
 * saas : Software-as-a-Service (Docker的容器里面部署上项目，并生成镜像文件)

## 3.镜像与容器
  ![镜像与容器](http://pl2eyyvre.bkt.clouddn.com/docker3-4.3%20%E9%95%9C%E5%83%8F%E4%B8%8E%E5%AE%B9%E5%99%A8%E7%9A%84%E5%85%B3%E7%B3%BB.png)

## 4. Docker相关指令
### 4.1 安装（先更新yum,再安装Docker）：
```
yum -y uodate
yum install -y docker
```
### 4.2 启动、关闭与重启：
```
service docker start
service docker stop
service docker restart
```
### 4.3 Docker常用指令：
  ![Docker常用指令](http://pl2eyyvre.bkt.clouddn.com/docker3-4.4%20docker%E7%AE%A1%E7%90%86.png)

配置docker加速器：[daocloud网站](https://www.daocloud.io/mirror)，配置完成后即安装相应镜像

```
# 在镜像仓库里面搜索跟java相关的镜像文件
docker search java
# 下载合适的镜像文件
docker pull java 
# 查看已经安装的镜像
docker images
```
导出导入镜像 (压缩文件的形式)
```
# 导出
docker save java > /home/java.tar.gz
# 导入
docker load < /home/java.tar.gz
# 删除
rmi java
```

启动容器（以下指令，参数可叠加）
```
#1. 启动镜像会创建出一个运行状态的容器， -it：启动容器后开启交互界面; --name:给容器起名字; java:镜像的名字; bash：运行程序时bash命令行
docker run -it --name myjava java bash

#2. 启动镜像的时候做端口映射，-p:映射端口,9000：宿主机端口，8080：容器端口；
# （解释：把容器的8080端口映射到真实的宿主机的9000端口上）
docker run -it --name myjava -p 9000:8080 -p 9001:8085 java bash

#3.  文件或者目录的映射, -v:文件映射,  /home/project:宿主机真实文件目录;  soft：容器的目录; --privileged:给容器富有最高权限
# (解释：将宿主机的/home/project目录 映射到 容器的soft目录， 并且给soft目录最高的文件读、写、执行权限)
docker run -it --name myjava -v /home/project:/soft --privileged java bash
```

暂停和停止容器
```
# 暂停容器（myjava是容器的名字，或者根据容器的编号）
docker pause myjava
# 恢复运行
docker unpause myjava
# 停止容器
docker stop myjava
# 恢复运行
docker start -i myjava
 
# 退出容器的交互界面，同时也stop容器
exit

# 删除容器
docker rm myjava

# 查看现存容器
docker ps -a
```