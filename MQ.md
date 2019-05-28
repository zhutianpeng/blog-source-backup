---
title: 以activemq为例的消息中间件浅析
date: 2019-05-06 16:58:47
tags:
	- 中间件
    - activemq
---

> 1. 传统的前端向后端发起http request,后端回复http response的方式，在服务器端主动推送模型中不再适用。
> 2. 传统的单体结构的项目升级为多模块的项目之后，模块与模块之间的通信方式有所改变。  
>上述两个问题都可以用“消息中间件”的思想来解决，本文做一个基础知识的学习。并且就activemq进行一些分析和总结。

# 1. web项目中 server 向 client推送消息的方式调研
要实现 服务器端推送，以及异步处理，需要使用基于mqtt的消息中间件，或者使用websocket来进行

# 2. 消息队列的模型分类（JMS规范支持的消息模型）
## 2.1 ： 点对点（point to point， queue） vs 发布/订阅（publish/subscribe，topic）
![JMS规范支持的消息模型](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E9%97%B2%E6%95%A3/MQ_1.bmp)


点对点模型：queue，只能消费一次。

- 定义：消息生产者生产消息发送到queue中，然后消息消费者从queue中取出并且消费消息。
消息被消费以后，queue中不再有存储，Queue支持存在多个消费者，但是一个消息只能被消费一次。
- 特点：queue中不存储已被消费一次的消息。

发布/订阅：Topic，可以重复消费：

- 定义：消息生产者（发布）将消息发布到topic中，同时有多个消息消费者（订阅）消费该消息。
- 特点1：发布到topic的消息会被所有订阅者消费。
- 特点2：支持“订阅组”的发布订阅模式。当发布消息量过大时，采用多订阅节点组成一个订阅组来 负载均衡地分组订阅，可以简单地将消费能力进行线性扩展。

## 2.2 推 vs 拉
Push方式：由消息中间件主动地将消息推送给消费者；

- 优点：尽可能快地将消息发送给消费者
- 缺点：如果消费者的处理消息的能力很弱(一条消息需要很长的时间处理)，而消息中间件不断地向消费者Push消息，消费者的缓冲区可能会溢出。

Pull方式：由消费者主动向消息中间件拉取消息。

- 特点： 会增加消息的延迟，即消息到达消费者的时间有点长

举例：

- ActiveMQ： 包含“推”，“拉”两种模型，较为复杂，后面详细介绍。
- RabbitMQ： 既支持内存队列也支持持久化队列，消费端为“推”模型，消费状态和订阅关系由服务端负责维护，消息消费完后立即删除，不保留历史消息。
- Kafka: 只支持消息持久化，消费端为“拉”模型，消费状态和订阅关系由客户端端负责维护，消息消费完后不会立即删除，会保留历史消息。

# 3. activemq浅析
## 3.1 名词介绍
ActiveMQ 是一个 MOM，具体来说是一个实现了 JMS 规范的系统间远程通信的消息代理。

- MOM:
    - 面向消息中间件(Message-oriented middleware)。
    - 以分布式应用或系统中的异步、松耦合、可靠、可扩展和安全通信的一类软件。
    - MOM 的总体思想是它作为消息发送器和消息接收器之间的消息中介,这种中介提供了一个全新水平的松耦合。
- JMS：
    - Java 消息服务(Java Message Service)。java平台上有关面向 MOM 的技术规范。
    - 旨在通过提供标准的产生、发送、接收和处理消息的 API 简化企业应用的开发，类似于 JDBC 和关系型数据库通信方式的抽象。

消息队列的另外四个基本概念：分别是Provider，Domains，Connection factory，Destination。

- Provider：纯 Java 语言编写的 JMS 接口实现（eg: ActiveMQ）
- Domains：消息传递方式，包括点对点（P2P）、发布/订阅（Pub/Sub）两种
- Connection factory：客户端使用连接工厂来创建与 JMS provider 的连接
- Destination：消息被寻址、发送以及接收的对象

## 3.2 Activemq使用的通用步骤
- 获取连接工厂
- 使用连接工厂创建连接
- 启动连接
- 从连接创建会话
- 获取 Destination
- 创建 Producer
- 创建 Consumer
    - 创建 Consumer
    - 注册消息监听器（可选）
- 发送或接收 message
- 关闭资源（connection, session, producer, consumer 等)

## 3.3 Activemq的 “P2P”与“Pub/Sub”的比较
![activemq的 “P2P”与“Pub/Sub”的比较](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E9%97%B2%E6%95%A3/MQ.png)

解释：Topic和queue的区别：

- 对于消费者延迟启动，是否还有消息保留的问题： （本质上是消息发送的方式）

    - Pub/Sub（topic）： 以 广播 的形式，通知所有在线监听的客户端有新的消息，没有监听的客户端将收不到消息；
    - P2P（queue）：则是以点对点的形式通知多个处于监听状态的客户端中的一个

- 对于Broker重启时消息是否保留的问题： （本质上是持久化问题）
    - P2P:
        - 当DeliveryMode设置为NON_PERSISTENCE时，消息被保存在内存中,broker重启即消息丢失；
        - 而当DeliveryMode设置为PERSISTENCE时，消息保存在broker的相应的文件或者数据库中。broker重启不会导致消息丢失，只有P2P中消息被Consumer消费后才从broker中删除。
    - Pub/Sub模式：
        - 当DeliveryMode设置为PERSISTENCE，且有持久订阅者(Durable Subscribers)时，Broker重启后，消息会保留。

## 3.4 Activemq的 “推与拉”
### 3.4.1 ActiveMQ消息传送机制
prefetch limit:  
由来：  
- 生产者给消费者推送消息，消费者来不及消费，会将消息缓存在 缓冲区中，缓冲区可能有溢出的风险。prefetch limit 规定了一次可以向消费者Push(推送)多少条消息。当生产者发送的消息累计达到prefetch limit，但是消费者没有回复ack时，消息中间件将不再推送消息。

怎么取值：  
- 大于0: 是push的方式，允许消息的堆积
- 等于1：消费者收到一条消息，如果没有处理完，回复一个ack给中间件，就没法继续推送。适用于消息的数很少(生产者生产消息的速率不快)，但是每条消息 消费者需要很长的时间处理的情况。
- 等于0：是pull的方式了。

### 3.4.2 ActiveMQ的ack确认机制
optimizeACK:    
由来：  
 
- 可优化的ACK”，这是ActiveMQ对于consumer在消息消费时，对消息ACK的优化选项，也是consumer端最重要的优化参数之一。
- 只有开启了optimizeAcknowledge=true,后面的optimizeAcknowledgeTimeOut，prefetchSize，redelivery才有意义。
 
作用：

- 通过optimizeACK和prefetch配合，可以达成高效的消息消费模型：批量获取消息，并“延迟”确认(ACK)


### 3.4.3 通过prefetch与optimizeACK 实现高效消息消费模型
上文看到：prefetch优化了消息传送的性能，optimizeACK优化了消息确认的性能。现在进行适用的场景分析：

- （consumer消费速率 > producer生产速率） && 消息的数量也很大：
    使用optimizeACK + prefetch的消费模型可以大大提升consumer的性能。

- consumer的消费速度慢 && 消息的数量也很大:
    prefetchSize设置过大反而不利于consumer端的负载均衡。应当使用较小的prefetch，同时关闭optimizeACK，可以让消息在多个consumer间“负载均衡”(即均匀的发送给每个consumer)

- consumer的消费速度 >> producer的生产速度 && 部署了多个consumer：
    建议开启optimizeACK，但是需要设置的prefetchSize不能过大；这样可以保证每个consumer都能有”活干”，否则将会出现一个consumer非常忙碌，但是其他consumer几乎收不到消息

- 消息很重要，不愿意接受redelivery的消息:
    optimizeACK=false，prefetchSize=1，解释：当optimizeACK=true时，存在这种风险（当consumer收到消息，在延迟ack的时间内，发生了故障，则其他consumer可能会受到redelivery的消息）。所以不要延迟发送ack。而prefetchSize=1则是确保消息确认收到后，再继续推送下一条消息。

### 3.4.4 代码中如何使用 “推/拉” - “同步调用”和“异步调用”的比较
同步调用：

- 语法： ActiveMQMessageConsumer的receive()方法。
- prefetch limit取值：
    - 大于0：push
    - 等于0：pull

异步调用：

- 语法： 消费者实现MessageListener接口，监听消息。
- prefetch limit取值：只能大于0，因为是被动监听的模式：push。


## 3.5 ActiveMQ的持久化方式

## 3.6 ActiveMQ的部署模式 (向高可用进发)
1. 单例模式

2. 无共享主从模式

3. 共享存储主从模式

## 3.7 ActiveMQ的网络连接



---
参考资料

[消息队列的两种模式](https://blog.csdn.net/heyutao007/article/details/50131089)  
[ActiveMQ的推拉模型](http://www.cnblogs.com/hapjin/p/5683648.html)  
[成小胖学习ActiveMQ·基础篇](https://www.cnblogs.com/cyfonly/p/6380860.html)  
[ActiveMQ持久化方式](https://blog.csdn.net/xyw_blog/article/details/9128219)  
[ActiveMQ消息传送机制以及ACK机制详解](https://blog.csdn.net/xyw_blog/article/details/9128219)