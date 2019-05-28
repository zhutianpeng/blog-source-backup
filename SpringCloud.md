---
title: SpringCloud  
date: 2019-04-14 16:58:47
tags:
	- java
    - SpringCloud
    - 微服务
---

SpringCloud系列学习  

Spring Cloud是一个基于Spring Boot实现的云应用开发工具，它为基于JVM的云应用开发中涉及的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。   

# 一.总体把握
## 架构发展史：  
![架构发展史](http://selfstudy.oss-cn-beijing.aliyuncs.com/blog/1.%20%E6%9E%B6%E6%9E%84%E5%8F%91%E5%B1%95%E5%8F%B2.png)

- 单一应用架构：

    就一个应用，所有功能聚合在一起，以减少部署节点和成本，ORM是关键

- 垂直应用架构：

    当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。此时，用于加速前端页面开发的 Web框架（MVC）是关键。

- 分布式服务架构：

    当垂直应用逐渐增多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快的响应多变的市场需求。此时，用于提高业务复用以及整合的分布式服务框架（RPC）是关键。

    旨在支持应用程序和服务开发，可以利用物理架构由多个自制的处理元素，不共享主内存，但通过网络发送消息合作。

- 流动计算架构：

    当服务增多，容量的评估，小服务器资源的浪费等问题逐渐显现，此时需要增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。此时，用于提高机器利用率的 资源调度和治理中心（SOA）是关键。

## 组件：
- 服务注册与发现
- 服务网关：前端路由，连接内外的大门，屏蔽内部细节，路由功能，限流，容错，监控，日志
- 后端通用服务：（也称中间层服务 Middle Tier Service）
- 前端服务：(也称边缘服务 Edge Service)：聚合，裁剪后端服务，暴露。
    - 聚合： eg: 将多个API合并成一个
    - 裁剪： eg: 同一个接口，PC端调用返回得多，移动端调用返回的信息少

## 两大实现方案：
- 阿里系：
    - Dubbo: 服务化治理
    - Zookeeper: 服务注册中心
    - SpringMVC or SpringBoot：

- Spring Cloud 全家桶:
    - Spring Cloud Netflix Eureka
    - SpringBoot

    Spring Cloud 的版本：https://spring.io/projects/spring-cloud

## 组件概述

参考知乎，以订单服务、库存服务、仓储服务、积分服务为例

具体业务例如：用户针对一个订单完成支付之后，就会去找订单服务，更新订单状态。订单服务调用库存服务，完成相应功能；订单服务调用仓储服务，完成相应功能；订单服务调用积分服务，完成相应功能。

- Eureka：

    - EurekaClient：负责将这个服务的信息注册到Eureka Server中
    - Eureka Server：注册中心，里面有一个注册表，保存了各个服务所在的机器和端口号

    ![Eureka举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/2.Eureka%E4%B8%BE%E4%BE%8B.jpg)

    订单服务本地有一个Eureka的client，会去远端的Eureka server询问 需要调用的其他服务（库存、仓储、积分）的地址和端口，并且拉取到本地缓存。 之后订单服务就可以 调用 其他三种服务。

- Feign:
    - Feign使用了动态代理，Feign Client会在底层根据注解，跟指定的服务建立连接、构造请求、发起靕求、获取响应、解析响应，等等。

     ![Feign举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/3.Feign%E4%B8%BE%E4%BE%8B.jpg)

    对一个接口定义@FeignClient注解，Feign就会针对这个接口创建一个动态代理；接着你要是调用那个接口，本质就是会调用 Feign创建的动态代理，这是核心中的核心；Feign的动态代理会根据你在接口上的@RequestMapping等注解，来动态构造出你要请求的服务的地址；最后针对这个地址，发起请求、解析响应

- Ribbon:
    - 负载均衡
    - 载均衡默认使用的最经典的Round Robin轮询算法，将服务均匀的依次发送到对应的机器（1,2,3,4,1,2,3,4…）
    ![Ribbon举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/4.Ribbon%E4%B8%BE%E4%BE%8B.jpg)

    Ribbon是和Feign以及Eureka紧密协作，完成工作的。
    1. 首先Ribbon会从 Eureka Client里获取到对应的服务注册表，也就知道了所有的服务都部署在了哪些机器上，在监听哪些端口号；  
    2. 然后Ribbon就可以使用默认的Round Robin算法，从中选择一台机器；
    3. Feign就会针对这台机器，构造并发起请求  

- Hystrix:
    - Hystrix是隔离、熔断以及降级的一个框架。
    - Hystrix会有很多个小小的线程池，每个服务职能调用自己的线程池里面的线程，线程卡住了，不会影响到其他服务的线程调用。

    ![Hystrix举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/5.Hystrix%E4%B8%BE%E4%BE%8B.jpg)

    库存服务、仓储服务、积分服务分别有自己的线程池，当积分服务down掉后，不会影响库存和仓储，积分服务自己会选择做服务熔断或者服务降级，然后去故障数据库里记录数据。

- Zuul：

- 网关
- 可以做：统一的降级、限流、认证授权、安全

    像android、ios、pc前端、微信小程序、H5等等，不用去关心后端有几百个服务，就知道有一个网关，所有请求都往网关走，网关会根据请求中的一些特征，将请求转发给后端的各个服务

- 整体的架构图：

    ![整体架构图举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/6.%20%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.jpg)


## 微服务的管理学（康辉定律）【微服务的精髓】
> Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations.
—— 任何组织在设计一套系统（广义概念上的系统）时，所交付的设计方案在结构上都与该组织的沟通结构保持一致

ztp: 什么样的系统适合用微服务架构来设计，涉及到系统本身的功能，团队的人员配置与管理。

- 传统团队：将不同能力的人才从资源池里面选调出来完成项目，项目结束后释放人才资源。
- 微服务团队：人员组织是团队模式，倾向于让团队负责整个服务（模块）的生命周期，以提供更优质的服务。小团队里面要能自己处理从前端到后端，从开发到运维部署的所有事情。  

![传统团队与微服务团队](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/11.%E4%BC%A0%E7%BB%9F%E5%9B%A2%E9%98%9F%E4%B8%8E%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%9B%A2%E9%98%9F.png)  

# 二.服务的注册与发现（Eureka）

- Eureka: 属于 Spring-cloud-netflix包下，主要内容是对Netflix公司一系列开源产品的包装
- 本章节采用基于下列版本来搭建
```
spring-boot:  1.5.4.RELEASE
spring-cloud: Dalston.SR1
```
## 搭建过程

参考 程序猿DD的这篇博客

1. 使用IDEA创建一个工程Spring-Cloud-Demo

2. 创建 Eureka-server

- 新建一个Module名为 Eureka-server，注意：new -module -> spring initializr -> Cloud Discovery -> Eureka Server
- 并在pom.xml中引入相应的依赖；【注意对应的版本信息】
- 在主类中添加这个@EnableEurekaServer注解，启动一个服务注册中心提供给其他应用进行对话
- 将“服务注册中心自己作为客户端来尝试注册它自己”的功能禁掉。需要在 application.properties/application.yml 配置文件里面添加一些信息，我的application.yml如下：

```YAML
spring:
  application:
    name: eureka-server

server:
  port: 1001

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
```

- 启动工程,访问 http://localhost:1001/ ,此时没有任何服务来注册

3. 创建“服务提供方” Eureka-Client
- 创建一个基本的Spring Boot应用。命名为 eureka-client
- 在pom.xml中，加入相应配置， 大概是spring-boot的版本信息，和Spring Cloud的版本信息。
- 添加相应的接口和服务信息（这一步实际工程会有）
- 在应用主类中通过加上@EnableDiscoveryClient注解，该注解能激活Eureka中的DiscoveryClient实现
- 配置application.properties/application.yml，我的application.yml如下：
```YAML
spring:
  application:
    name: eureka-client

server:
  port: 2001

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1001/eureka/ #此处应当能找到上面定义的 server
```

- 启动服务，能观察到服务已经被注册进eureka中心  
![eureka示例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/7.eureka.png)
  
4. Eureka的高可用
- 两个Eureka互相注册

    注意在测试的时候会出现更新不及时的情况，是由于eureka的心跳检测机制引起的，重启eureka即可

    ![Eureka的高可用](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/8.euraka%E9%AB%98%E5%8F%AF%E7%94%A8.png)

- 利用IDEA,Edit configuration -> copy 一份 eureka server， 给两个server修改启动参数的 VM options 分别为 -Dserver.port=8761 和 -Dserver.port=8762。
- 启动server1前，修改其配置文件如下，并启动server1：
    ```YAML
    spring:
      application:
        name: eureka-server

    eureka:
      instance:
        hostname: localhost
      client:
        #register-with-eureka: true
        # 表示是否从Eureka Server获取注册信息，默认为true。 如果这是一个单点的 Eureka Server，不需要同步其他节  点的数据，可以设为false
        fetch-registry: true
        serviceUrl:
              defaultZone: http://localhost:8762/eureka/ # 意思是让server1在server2中注册
      server:
        #  关闭自我保护模式，只能在开发环境这样做，生产环境需要打开
        #    port: 1001
        enable-self-preservation: true
    ```

- 启动server2前，修改其配置文件如下，并启动server2：
    ```yaml
    spring:
      application:
        name: eureka-server

    eureka:
      instance:
        hostname: localhost
      client:
        #    register-with-eureka: true
        # 表示是否从Eureka Server获取注册信息，默认为true。 如果这是一个单点的 Eureka Server，不需要同步其他节  点的数据，可以设为false
        fetch-registry: true
        serviceUrl:
              defaultZone: http://localhost:8761/eureka/  # 意思是让server2在server1中注册
      server:
        #  关闭自我保护模式，只能在开发环境这样做，生产环境需要打开
        #    port: 1001
        enable-self-preservation: true
    ```

- eureka-client照常启动即可，注意只需要在一个server1中注册，即可将注册信息同步到server2；

    挂掉server1,发现client仍然在server2上面注册存在；

    但是如果server1挂掉了，再重启client,则会发现server2上的client也不复存在，所以最保险的做法应该是如下的配置：(配置多台server的地址)
    ```yaml
    spring:
      application:
        name: eureka-client

    server:
      port: 2001

    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/,http://localhost:8762/eureka/
    ```

- 大集群三个eureka的相互注册，延伸一下即可,server之间两两相互注册，并且client注册三个server

    ![三个eureka的相互注册](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/9.eureka%E9%AB%98%E5%8F%AF%E7%94%A8.png)

## 服务注册的一些理解

1. 为什么要在分布式系统中使用 “服务发现”：  

    ![为什么要在分布式系统中使用 “服务发现”](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/10.eureka%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0.png)

随着集群中机器数量的增加，B的数量过多，不适合用写配置文件的方式来记录

2. 服务发现的两种分类（需要权衡）：

- 客户端发现：

    - 由A发起，去注册中心找到一个B，然后通过ip地址调用B。关键点在于：使用某种“负载均衡”的机制来选择提供服务的一台机器， 例如： 轮训，随机，哈希等。
    - 优点：简单，不需要代理的介入，client知道所有服务提供方的地址。
    - 缺点：client需要自己实现一套逻辑来挑选服务提供方。
    - 举例： Eureka

- 服务端发现：

    - 增加一个新的角色：代理。 代理帮助A从众多可用的B中找到一个。
    - 优点：B对A是不可见的，A只需要访问代理即可。
    - 举例： Nginx, Dubbo系的Zookeeper, Kubernetes(k8s：集群中的每一个节点都运行一个代理，来实现服务发现的功能)

3. pringCloud的服务调用方式（不同语言怎么在eureka中注册）

    eureka允许其他语言被纳入到他的服务治理体系中去，具体是这种语言需要实现eureka的客户端程序，例如 node.js实现了eureka-js-client。