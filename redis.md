---
title: redis系列（一）
date: 2019-09-17 22:33:14
tags: redis
---


为了深入学习 **redis相关知识** 的系统，开始了[本课程](https://ke.qq.com/course/442946?tuin=7b6410e7)的学习。 

# 1. redis快速入门
## 1. redis的特性：
- 速度快：基于内存型存储
- 基于键值对的数据结构服务器
- 数据类型比较丰富：
    - key：String
    - value: String, hash, list, set, zset
- 简单稳定
- 持久化
- 主从复制
- 高可用和分布式转移
- 客户端API较多
    - java：Jedis

## 2. redis的应用场景
- 缓存数据库
- 排行榜
- 计数器应用
- 社交网络
- 消息队列
- 分布式锁
- session共享

## 3. redis的相关配置文件说明
![redis的相关配置文件说明](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918134934.png)
- linux关闭redis的方式
    - 错误方式：直接kill进程，会丢失数据
    - 正确方式：./redis-cli -h 192.168.xx.xx -p 6379 -a xxxx shutdown，  数据不会丢失

# 2. redis五种数据结构
## 1. String 
常用指令
```
# ex：10秒后过期
set age 23 ex 10 
# ttl： 查看此 key的有效时间
ttl age

# setnx: set if not exist （如果key不存在则set，返回1；key存在则不进行任何操作，返回0） 
setnx name test

# 批量设值
mset country china city beijing address haidian (设置了三个键值对)
# 批量获取
mget country city address

# 原子性操作
incr age                # 整数自增，非整数则返回错误，无此key则会先set
decr age                # 整数自减
incrby age 2            # 以2自增
decrby age 2            # 以2自减
incrbyfloat score 1.1   # 以浮点数自增
```

## 2. Hash

![Hash数据存储](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918163212.png)

类似这种数据存储，String可是实现，但是会出现Key值的冗余多占存储空间。而hash比较适合“对象类型存储”

```
# 设置key为 user:1,  value为一个field(map)，其中name：james； age：18
hmset user:1 name james age18


命令   hset key field value

设值：hset user:1 name james           //成功返回1，失败返回0
取值：hget user:1 name                 //返回james
删值：hdel user:1 age                  //返回删除的个数
计算个数：hset user:1 name james; hset user:1 age 23; 
         hlen user:1                  //返回2，user:1有两个属性值
批量设值：hmset user:2 name james age 23 sex boy //返回OK
批量取值：hmget user:2 name age sex    //返回三行：james 23 boy
判断field是否存在：hexists user:2 name //若存在返回1，不存在返回0
获取所有field: hkeys user:2            // 返回name age sex三个field
获取user:2所有value：hvals user:2     // 返回james 23 boy
获取user:2所有field与value：hgetall user:2 //name age sex james 23 boy值
增加1： hincrby user:2 age 1          //age+1
       hincrbyfloat user:2 age 2     //浮点型加2

```
比较：三种方案实现用户信息存储优缺点


- 原生：

    ```
    set user:1:name james; 
    set user:1:age 23;  
    set user:1:sex boy; 
    ```

    优点：简单直观，每个键对应一个值  
    缺点：键数过多，占用内存多，用户信息过于分散，不用于生产环境  

- 将对象序列化存入redis:  
    ```
    set user:1 serialize(userInfo); 
    ```

    优点：编程简单，若使用序列化合理内存使用率高  
    缺点：序列化与反序列化有一定开销，更新属性时需要把userInfo全取出来进行反序列化，更新后再序列化到redis  (操作简单但是更新难)

- 使用hash类型：  
    ```
    hmset user:1 name james age 23 sex boy  
    ```

    优点：简单直观，使用合理可减少内存空间消耗  
    缺点：要控制ziplist与hashtable两种编码转换，且hashtable会消耗更多内存erialize(userInfo);  

## 3. list
![list](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918165946.png)
- 用来存储多个有序的字符串，一个列表最多可存2的32次方减1个元素
- lpush从左侧插值，rpush从右侧插值
- 有序：可以通过索引下标获取元素或某个范围内元素列表，列表元素可以重复

```
添加命令：
 rpush james c b a //从右向左插入cba, 返回值3
 lrange james 0 -1 //从左到右获取列表所有元素 返回 c b a
 lpush key c b a //从左向右插入cba
 linsert james before b teacher //在b之前插入teacher, after为之后，使					                用lrange james 0 -1 查看：c teacher b a
 

查找命令：
 lrange key start end //索引下标特点：从左到右为0到N-1
 lindex james -1 //返回最右末尾a，-2返回b
 llen james        //返回当前列表长度
 lpop james       //把最左边的第一个元素c删除
 rpop james      //把最右边的元素a删除
```

## 4. set
![set](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918171115.png)
- 应用场景：用户标签（求集合交集），社交，查询有共同兴趣爱好的人（求集合交集）,智能推荐
- 多元素，不重复，无序，
- 一个集合最多可存2的32次方减1个元素，除了支持增删改查，还支持集合运算：交集、并集、差集；

基础操作
```
exists user             //检查user键值是否存在
sadd user a b c         //向user插入3个元素，返回3
sadd user a b           //若再加入相同的元素，则重复无效，返回0
smember user            //获取user的所有元素,返回结果无序

srem user a             //返回1，删除a元素

scard user              //返回2，计算元素个数
```

集合运算
```
标签，社交，查询有共同兴趣爱好的人,智能推荐
使用方式：
给用户添加标签：
  sadd user:1:fav basball fball pq
  sadd user:2:fav basball fball   
 
或给标签添加用户
  sadd basball:users user:1 user:2
  sadd fball:users user:1 user:2
 
计算出共同感兴趣的人：
  sinter user:1:fav user2:fav
```

## 5. Sorted Set
![sort set](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918173717.png)
- 常用于排行榜，如视频网站需要对用户上传视频做排行榜，或点赞数
- 与集合有联系，不能有重复的成员,集合中的每一项多了一个权重，可以进行排序

普通指令： 
```
zadd key score member [score member......]
zadd user:zan 200 james                                   //james的点赞数1, 返回操作成功的数1
zadd user:zan 200 james 120 mike 100 lee    // 返回3
zadd test:1 nx 100 james                     //键test:1必须不存在，主用于添加
zadd test:1 xx incr 200 james             //键test:1必须存在，主用于修改,此时为300
zadd test:1 xx ch incr -299 james      //返回操作结果1，300-299=1

zrange test:1 0 -1 withscores   //查看点赞（分数）与成员名

zcard test:1       //计算成员个数， 返回1
```

排名场景：
```
zadd user:3 200 james 120 mike 100 lee      //先插入数据
zrange user:3 0 -1 withscores                         //查看分数与成员

zrank user:3 james           //返回名次：第3名返回2，从0开始到2，共3名
zrevrank user:3 james     //返回0， 反排序，点赞数越高，排名越前
```

比较：
list vs Set vs sorted set

![list set zset](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190918194618.png)

## redis全局命令

redis键的操作：
```
查看所有键：
    keys * 
键总数 ：
    dbsize         //2个键，如果存在大量键，线上禁止使用此指令
检查键是否存在：
    exists key    //存在返回1，不存在返回0
删除键：
    del key      //del hello school, 返回删除键个数，删除不存在键返回0
键过期：
    expire key seconds        //set name test  expire name 10,表示10秒过期
    ttl key                   // 查看剩余的过期时间
键的数据结构类型：
    type key                  //返回string,键不存在返回none
```

redis数据库管理：
```
# 切换0号数据库（共0~15）
select 0
# 清空该数据库 （危险操作）
flushdb
# 清空所有数据库（危险操作）
flushall
# 键的总数
dbsize
```