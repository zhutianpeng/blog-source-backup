---
title: Dubbo
date: 2019-03-23 16:58:47
tags:
	- dubbo
    - 微服务
---

> Apache Dubbo™ (incubating)是一款高性能Java RPC框架。具有面向接口的“远程方法调用”，“智能容错和负载均衡”，以及“服务自动注册和发现”的三大功能。本文是基于慕课网的 2小时实战Apache顶级项目-RPC框架Dubbo分布式服务调度 的学习笔记。

# 1.Dubbo基础知识
## 1.架构
![Dubbo的架构](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/dubbo/1.%20dubbo%E6%A0%B8%E5%BF%83%E6%9E%B6%E6%9E%84.png)

## 2. Dubbo支持的通信协议
推荐使用：

- RPC协议/Dubbo协议：同构项目（同样结构，搭建框架相同）
- Http协议的Rest Api（走Json序列化）：通用项目

Dubbo支持8种通信协议，分别是

- 1、dubbo 协议 (默认)
- 2、rmi 协议
- 3、hessian 协议
- 4、http 协议
- 5、webservice 协议
- 6、thrift 协议
- 7、memcached 协议
- 8、redis 协议

## 3. 利用dubbo在具体的业务场景中需要考虑的相关知识：
- 服务拆分：eg:商品服务；用户商城；订单服务；支付服务。
- 服务解耦：明确职责、服务调度、网络通信。
- 服务管理：统一注册中心配置管理（发布注册；订阅调度）。

# 2. 搭建基于Dubbo+Zookeeper的SpringBoot项目Demo
## 1. 搭建基本的工程：
不做详细介绍，只讲基本步骤：

- 利用IDEA创建SpringBoot工程，引入相关依赖（Zookeeper依赖，dubbo依赖）
- 创建两大dubbo的配置文件
    - Spring-dubbo.xml:
        - 注解发布的dubbo服务所在的包
        - 配置支持的两种调用方式对应的协议 RPC,http
        - 消费服务配置
    - dubbo.properties:
        - 配置两大协议对应的端口信息
- 下载Zookeeper，需要启动 ZKserver
- 运行项目

项目结构：

![工程结构：](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/dubbo/2.%20%E6%90%AD%E5%BB%BA%E5%BE%AE%E6%9C%8D%E5%8A%A1SpringBoot%E5%A4%9A%E6%A8%A1%E5%9D%97%E9%A1%B9%E7%9B%AE.png)

## 2. 具体的应用场景实战-基于RPC/Dubbo协议的调用方式

1. 场景的需求分析： （商品列表服务）  
![场景的需求分析](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/dubbo/3.%E6%9C%8D%E5%8A%A1%E5%9C%BA%E6%99%AF%E5%88%86%E6%9E%90.png)

2. 实现思路：

    1. 创建 dubboOne项目（商品库存服务），按照上述描述，引入依赖，创建dubbo两大配置文件，并且实现一些商品列表服务接口，等待调用。
    2. 创建 dubboTwo项目（商城平台服务），按照上述描述，引入依赖，创建dubbo两大配置文件。
    3. 将 dubboOne项目 mvn clean > mvn install 到本地的maven仓库，dubboTwo项目的pom.xml文件则可引入dubboOne的api模块的这个jar包。作为服务间通信的规范和桥梁。如果修改了dubboOne的接口，需要重新mvn clean > mvn install。 （mvn clean 删除本地maven库里的包， mvn install 将这个dubboOne聚合项目生成的jar包 install到本地的maven库里面，dubboTwo项目才能找到）
    ```xml
    <!--dubboTwo引入了dubboOne发布的api-->
    <dependency>
        <groupId>com.debug.mooc.dubbo.one</groupId>
        <artifactId>api</artifactId>
        <version>1.0.1</version>
    </dependency>
    ```
启动 zookeeper的server，运行 dubboOne(provider)，运行dubboTwo(consumer) , 调用dubboTwo 的controller中的接口。

3. 注意：

- 基于RPC的调用方式，在dubboTwo项目中调用 dubboOne提供的接口，需要dubboTwo引入dubboOne的api模板的jar包。dubboTwo配置dubbo相关信息后，经过Spring的自动装配，犹如本地调用一般自然。
- entity都要implements Serializble， 因为走网络通信需要实现序列化。

## 3. 具体的应用场景实战-基于http协议的调用方式
1. 场景的需求分析 （用户下单服务）   

![场景的需求分析 （用户下单服务）](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/dubbo/4.restFul%20%E8%B0%83%E7%94%A8%E6%96%B9%E5%BC%8F%E7%9A%84%E5%9C%BA%E6%99%AF%E5%88%86%E6%9E%90.png)
