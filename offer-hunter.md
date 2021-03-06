---
title: 求生之路
date: 2019-03-12 16:58:47
tags:
    - 面试
encrypt: true
enc_pwd: ztp1017
---

# Tencent_2019.3.12_person_round1_fail
1. 算法题
Given a collection of intervals, merge all overlapping intervals.  
```
Example:
Input: [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]
Explanation: Since intervals [1,3] and [2,6] overlaps, merge them into [1,6].
```

来源 [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/)   
答案：  
```java
public static List<Interval> merge(List<Interval> intervals) {
    List<Interval> result = new ArrayList<Interval>();
    Collections.sort(intervals, new Comparator<Interval>() {
        @Override
        public int compare(Interval o1, Interval o2) {
            if(o1.start<o2.start){
                return -1;
            }else if(o1.start==o2.start){
                return 0;
            }else{
                return 1;
            }
        }
    });
    for(int i=0,j=0;i<intervals.size();i++){
        if(result.size()==0){
            result.add(intervals.get(i));
        }else{
            Interval temp=overlap(result.get(j),intervals.get(i));
            if(temp==null){
                result.add(intervals.get(i));
                j++;
            }else{
                result.remove(j);
                result.add(temp);
            }
        }
    }
    return result;
}

public static Interval overlap(Interval o1, Interval o2){
    if(o2.start>o1.end){
        return null;
    }else if(o1.end>=o2.end){
        return o1;
    }else{
        return new Interval(o1.start,o2.end);
    }
}
```

2. 进程间切换，操作系统发生了哪些步骤
3. 进程间通信的方式
4. java里面继承和实现的区别，细节  

    继承：  
    - 只能继承一个父类。 （如果能继承两个父类，会出现“致命的钻石问题”）
    - 子类拥有父类非 private 的属性、方法; 子类可以重写父类的方法。
    - 静态方法：

    实现：
    - 可以实现多个接口。
    - 实现类必须实现接口的所有方法。  


5. 数据库的三大范式，细节解释  

# 360_2019.3.20_person_round3_AC
1. 项目相关：  
- 能力开放平台，鉴权；如何实现正常调用程序（而非测试环节）；计费。

2. jvm：
- java内存模型
- java垃圾收集：判断对象是否被回收，类的卸载，垃圾收集算法，垃圾收集器，G1垃圾收集器

# Alibaba_新零售平台事业部_2019.3.22_phone_round1_AC
1. 自我介绍  

2. 项目相关：

- 背景、解决的问题、系统的架构设计、模块分层

- redis：1. 为什么选取redis而不选择其他 消息中间件； 2. MySQL数据库与Redis缓存数据一致性问题
```
问题1：选取redis而不选择其他 消息中间件

 优点：
    1. 将redis 当做轻量级的队列服务使用，与RabbitMQ相比：
        对于RabbitMQ和Redis的入队和出队操作，各执行100万次，每10万次记录一次执行时间。
        测试数据分为128Bytes、512Bytes、1K和10K四个不同大小的数据。实验表明：
        入队时，当数据比较小时Redis的性能要高于RabbitMQ，而如果数据大小超过了10K，Redis则慢的无法忍受；
        出队时，无论数据大小，Redis都表现出非常好的性能，而RabbitMQ的出队性能则远低于Redis。

    2. redis 消息推送是基于分布式 pub/sub，多用于实时性较高的消息推送，并不保证可靠。
        其他的mq和kafka保证可靠但有一些延迟（非实时系统没有保证延迟）

 缺点：
    1. redis 消息推送不可靠，redis-pub/sub断电就清空。
    2. redis-pub/sub没有提供“分组”这个概念, 与kafka相比，
        比如kafka中发布一个东西，多个订阅者可以分组，同一个组里只有一个订阅者会收到该消息，
        这样可以用作负载均衡

问题2：MySQL数据库与Redis缓存数据一致性问题：

  解决方式：
    增：写数据只写db
    改：更新数据先更新db，再失效cache
    查：读数据，先读cache，未命中读db，写入cache
```

3. 基础知识：   
- 解决hash冲突的方法：拉链法，rehash:
```
1.『hash』: 翻译为“散列”，就是把任意长度的输入，通过散列算法，变成固定长度的输出，该输出就是散列值。
这种转换是一种压缩映射，散列值的空间通常远小于输入的空间，不同的输入可能会散列成相同的输出，所以不可能从散列值来唯一的确定输入值。

2. 『hash冲突』：根据key即经过一个函数f(key)得到的结果的作为地址去存放当前的key value键值对(这个是hashmap的存值方式)，但是却发现算出来的地址上已经有人先来了。

3. 『解决hash冲突的办法』：
开放地址法：(open addressing)
    线性探测法(linear probing)、平方探测法(Quadratic probing)、双散列算法(double hashing)、再散列（rehashing）
分离链表法：（Sperate Chaining）【拉链法】
    将相应位置上冲突的所有关检测存储在同一个单链表中
```

- TCP的可靠性
```
1. 『差错控制』:
TCP通过 校验和 、 确认 以及 超时重传 这三个工具，来检测和重传受到损伤的报文段、并重传丢失的报文段、保存失序到底的报文段直至缺失的报文段到期，以及检测和丢弃重复的报文段。

2. 『流量控制』:
TCP通过接收窗口的大小来调节生产者产生数据的速度和消费者消耗数据的数据，达到一种动态平衡。

3.『拥塞控制』:
慢开始、拥塞避免、快重传、快恢复
```

- java的类加载机制，一个应用从启动到运行的机制
```
虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。

类从被虚拟机加载到内存中开始，到卸载出内存为止，它的生命周期经历了
    加载（Loading）、
    连接
        验证（Verification）、
        准备（Preparation）、
        解析（Resolution）、
    初始化（Initialization）、
    使用（Using）
    卸载（Unloading）
```

- java的内存管理
```
Java 的内存管理就是对象的分配和释放问题。（两部分）

1. 对象的分配与存储：
    1.1 JVM 的内存区域组成
    1.2 各种数据类型在java中的存储：
        在方法中声明的变量可以是基本类型的变量，也可以是引用类型的变量。
         （1）当声明是基本类型的变量的时，其变量名及值（变量名及值是两个概念）是放在JAVA虚拟机栈中
         （2）当声明的是引用变量时，所声明的变量（该变量实际上是在方法中存储的是内存地址值）是放在JAVA虚拟机的栈中，该变量所指向的对象是放在堆类存中的。

        在类中声明的变量是成员变量，也叫全局变量，放在堆中的：
        （1）当声明的是基本类型的变量其变量名及其值放在堆内存中的
        （2）引用类型时，其声明的变量仍然会存储一个内存地址值，该内存地址值指向所引用的对象。引用变量名和对应的对象仍然存储在相应的堆中。

2. 对象的释放（GC）:
	2.1 如何判断对象应该被回收:
		引用计数算法；可达性分析算法；方法区域的回收。
	2.2 怎么回收（垃圾收集的4种算法）：
		标记-清除；标记-整理；复制；分代收集
	2.3 7种垃圾收集器
		Serial,ParNew,Parallel Scavenge, Serial Old, Parallel Old, CMS, G1
	2.4 内存分配与回收策略
```

- java OOM 异常处理 （OutOfMemoryError）
```
1. Java堆溢出
    Java堆用于存储对象实例，只要不断地创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，对象数量达到最大堆容量限制，则发生溢出。(有点类似于“内存泄露”)
  
2. 虚拟机栈和本地方法栈溢出
    StackOverFlow：线程申请的栈深度超过允许的最大深度
    OutOfMemoryError： 虚拟机扩展时无法申请到足够的内存空间

3. 方法区和运行常量池溢出
    方法区用于存放Class的相关信息，如果运行时产生大量的类去填满方法区，就可能发生方法区的内存溢出。
    例如主流框架Spring、Hibernate对大量的类进行增强时，利用CGLib字节码生成动态类；
    大量JSP或动态JSP(JSP第一次运行时需要编译为Java类）。

    运行时常量池：JDK7之前会有，不做研究。

4. 本地直接内存溢出 
    通过Unsafe类直接分配内存时，过大则会溢出。
```

详见：[java OOM 异常处理](http://ztxpp.cc/2019/03/23/java-OOM/)

4. 你最近在学习什么新技术？在看什么书？
```
技术层面：深入理解JVM虚拟机
架构层面：微服务
```

# Alibaba_新零售平台事业部_2019.3.28_phone_round2_AC
1. Java 代码编译和执行的整个过程； 类加载机制；
![java编译过程](http://wiki.jikexueyuan.com/project/java-vm/images/javadebug.gif)
![java字节码的执行](http://wiki.jikexueyuan.com/project/java-vm/images/jvmdebug.gif)

```
Java 代码编译和执行的整个过程,包含以下三个重要的机制：
 1. Java源码编译机制：
                    分析和输入到符号表
                    注解处理
                    语义分析和生成 class 文件
 2. 类加载机制：
                    双亲委派模型
 3. 类执行机制：
                    JVM 是基于栈的体系结构来执行 class 字节码的。
                    线程创建后，都会产生程序计数器（PC）和栈（Stack），
                    （1）程序计数器存放下一条要执行的指令在方法内的偏移量，
                    （2）栈中存放一个个栈帧，每个栈帧对应着每个方法的每次调用，而栈帧又是有局部变量区和操作数栈两部分组成，局部变量区用于存放方法中的局部变量和参数，操作数栈中用于存放方法执行过程中产生的中间结果。
```

2. 两个类，类和包名完全相同，应该从classLoader上面怎么处理
```
当项目中同时出现两个相同的名称的 类（包名、类名都相同），且 两个jar包都要引用，不能删除某个jar包的import：
    可以利用编译器修改引入 jar包的顺序,例如用IDEA, project structure -> modules -> Dependencies,手动修改顺序。
```

3. 七层协议，每个有什么用；物理层有什么协议；wifi是那一层的协议；
```
(1) 物理层：传送数据的单位是比特。物理层的任务就是透明地传送比特流。

(2) 数据链路层（链路层）：将网络层交下来的 IP 数据报组装成帧，在两个相邻结点（主机和路由器，或两个路由器）之间的链路上“透明”地传送帧中的数据。每一帧包括数据和必要的控制信息（如同步信息、地址信息、差错控制等）。

(3) 网络层：使用无连接的网际协议 IP 和许多种路由选择协议。负责为分组交换网上的不同主机提供通信服务，把运输层产生的报文段或用户数据报封装成分组（也叫IP数据报或数据报）或包进行传送。网络层的另一个任务就是选择合适的路由

(4) 传输层：向两个主机中进程之间的通信提供服务。

(5) 应用层：直接为用户的应用进程提供服务，如 HTTP、支持文件传输的 FTP 协议等
```

4. 加密算法有什么了解?

5. 数据库事务，怎么保障的，全盘通讲。
6. 数据库索引的数据结构；多列索引
7. B+树 和 B树 相比有什么优点；
8. 平衡树 都有哪些？
9. Spring IOC, AOP
10. 项目里面为啥使用redis
11. servlet 与 filter， 流 有什么共同点
12. 设计模式 了解哪些
13. 算法方面，深度学习了解多少
14. 学科专业上的安排，毕设的情况

# Amazon_SDE(Software Develop Engineer)_2019.3.29_round1&2
## round_1 (mentor面)
1. 系统设计 （商品的推荐商品），这个功能怎么设计，细化到数据库的表结构

2. 一个印象最深刻的项目（设计模式）

3. 项目中 一个与老师之间battle的例子，细节

## round_2 (基础面)
1. 讲比赛的项目（天池）

2. 两道算法题
- Symmetric tree： 题目和解答
- 博物馆的max人数的时刻：
```
题目：给定array[]长度为1000,  存的 <a,b>，意义为 a:某人进入博物馆的时刻，b 此人离开博物馆的时刻，问哪些时刻博物馆的人最多。
```

## Amazon_SDE(Software Develop Engineer)_2019.4.2_offerCall_accept

# 360_2019.4.4_IT架构中心java开发_offerCall_reject

# Alibaba_新零售平台事业部_2019.4.4_phone_round3_AC
1. 自我介绍（着重介绍技术项目）

2. 流控（redis）：

- 使用滑动窗口的具体业务需求是什么？
- 三年之后的业务量大概是多少？
- 假如 api的数量到达很大的级别（千万级），redis的性能怎么保证，怎么解决？  

3. 权限控制 表级别的权限控制

4. 协议转换 代码细节

5. 程序里面发生了很多 full GC， 你该怎么解决？

6. 流控的时候，redis的并发安全问题？内存的并发安全问题？

7. 你的能力开放平台系统，压力测试情况，系统测试情况？

# 阿里hr面准备：
1. 做项目的启发和经历：
- 国家电网的项目，系统的设计，微服务的思想，设计模式等等。重设计，轻实现。
2. 流控和协议转换，是自己做的么？

3. 对阿里的看法：  
- 让天下没有难做的生意，让天下没有难做的算法和应用。
- 一流的企业做标准，二流三流才是做文化.品牌.解决方案。阿里有完整的java的生态体系，和自己的jvm，从几轮面试中也能看出来。阿里和我自己的技术展非常契合，我在前几年的确是想好好来阿里学习、钻研技术的。
  
4. 一个最困难的事情，怎么解决的：
- 系统架构的设计；能力开发平台的加班；线上线下的联调；
5. 对自己的职业发展的规划：
- 前几年先在大公司好好扎实做技术，然后做技术深入或者，转管理岗位。
6. 我的缺点;我的优点：
- 缺点：有时候太细致，较真儿。可能会拖慢项目进度。总喜欢最后check一下。不确定正确的事情也不敢下定论。

7. 你最自豪的一件事情：

8. 其他家公司的offer已有的：

9. 还在面试其他家公司吗？

10. 其他家的公司和阿里比较，会选择阿里吗？

11. 可以选择的工作地点？

12. 996怎么看待？   

# 阿里hr面试_2019.4.10：
1. 能接受在杭州工作吗？

2. 女朋友有吗？女朋友的工作地点？

3. 你影响最深刻的一个项目，项目的背景，你在当中承担的职责，项目中你解决的最大的一个难题。体现出你的什么品质？

4. 你的优点，缺点

# 京东1面_onsite_2019.9.1:
主要问项目

# 京东2面_onsite_2019.9.4:
面试官让从 数据库、数据结构、操作系统、网络这四个部分选择一个，我选的数据库。

1. 数据库索引
```mysql
给mysql数据库的一张表，设置（a,b,c）三列的联合索引，问下列哪种情况能够用到这个索引？
1. select * from table where a = 1 and b = 1 and c = 1;
2. select * from table where b = 1 and c = 1;
3. select * from table where c = 1;
4. select * from table where a LIKE 'com%';
5. select * from table where a LIKE '%com';
6. select * from table where md5(a) = 1;
7. select * from table where a = now(); 
```

答案：
```
1. 用到   2，3 没有     // 索引最左前缀匹配，原理上是符合排序，有a即用到。 
4. 用到   5.没有用      // LIKE模糊查询，右模糊用到了a的索引，左模糊就不知道怎么往b+树里面找了
6. 没有   7.用到        // 对左侧做函数操作用不了索引，对右侧做函数操作，能用索引，本质上还是在b+树里面能不能找到
```

2. 设计数据库结构，存储 “组织架构”（树）, 满足下面三个功能，并说出算法复杂度：

![](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190905163425.png)

```
1. 能利用ID进行CRUD操作
2. 找到一个节点的全路径
3. 找到一个节点的下属所有节点
```

![](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/9fb03b0fb1f09c2813910bee0518c37.png)

回答：
```
法1.第一反应
1.  O(1)的查找，增加删除节点要满足一定的约束条件
2.  O(n)的递归方式查找： 这里指的需要n次查询
3.  O(n)的方式循环查找： 这里指的需要n次查询
```
```
法2.经过提示，数据库里面可以存更多信息，即保存路径
1. 增加节点的时候，需要先查出父节点的parent,添加到自己的parent上，O(1)
2. O(1) 找一个节点的全路径
3. O(1) 找到一个节点的所有子节点，一次查询可以知道结果
eg: select * from table where parent LIKE A%;
```

3. 上十亿级别的数据，怎么做多列筛选（待整理）

https://segmentfault.com/a/1190000017180119

主要方案：

- 更换存储引擎，选用列式存储，hybridDB之类。
- 采用缓存，记录中间结果。
- 采用计算中间表，来进行保存结果。

# 京东hr面_他人题目_准备
1. 自我介绍

2. 个人目前面试实习情况

- 目前offer有阿里，面试中有京东；


3. why 京东 not others

- 个人观感，更喜欢在京东上买东西，更快，更正规。
- 京东优势：
    - 自营型电商更有优势，阿里更多的是平台型电商
    - 最好最快的快递：物流体系更加健全。
    - 只卖真货：对于库存的把控更加严格
- 当马云轻轻松松赚的盆钵满盈、走上“神坛”、呼风唤雨之时，刘强东还在苦逼兮兮地和快递员们称兄道弟、打成一片。问题是，羊毛不会出在狗身上，马云虽富，却是建立在数百万商家挣扎生存的前提下；刘强东虽穷，但效率未必就不能致胜。在零售业的链条上，京东和阿里都各占一环，到底谁的模式才是更合理的资源配置模式？探讨这个问题需要对产业链的上下游进行综合分析。

- 京东供应链部门：复杂性更高，更加成熟，sku数更多



4. 个人优势

- 怎么体现出很match
    - 开发项目经历
    - 获奖经历
    - 更喜欢了解业务

5. 京东企业价值观
- 使命：科技引领生活
- 核心价值观：
    - 客户为先、诚信、 协作
    - 感恩 、拼搏 、担当
- 愿景
    - 成为全球最值得信赖的企业

6. 个人诚信自我评分
 
 95分

7. 有什么问题

地点问题，换地点，换岗位的相关问题

# 京东hr面_onsite_2019.9.6
1. 名字的含义

2. 自我介绍

3. 讲讲阿里的经历，对自我的提升

4. 对于互联网的负面性怎么看？人员流动性大，压力大，怎么看？

5. 身边的人怎么评价你？

# 拼多多1面_电话+视频_2019.9.17(耻辱之战)

1. 数据库与消息队列，怎么保证数据正确读取时发送消息；数据存取失败时，不发送消息

2. HashMap 和 linkedHashMap的底层数据结构； hashmap线程安全怎么保证； hashmap实现LRU算法； LinkedHashMap 的LRU算法怎么保证线程安全？

3. redis在进行并发操作的时候怎么保证线程安全

4. 数据库索引

5. 多路归并算法

6. 讲一下微服务

7. 怎么实现论坛的回帖数在多线程情况下不重复
```
1. mysql层面： 采用 for update 操作，保证读取时，具有排它锁

2. redis层面： 采用incr 原子性操作，设计一个分布式锁。
```

# 字节跳动_北京_商业变现部门_1面_onsite_2019.9.23
1. java反射，Spring IOC
2. 线程与进程区别，用户态与内核态
3. JVM：垃圾回收机制，垃圾回收器，java内存模型
4. TCP四次挥手，time wait的作用，导致的问题； 
5. TCP的拥塞控制细节（四个过程）；慢开始为什么是指数型增长？
6. HTTP的状态码分类；
7. 数据结构hash表，b+树讲解；hashmap的效率当一个节点下面的链表过长怎么优化。
8. 讲一下rocketMQ，消息中间件
9. 微服务系统之间三种通信方式的优缺点，RPC,HTTP,MQ
10. 讲一下RPC为什么比较快 > 序列化
11. 讲一下项目中的dstunnel
12. 算法题：将 ../a/b/c/./d/../e/../f 优化成为 a/b/c/f 

# 字节跳动_北京_商业变现部门_2面_onsite_2019.9.23
1. 讲一下数据库索引，b树与b+树的区别
2. 数据库优化优化查询
3. 阿里的实习给你带来什么收获，层次不高面试官不开心
4. 算法题：归并两个有序链表