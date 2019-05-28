---
title: mysql 优化
date: 2019-03-23 16:58:47
tags:
	- java
    - mysql
---

MYSQL相关

# 1. Mysql索引（InnerDB）
## 1. 创建索引

```mysql> create table  information(
    -> inf_id  int(11)  auto_increment  primary  key  not  null,
    -> name  varchar(50)  not  null,
    -> sex  varchar(5)  not null,
    -> birthday  varchar(50)  not  null,
    -> index  info(name,sex)
    -> );
```   
详见 https://blog.csdn.net/qq_41573234/article/details/80250279

## 2. 索引的概念
### 2.1 意义：
想使用某种快速查找的算法，这前提必须是建立在某种有规律的特定的数据结构之上的。我们创建索引的过程，就是创建为了实现快速查找算法所必须的数据结构的过程。而在mysql中，想使用索引实现快速查找，可以简单理解为：必须要求索引的数据是按顺序排列的。

### 2.2 实现方式
Mysql中，索引由 B+ Tree 实现，不用红黑树的原因：

- 更少的查找次数： 平衡树的查找深度与树高相关, B+树的出度更大，红黑树出度=2
- 利用了磁盘的预读特性，数据库系统将索引的一个节点大小设置为页的大小，一次I/O就能完全载入一个节点。
## 3. 优化查询
### 3.1 单列索引
算法成本：二叉树查找法，通过有规律的数据结构，快速定位到某个数据，比全表扫描快。使用索引之后，在一般情况下，无论是IO成本还是计算查找成本都远低于全表扫描。

### 3.2 多列索引（最左前缀查找）
- 多列索引的本质是：对表重新进行 复合排序。

- 最佳设计是：按照索引的第一个元素进行二分查找，再按照 索引的 第二个元素进行二分查找。

- 符合最佳设计的复杂索引，要比 单列索引 查找快 （前者是多次二分查找，后者是先二分查找出所有，再扫描找）

- mysql查询优化器会判断纠正sql语句该以什么样的顺序执行效率最高，最后才生成真正的执行计划。

例如 index info(name,cid), 会按照 name先排序，name 相同的情况下，再按照 cid 排序。
```
使用到索引来查找：
EXPLAIN SELECT * FROM student WHERE name='小红';

EXPLAIN SELECT * FROM student WHERE cid=1 AND name='小红';
等价于
EXPLAIN SELECT * FROM student WHERE name='小红' AND cid=1;


没有使用索引查找：
EXPLAIN SELECT * FROM student WHERE cid=1;
```

# 2. MYSQL优化
## 2.1 学会查看sql语句的执行的各项性能消耗
在MySQL数据库中，可以通过配置profiling参数来启用SQL剖析。但是这个功能默认是关闭的。

- 查看是否开启：
    ```
    show variables like "%pro%";
    ```

- 使用命令开启：
    ```
    set profiling =1;
    ```

- 开启后，执行sql语句，然后使用命令查看执行过的语句的耗时：
    ```
    select * from stu;
    show profiles;
    ```

- 查看一个具体语句的详细情况,可以查看该sql语句每执行步骤的cpu、 io、 memory 等消耗情况：
    ```
    show profile for query 3;
    show profile cpu,block io for query 4;
    ```

## 2.2 多表查询的优化
```
select * from  stu  where  class_id  in (select id from class);

select * from stu join class on stu.class_id = class.id;
```

1. 比较两种连表查询：
- in: 需要查询的信息只来源于一张表 （eg: stu）
- join：需要查询的信息来自于多张表（eg: stu,class）
2. 真实的执行过程
- in：mysql会把in子查询转换成exists相关子查询，所以它实际等同于这条sql语句：
    ```
    select * from stu where exists(select id from class where stu.class_id=class.id );
    ```
    而exists相关子查询的执行原理是: 循环取出外表的每一条记录与子查询中的表进行比较，比较的条件是stu.class_id=class.id 然后看外表的每条记录的class_id是否在内表的id字段存在，如果存在就行返回外表的这条记录。

    弊端：exists 查询的 外表没有用到索引，用了全表扫描。

- join: 是 exists的优化

```
select * from stu inner join class on stu.class_id=class.id; 
select * from stu left join class on stu.class_id=class.id; 
select * from stu right join class on stu.class_id=class.id;
```

结果上的比较：

- inner join: 筛选出两张表中都符合on条件的行；
- left join：左表数据全部选出，右表能匹配就匹配上，不能匹配就填null
- right join：右表数据全部选出，左表能匹配就匹配上，不能匹配就填null

执行上的比较：（前提是on后面的字段配了索引）

- left join： 左表全表扫描列出每一行，去右表用索引查找，看是否满足on条件：nlogn
- right join： 右表全表扫描列出每一行，去左表用索引查找，看是否满足on条件：nlogn
- inner join： mysql查询优化器,会自己评估使用a表查b表的效率高还是b表查a表高，如果两个表都建有索引的情况下，mysql同样会评估使用a表条件字段上的索引效率高还是b表
 
3. 连表查询的优化方式：

一般而言，需要连表的字段都需要加上索引，把两个表的连接条件的两个字段都各自建立上索引，然后explain 一下，查看执行计划，看mysql到底利用了哪个索引，最后再把没有使用索引的表的字段索引给去掉就行了。

4. 关于：“阿里程序规约表示，最好不要使用超过 3张表 的连表查询” 的 解读：

- 数据库优化器，会根据连表的多少决定相应的算法选择相应的算法：

    动态规划、贪心算法。在连接表数比较少的时候选择动态规划，一阶段一阶段的去分析各种可能得连接顺序得到一    个最优的执行计划，但动态规划场景下，随表的增加，计划也会爆炸式增加，优化器在选择最优计划的前提下，消    耗的内存和CPU是不能不考虑的，所以当表的数量太多，数据库会退化成贪心算法。

- 轻DB重应用的思路：

    从阿里的角度而言，数据库服务也从垂直拓展的集中式架构变成了水平拓展的分布式架构。由于数据规模太大，不得不考虑 “分库分表+中间件” 的模型。在分库分表场景下，能在数据库层面做join的场景自然也不多，所以大家 更多的是将数据库当成一个带多行事务能力的KV系统去用，这是轻DB重应用的思路，跟银行等传统行业重DB轻应   用的思路刚好相反；然而基于“中间件”来做join，中间件拿到数据sharding信息更难，成本肯定更大，所以这个 问题自然就落到了应用上。

    链接：https://www.zhihu.com/question/56236190/answer/250862637

5. 一个例子：

```
select * from
    a inner join b on a.uid=b.uid 
    inner join c on a.shopid=c.id 
    where a.code=0 and a.status like '%未发货%'
    and b.auto_user=0 order by a.time;
```

其中：
a 表一百六十万多条数据，数据量最大。
b 表两千多条。
c 表十五万条。

优化的原则是：

- 尽量使用索引
- 不要全表遍历大表

具体的方法是：

- 查b连a(遍历b小表，去a大表里面用索引查询，具体方法是给a的uid加索引)，得到一张结果表d
- 查d连c(遍历d表小表，去c大表里面用索引查询，具体方法是给c的id加索引，不用加，c的主键id就是索引)

## 2.3 Group by 和 临时表的优化
1. group by 执行原理：
- 首先mysql会把最终需要分组的结果集提取出来作为一个临时的表存放到内存空间。
- 对该临时表进行排序
- 排序之后进行分组

2. 优化方案：

    对该字段加上索引，则不需要产生临时表，也不需要重新排序了。因为索引已经排序了。

3. 其他统计函数(min，max)

```sql
select max(age) from stu group by class_id;
```

这个 max的操作也会用到 临时文件和额外的排序。解决的方式是 建立 index（class_id,age）。解释： 会按照class_id和age来进行 复合排序，（在class_id相同的情况下会再对age来排序），所以不会生成中间表，也不需要再额外的排序。


## 2.4 其他优化方式
### 2.4.1 避免 SELECT *
从数据库里读出越多的数据，那么查询就会变得越慢。并且，如果你的数据库服务器和WEB服务器是两台独立的服务器的话，这还会增加网络传输的负载。

### 2.4.2 垂直分割
“垂直分割”是一种把数据库中的表按列变成几张表的方法，这样可以降低表的复杂度和字段的数目，从而达到优化的目的。

例如： 在Users表中有一个字段是家庭地址，这个字段是可选字段，相比起，而且你在数据库操作的时候除了个人信息外，你并不需要经常读取或是改写这个字段。那么，为什么不把他放到另外一张表中呢？ 这样会让你的表有更好的性能。对于用户表来说，只有用户ID，用户名，口令，用户角色等会被经常使用。小一点的表总是会有好的性能。

### 2.4.3 拆分大的 DELETE 或 INSERT 语句
原因：DELETE和INSERT操作会进行“锁表”，如果表锁上一段时间，比如30秒钟，那么对于一个有很高访问量的站点来说，这30秒所积累的访问进程/线程，数据库链接，打开的文件数，可能不仅仅会让WEB服务Crash，还可能会让整台服务器马上挂了。

解决方案： 使用LIMIT限定条数

### 2.4.4 越小的列会越快
对于大多数的数据库引擎来说，硬盘操作可能是最重大的瓶颈。但是做下面的操作的时候请注意可扩展性。

- 如果一个表只会有几列罢了（比如说字典表，配置表），那么，我们就没有理由使用 INT 来做主键，使用 MEDIUMINT, SMALLINT 或是更小的 TINYINT 会更经济一些
- 如果不需要记录时间，使用 DATE 要比 DATETIME 好得多。

## 2.4.5 InnoDB 还是 MyISAM? 选择合适的存储引擎
1. InnoDB：

- 事务处理
- 外键
- 大尺寸的数据 ( 因为INNODB其支持事务处理和故障恢复。数据库的在小决定了故障恢复的时间长短，InnoDB可以利用事务日志进行数据恢复，这会比较快)
updates 在InnoDB 下会更快一些，尤其在并发量大的时候

2. MyISAM：

- 全文索引
- COUNT() 很快
- 大批的inserts 语句在MyISAM下会快一些
- 小型的应用或项目

### 2.4.6 选择一个ORM(Object Relational Mapper)
Lazy Loading

# 3. 项目中出现的MySQL问题
## 1. Spring 事务：
### ztp遇到的问题：
数据库事务在代码中需要注意，例如 Spring Transcation, 实际的项目里面不写会出现问题。

```java
// 这是一段service代码，比较丑...
// 2019/04/22
@Override
@Transactional(rollbackFor = Exception.class)
public Long saveEvaluationRealtimeAndGradeRealtime(GradeEvaluationRealtimeView gradeEvaluationRealtimeView, Long userId,Long trainRealtimeId, Float distance) {
    TrainRealtimeInstanceEntity trainRealtimeInstance = new TrainRealtimeInstanceEntity();
    trainRealtimeInstance.setUserId(userId);
    trainRealtimeInstance.setTrainRealtimeId(trainRealtimeId);
    trainRealtimeInstance.setTimemark(new Date());
    trainRealtimeInstance.setDistance(distance);
    // 1. 存储 训练计划 
    trainRealtimeInstanceService.insert(trainRealtimeInstance);
    Long trainRealtimeInstanceId = trainRealtimeInstance.getTrainRealtimeInstanceId();
    List<EvaluationRealtimeEntity> evaluationRealtimeList = gradeEvaluationRealtimeView.getEvaluationRealtimeList();
    GradeRealtimeEntity gradeRealtime = gradeEvaluationRealtimeView.getGradeRealtime();
    //存储EvaluationRealtime相关：
    for(EvaluationRealtimeEntity e:evaluationRealtimeList){
        e.setTrainRealtimeInstanceId(trainRealtimeInstanceId);
        e.setUserId(userId);
    }
    // 2. 存储 训练结果相关，是一个耗时操作
    evaluationRealtimeService.insertBatch(evaluationRealtimeList,500);
    // 3. 存储 分数相关：
    gradeRealtime.setUserId(userId);
    // 这边不设置时间，这里的timemark允许是 保存历史的时间。
    gradeRealtime.setTrainRealtimeInstanceId(trainRealtimeInstanceId);
    insert(gradeRealtime);
    return trainRealtimeInstanceId;
}
```

这段代码里面有3处存储数据库的操作，其中2是耗时操作，要存储1800条数据，比较容易出问题，所以可能导致1存储成功，2部分存储成功，3没有存储。导致数据库数据不一致的问题（原本TrainRealtimeInstance与GradeRealtime应该是1对1的关系）

解决方案：将这三个操作写入同一个service方法，在方法前面加上 @Transactional，保证这些是同一个事务，要么全部存储成功，要么一次db操作失败全部回滚，问题解决。

### Spring 事务管理：
- 编程式事务管理 vs. 声明式事务管理:

    - 编程式： 在编程的帮助下有管理事务。极大的灵活性，但却很难维护。
    - 声明式: 从业务代码中分离事务管理。仅仅使用注释或 XML 配置来管理事务。

- 局部事物 vs. 全局事务:

    - 集中的计算环境中,事务管理只涉及到一个运行在一个单一机器中的本地数据管理器。更容易实现。
    - 分布式计算环境中,所有的资源都分布在多个系统中。分布式或全局事务跨多个系统执行，它的执行需要全局事务管理系统和所有相关系统的局部数据管理人员之间的协调。

- Spring中事务源码
```java
public interface PlatformTransactionManager {
   TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;  //申明
   void commit(TransactionStatus status) throws TransactionException;    //提交（成功）
   void rollback(TransactionStatus status) throws TransactionException;  //回滚（失败）
}
```

```java 
//事务的定义：属性
public interface TransactionDefinition {
   int getPropagationBehavior();    //事务的传播行为
   int getIsolationLevel();         //事务的隔离级别
   String getName();                //事务的名称
   int getTimeout();                //事务完成的最大时间
   boolean isReadOnly();            //事务是否只读
}
```  

- Spring的事务隔离级别
    - ISOLATION_DEFAULT: 这是默认的隔离级别。(ISOLATION_READ_COMMITTED)
    - ISOLATION_READ_UNCOMMITTED: 可以发生误读、不可重复读和虚读。
    - ISOLATION_READ_COMMITTED: 能够阻止误读；可以发生不可重复读和虚读。
    - ISOLATION_REPEATABLE_READ: 够阻止误读和不可重复读；可以发生虚读。
    - ISOLATION_SERIALIZABLE: 能够阻止误读、不可重复读和虚读。

关于事务的并发一致性，封锁协议，隔离级别 ztp的理解如下（属于ACID的I）：
| 并发一致性问题 | 封锁协议 | 隔离级别 |
| :-----:|:------:|:-----:|
| 丢失更新 | 一级封锁协议 | 未提交读 |
| 脏读 | 二级封锁协议 | 提交读 (默认) |
| 不可重复读 | 三级封锁协议 | 可重复读 |
| 幻影读 | 四级封锁协议 | 可串行化 |
|

- Spring事务的传播类型

    - PROPAGATION_MANDATORY: 支持当前事务；如果不存在当前事务，则抛出一个异常。
    - PROPAGATION_NESTED: 如果存在当前事务，则在一个嵌套的事务中执行。
    - PROPAGATION_NEVER: 不支持当前事务；如果存在当前事务，则抛出一个异常。
    - PROPAGATION_NOT_SUPPORTED: 不支持当前事务；而总是执行非事务性。
    - PROPAGATION_REQUIRED: 支持当前事务；如果不存在事务，则创建一个新的事务。
    - PROPAGATION_REQUIRES_NEW: 创建一个新事务，如果存在一个事务，则把当前事务挂起。
    - PROPAGATION_SUPPORTS: 支持当前事务；如果不存在，则执行非事务性。
    - TIMEOUT_DEFAULT: 使用默认超时的底层事务系统，或者如果不支持超时则没有。   

- Spring声明式事务管理（基于AOP）

    声明式事务管理建立在AOP之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

    使用方式：@Transactional注解添加到类或者方法上，并且配置一些属性：

    - @Transactional 添加到方法上：该方法具有事务性。
    - @Transactional 添加到类上：所有该类的公共方法都配置相同的事务属性信息。[例如这个例子](https://ztxpp.cc/2019/03/24/mysql/#jump)

## 2. varchar的长度：
- mysql 4.0版本以下，varchar(50), 指的是50字节，如果存放utf8汉字时，只能存放16个（每个汉字3字节）

- mysql 5.0版本以上，varchar(50), 指的是50字符，无论存放的是数字、字母还是UTF8汉字（每个汉字3字节），都可以存放50个。