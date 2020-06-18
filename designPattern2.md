---
title: 设计模式（二）结构型
date: 2019-10-21 18:18:38
tags:
- 设计模式
---


本文是参照 [图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/index.html)和 [设计模式](http://www.phperz.com/article/15/0814/148654.html)总结的设计模式学习系列博客（二）。

# 一. 适配器模式
## 1. 定义
- 类似于电源适配器的设计和编码技巧被称为适配器模式
- 现有的类提供了客户想要的功能，但是接口格式不是客户想要的，这种情况下，写一个适配器类即可。
- 成员：
    - Target：目标抽象类
    - Adapter：适配器类
    - Adaptee：适配者类
    - Client：客户类
- 分类：
    - 类适配器：
    - 对象适配器
    - 缺省适配器
- 经典应用：

    Sun公司在1996年公开了Java语言的数据库连接工具JDBC，JDBC使得Java语言程序能够与数据库连接，并使用SQL语言来查询和操作数据。JDBC给出一个客户端通用的抽象接口，每一个具体数据库引擎（如SQL Server、Oracle、MySQL等）的JDBC驱动软件都是一个介于JDBC接口和数据库引擎接口之间的适配器软件。抽象的JDBC接口和各个数据库引擎API之间都需要相应的适配器软件，这就是为各个不同数据库引擎准备的驱动程序。


## 2. 类适配器
![类适配器](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191021171812.png)

- Adapter与Adaptee是继承关系，这决定了这个适配器模式是类的
- 是静态的定义方式

## 3. 对象适配器
![对象适配器](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191021172326.png)

- Adapter与Adaptee是委派关系，这决定了适配器模式是对象的
- 是动态组合的方式

## 4. 缺省适配器
![缺省适配器](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191021173028.png)

- 为一个接口提供缺省实现，这样子类型可以从这个缺省实现进行扩展，而不必从原有接口进行扩展。
- 举例：[鲁智深和尚](https://www.cnblogs.com/java-my-life/archive/2012/04/13/2442795.html)

## 5. 举例
![举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191021193707.png)

- client使用者，server提供服务。
- clientPlayer 想使用 serverPlayer的某个功能，但是接口不匹配
- 通过添加Adapter， 将serverPlayer的功能适配到 client上使用 

# 二. 装饰者模式
## 1. 定义
- 子类复子类，子类何其多
- 给一个类或者对象添加多的行为，不想用“继承”的方式，可以选择“关联”的方式
- 将一个类的对象嵌入另一个对象中，由另一个对象来决定是否调用嵌入对象的行为以便扩展自己的行为，我们称这个嵌入的对象为装饰器(Decorator)
- 成员
    - Component: 抽象构件
    - ConcreteComponent: 具体构件
    - Decorator: 抽象装饰类
    - ConcreteDecorator: 具体装饰类
- 装饰器模式强就抢在：可以级联几个装饰器，来任意组装、叠加： 例如IO流； 例如：MyBatis的缓存模块


## 3. 举例
![装饰者模式举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191022113802.png)

- 想给 Circle 的 draw方法 增加一些行为，但是不希望继承Circle，对Circle不造成影响
- 增加一个 RedShapeDecoprator, ，持有circle对象，并改写其draw方法，在draw方法中增加“红边框颜色”这个行为。
- Circle与RedShapeDecorator上层类有一些联系（shape,ShapeDecorator）保证了规范。
- [代码](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/DecoratorPattern)

# 三. 外观模式/门面模式
## 1. 定义
- 门面模式提供一个高层次的接口，使得外部不需要感知内部各种子系统的细节，通过统一的门面对象进行交互。
- 遵循了“迪米特法则”，违背了“开闭原则”。
- 拓展
    - 外观类不为子系统增加新行为
    - 遵循“迪米特法则”：一个对象应该对其他对象保持最少的了解，尽量降低类与类之间的耦合度；外观模式降低了 其他模块对于本模块子系统的耦合度
    - 违背了“开闭原则”：对新增开放，对修改关闭（子系统增加或者修改时，要改外观类），解决方式是：面向接口编程，添加接口外观类。 client程序根据接口外观类在编写，本系统子系统发生改变时，直接替换掉原有外观类即可。不会影响client程序的依赖。

## 2. 举例

![外观模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191030124155.png)

- 将子系统画形状的三个实现类，用一个ShapeMaker来统一管理，ShapeMaker持有三个实现类的对象，分发并调用对应的功能。
- 外观模式，不添加新的功能，只起到一个包装，聚合，解耦的作用。

# 四. 代理模式
## 1. 定义
- 通过聚合（而非继承）的方式，对原有的方法做增强。
- 与装饰者模式，适配器模式的区别：
    - 从意义上来说：
        - 装饰者模式： 为了对原有的类做增强
        - 代理模式  ： 为了对原有的类做增强
        - 适配器模式： 为了对原有类的接口做适配
    -  从实现上来说：
        - 装饰者模式： 通过新增类中，增加新的方法 
        - 代理模式  ： 通过新增类中，增强原有方法（一般在方法前后增加一些功能，aop）
        - 适配器模式： 通过新增类中，新增方法（是实现了client端的接口）。


## 2. 举例
![代理模式举例](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191104145910.png)

## 3. 静态代理

- 定义：代理对象和目标对象需要实现一样的接口
- 缺点：冗余：代理类过多； 不易维护：接口增加方法，目标对象和代理对象都要修改。（不符合开闭原则）

 <div id="JDK_PROXY_PATTERN">这里是指定位置</div>

## 4. JDK动态代理
### 4.1 原理：
- 定义：利用JDK提供的API, 动态地在内存中构建代理对象，从而实现对目标对象的代理功能。又称JDK代理或者接口代理。
- 优势：不需要新建代理类，而是动态构建。（减少冗余）
- 动态代理模式原理：
    - 不手动新建代理类，而是由 jdk > rt.jar > reflect包下的 两个类提供辅助功能： Proxy, InvocationHandler
    - Proxy.newProxyInstance( ): 运行时生成 实现了被代理接口的proxy类，proxy中聚合有 InvocationHandler接口的实现类。并且动态地加载到内存中。
    - 调用被代理对象，实际上调用的是 proxy对象（这里是通过多态实现，由于被代理对象和proxy对象均实现了target接口，创建对象的实际类型是 $proxy ）。 调用 target的实现类的 目标方法，实际是调用 proxy类中的的经过增强的invoke()方法，在此invoke()方法中，运用**反射**调用了 被代理对象的 目标方法。

- 动态代理的流程：
    1. Proxy.newProxyInstance( ): 创建实现了target接口的proxy对象。
    2. MyInvocationHandler.invoke( ): 调用target接口的实现类的目标方法，实际调用的是 InvocationHandler的实现类MyInvocationHandler的invoke方法。 

### 4.2 实例：
- 使用示例：[代码](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/ProxyPattern/DynamicProxy)
- 例子中的流程解析：
![动态代理模式示意图](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191104223802.png)
    - 第一步：首先动态创建myProxy类，此类中聚合了MyInvocationHandler, MyInvocationHandler中的invoke方法即是强化方法。
    - 第二步： 调用MyInvocationHandler.invoke(),会通过反射调用UserDao的真实save方法。

## 5. cglib代理（Code Generation Library）
### 5.1 原理
- 定义：对于对象行为增强的方式一般有两种，一种是聚合，一种是继承，cglib采用继承的方式，动态地生成target类的子类，来进行对象行为的增强。使用的
- 区别：
    - JDK动态代理：采用聚合的方式增强，缺点是必须有接口类。生成的类是：
    ```java
    $Proxy4
    ```
    - cglib代理：采用继承的方式增强，对代理类无侵入。生成的类是：
    
    ```java
    UserDao$$EnhancerByCGLIB$$552188b6
    ```

### 5.2 实例
- [代码](https://github.com/zhutianpeng/design-pattern-example/tree/master/src/main/java/ProxyPattern/cglibDynamicProxy)