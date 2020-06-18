---
title: 设计模式（三）行为型
date: 2019-10-22 20:25:40
tags:
- 设计模式
---

本文是参照 [图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/index.html)和 [设计模式](http://www.phperz.com/article/15/0814/148654.html)总结的设计模式学习系列博客（三）。

# 一. 观察者模式
## 1. 定义
- 一对多依赖关系，subject的state一旦改变，所有observers均会受到通知，做出响应
- 又叫做发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。
- 分类：
    - push模式：subject向observer推送主题的详细信息，不管观察者是否需要，推送的信息通常是主题对象的全部或部分数据。
    - pull模式：subject推送少量信息，observer接收到通知，如果需要，自行去拉取
    - 区分：update()方法传递了参数的：push。

- 应用：典型的MVC设计模式是基于 观察者模式， model是subject，view是observer，controller是两者之间的中介mediator。当model数据改变时，view将自动改变其显示内容。


## 2. 类图
![观察者模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191021220805.png)

## 3. 实例

- 类图中拉取模型的[例子](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/ObserverPattern/example)： 
- 使用jdk自带的observer包的:[例子](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/ObserverPattern/defaultClass) 

# 二.策略模式
## 1. 定义
- 不用硬编码（if...else; 或者将方法放在一个类中，用写死的方法名称调用），怎么管理“算法簇”
- 将每一个算法封装在一个类中，称一个“策略”,为了保证策略有规范，实现同一个 “策略接口”
- 官方定义：策略模式(Strategy Pattern)：定义一系列算法，将每一个算法封装起来，并让它们可以相互替换。策略模式让算法独立于使用它的客户而变化，也称为政策模式(Policy)。
- 角色：
    - Context: 环境类
    - Strategy: 抽象策略类
    - ConcreteStrategy: 具体策略类

## 2. 举例
![策略模式举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191022161431.png)

- 三种不同的策略实现自同一个接口，Context含有这个策略，但是执行方法是由策略类的实现类来控制。

    ```java
    Context context = new Context(new OperationAdd()); //这里也可以改成反射调用，根据策略名称来调用
    System.out.println("10 + 5 = " + context.executeStrategy(10, 5));
    ```

- [示例代码](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/StrategyPattern)

- 实用：
    - 策略模式包含了几个相同的策略类，可以配合工厂模式使用，利用工厂模式生产出对应的策略。
    - 如果算法是由几个小算法模块流程串联起来的，可以用将Context中保存一个策略的list,循环执行。


## 3.应用
- 诸葛亮的锦囊妙计，每一个锦囊就是一个策略。 
- 旅行的出游方式，选择骑自行车、坐汽车，每一种旅行方式都是一个策略。 
- JAVA AWT 中的 LayoutManager。

# 3. 状态模式
## 1. 定义
- 一个对象在其内部状态（一个值）改变时改变它的行为
- 省略大if...else
- 角色
    - Context 环境类：就是状态管理器(state Manager), 可以在环境类中对状态进行**切换**操作。
    - State 抽象类
    - 具体State实现类

## 2. 类图
![状态模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191023153455.png)

- Context
    - 持有State状态
    - changeState() 在环境类中可以切换状态
    - reuqest() 调用具体状态的方法
- State
    - handle() 该状态实现类根据自己的状态处理的方法

## 3. 举例
![TCP statePattern](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191023155255.png)

- 适合于状态转移的场景
- 工作流：尚未办理，正在办理，正在批示，正在审核；游戏

# 4.责任链模式
## 1. 定义

- 每个接收者都包含对另一个接收者的引用。如果一个对象不能处理该请求，那么它会把相同的请求传给下一个接收者，依此类推。
- 职责链将请求的发送者和请求的处理者解耦，请求的发送者无需关系请求处理的细节。请求被层层传递，直到可以被处理为止。
- 在处理消息的时候以过滤很多道
- 补充：
    - 责任链可能是一条直线，一个环链或者树状结构的一部分。
- 应用：
    - web中使用责任链较多，例如 Filter， 对erquest和 response的过滤。

- 示意图：
    
    ![责任链模式示意图](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/photo_2019-10-24_16-24-41.jpg)


## 2. 举例类图
![责任链模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191024163248.png)

- 链表的方式，先建立，安插好一条责任链。
- 关键：AbstractLogger的 Handler 里面聚合它自己，在 HanleRequest 里判断是否合适，如果没达到条件则向下传递，向谁传递之前 set 进去。
- 也可以用一个 chainManager 来保存 这条链，而不是元素之间 串起来。 