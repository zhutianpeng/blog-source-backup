---
title: Docker环境下前后端分离项目部署与运维(四)
date: 2019-01-22 14:10:50
tags:
- Docker
- 前后端分离
---

为了搭建 **高性能、高负载、高可用** 的系统，继续[本课程](https://coding.imooc.com/class/219.html)的学习。 

# 第6章 后端项目部署与负载均衡

## 1.后端项目部署与负载均衡

### 1.1 导入数据库，配置数据库连接

用数据库工具连接 在 [Docker环境下前后端分离项目部署与运维(二)](http://ztxpp.cc/2019/01/17/dockerStudy2/) 中 配置好的服务器数据库集群，宿主机虚拟ip：10.103.238.200:3306，导入renren_fast后端的mysql数据。

在renren_fast工程中修改数据库的连接配置文件 application-dev.yml， 修改如下：

```
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.jdbc.Driver
        druid:
            first:  #数据源1
                url: jdbc:mysql://10.103.238.200:3306/renren_fast?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
                username: root
                password: abc123456
            second:  #数据源2
                url: jdbc:mysql://10.103.238.200:3306/renren_fast?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
                username: root
                password: abc123456
```

### 1.2 添加 Redis 集群

修改renren_fast工程中的 Redis配置文件 application.yml。 修改如下：

```
 redis:
    open: false  # 是否开启redis缓存  true开启   false关闭
    database: 0
#    host: localhost
#    port: 6379
#    password:    # 密码（默认为空）
    timeout: 6000ms  # 连接超时时长（毫秒）
    cluster:
      nodes:
      - 172.19.0.2:6379
      - 172.19.0.3:6379
      - 172.19.0.4:6379
      - 172.19.0.5:6379
      - 172.19.0.6:6379
      - 172.19.0.7:6379
    jedis:
      pool:
        max-active: 1000  # 连接池最大连接数（使用负值表示没有限制）
        max-wait: -1ms      # 连接池最大阻塞等待时间（使用负值表示没有限制）
        max-idle: 10      # 连接池中的最大空闲连接
        min-idle: 5       # 连接池中的最小空闲连接
```

### 1.3 修改 tomcat 端口设置

注意：renren_fast工程运行的java网段，应当直接设置为服务器的宿主机的网段，因为docker禁止跨网段相互调用。（如果java环境设置为net3,之前设置的database集群为net1网段，redis集群为net2网段，则不方便调用）。然而如果要是后端程序能多节点负载均衡，则需要将不同后端代码开启不同的端口，此处修改renren_fast工程中的 Redis配置文件 application.yml。 修改如下：将端口改为 6001 
```
# Tomcat
server:
  tomcat:
    uri-encoding: UTF-8
    max-threads: 1000
    min-spare-threads: 30
  port: 6001
  connection-timeout: 5000ms
  servlet:
    context-path: /renren-fast

```

### 1.4 Maven打包部署
renren-fast包含了tomcat.jar文件，利用maven打成jar包，部署在具有java环境的docker容器中即可
```
mvn clean install -Dmaven.test.skip=true
```
这里我利用IDEA的maven 来打包

运行java容器，部署并运行JAR文件
```
docker run -it -d --name j1 -v j1:/home/soft --net=host java
nohup java -jar /home/soft/renren-fast.jar
```