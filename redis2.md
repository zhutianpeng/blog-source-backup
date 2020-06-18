---
title: redis系列（二）
date: 2019-09-19 15:53:55
tags:
---

为了深入学习 **redis相关知识** 的系统，开始了[享学课堂](https://ke.qq.com/course/442946?tuin=7b6410e7)的学习。 注：仅用于个人的学习记录。

# 3. redis五种数据类型的应用
## 1. String应用
### 1. INCR原子性操作：
- 文章浏览数 ++ ：在多操作（先读再写）的情况下不会重复出错（需要保证操作的原子性）
- 论坛回帖数 ++ ：先进行redis中的帖子数 INCR, 会返回加完的值，然后去写库，保证不会有重复的帖子序号出现。

### 2. 数据库主键自增的实现（类似auto increment）
- redis是单线程模型，如果大量的请求（比如10万）同时到达，都要获取自己的序号（主键），redis会排队执行，但是速度较慢。 解决方法是： 在web server前端加上nginx分流，分到不同的server上，再去请求redis的时候，redis直接返回1~1000个序号给server1,并缓存在server本地（预生成），redis返回 1001~2000给server2,并缓存在server2上... 这样可以节省redis的吞吐量。

### 3. session共享
- session信息存在redis里，多台机器能获取到一个用户的同一个session信息。

### 4. 分布式锁
- 问题：为了分布式中解决重复调用的问题（可能是调用一个远端的服务）。（单机中的防止重复调用，直接加锁，比如synchronized即可，但是无法解决分布式环境下的问题。）
- 解决：将单机的锁升级为分布式锁。 -> 使用redis实现分布式锁。
- 设计：
    - 加锁： SETNX + 缓存有效期 
    - 解锁： LUA脚本
- 解释： 
    - 锁的表示【SETNX】： set if not exist, 成功插入则返回1， key存在则返回0。 以此作为锁，set成功则拿到锁；失败则表示锁此时被人占用。
    - 解决死锁【缓存有效期】：

    ```
    setnx ticket 123615243123654 px 1;
    ```


    表示这把锁最长维持1s，需要在这段时间能完成业务，否则自动key失效，相当于自动是释放了锁
    - 解锁【lua脚本】：主要为了保持操作的整体原子性，包含三个操作：
        - 获取数据
        - 判断一致性
        - 删除数据
- 实现：

RedisLock实现Lock接口
```java
public class RedisLock implements Lock {
	
	private static final String  KEY = "LOCK_KEY";
	
	@Resource
	private JedisConnectionFactory factory;

	private ThreadLocal<String> local = new ThreadLocal<String>();
	
	
	@Override
	//阻塞式的加锁
	public void lock() {
		//1.尝试加锁
		if(tryLock()){
			return;
		}
		//2.加锁失败，当前任务休眠一段时间
		try {
			Thread.sleep(10);// 性能浪费
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		//3.递归调用，再次去抢锁
		lock();
	}

	@Override
	//阻塞式加锁,使用setNx命令返回OK的加锁成功，并生产随机值
	public boolean tryLock() {
		//产生随机值，标识本次锁编号
		String uuid = UUID.randomUUID().toString();
		Jedis jedis = (Jedis) factory.getConnection().getNativeConnection();

		/**
		 * key:我们使用key来当锁
		 * uuid:唯一标识，这个锁是我加的，属于我
		 * NX：设入模式【SET_IF_NOT_EXIST】--仅当key不存在时，本语句的值才设入
		 * PX：给key加有效期
		 * 1000：有效时间为 1 秒
		 */
		String ret = jedis.set(KEY, uuid,"NX","PX",1000);

		//设值成功--抢到了锁
		if("OK".equals(ret)){
			local.set(uuid);//抢锁成功，把锁标识号记录入本线程--- Threadlocal
			return true;
		}

		//key值里面有了，我的uuid未能设入进去，抢锁失败
		return false;
	}

	//正确解锁方式
	@Override
	public void unlock() {
		//读取lua脚本
		String script = FileUtils.getScript("unlock.lua");
		//获取redis的原始连接
		Jedis jedis = (Jedis) factory.getConnection().getNativeConnection();
		//通过原始连接连接redis执行lua脚本
		jedis.eval(script, Arrays.asList(KEY), Arrays.asList(local.get()));
	}

	//-----------------------------------------------

	@Override
	public Condition newCondition() {
		return null;
	}
	
	@Override
	public boolean tryLock(long time, TimeUnit unit)
			throws InterruptedException {
		return false;
	}

	@Override
	public void lockInterruptibly() throws InterruptedException {
	}
}
```

使用：
```java
lock.lock();
try {
    if (count > 0) {
        //模拟卖票业务处理
        amount++;
        count--;
    }
}finally{
    lock.unlock();
}
```

解锁的Lua脚本：
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then 
    return redis.call("del",KEYS[1]) 
else 
    return 0 
end
```

## 2. Hash应用
### 1. 保存对象 -> 淘宝购物车的实现
- 需求：
	- 全选购物车中的商品
	- 获取购物车商品数量
	- 删除购物车商品
	- 增加或减少商品的数量

- 使用hash来实现，当前登录用户ID号做为KEY，商品ID号为Field, 加入购物车数量为value。
- 每次增量操作时，先操作db，再操作redis，保证原子性；其中操作redis的方式就是上述提供的操作hash的方式。

	```
	hmset user:001 product:01 1 product:02 1
	```

## 3. List应用
### 1. 实现队列、栈、阻塞队列
- 队列：lpush + rpop
- 栈：lpush + lpop
- 阻塞队列 blockqueue：lpush + brpop （监听，如果有消息就接收，没有消息阻塞监听）

### 2. 微信公众号订阅文章（用redis的List实现）
``` 
# CSDN发布了一条消息ID为999
lpush mes:004 999

# 男子别输在说话上也发布了一条消息ID为1000
lpush mes:004 1000

# 那么左图Lison老师的微信消息列表实现如下
lrange mes:004 0 5
```

## 4. Set应用
### 1. set集合的元素不重复性
- 抽奖：
```
# 当Lison点击参与抽奖时，数据放入set集合
sadd act:001  004

# 开始抽奖2名中奖者
srandmember act:001 2 或 spop act:001 2

# 查看有多少用户参加了本次抽奖
smembers act:001
```

- 点赞：
```
# 张三用户ID 为userId:01

# 1）张三对消息ID008点赞
sadd zan:008  userId:01

# 2)张三取消了对消息008的点赞
srem zan:008  userId:01

# 3）检查用户是否点过赞
sismember zan:008  userId:01

# 4）获取消息ID008所有的点赞用户列表
smembers zan:008

# 5）消息ID008的点赞数计算
scard zan:008
```

- 投票，黑名单

### 2. set集合的集合运算性（交并补）
举例：微博的微关系设计

## 5. zset应用
### 1. zset的集合性
- 元素不重复
- 集合运算：交并补

### 2. zset的可排序性
百度热度榜,排行榜

- 将日期作为Key
- 按照点赞数排序,解决数据库排序的压力

```
# 1)点击话题
zincrby topic:20190912 1 军嫂怒怼张馨予

# 2)右侧排行实现,展示今日前9排名
zrevrange  topic:20190912 0 20  withscores

# 3)统计近3日点击数据(集合运算：求并集)
zunionstore topic:3day 3  topic:20190910  topic:20190911 topic:20190912

# 4)展示近3日的排行前9名
zrevrange topic:20190910-20190912 0 9  withscores
```
	