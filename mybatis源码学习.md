---
title: mybatis源码学习
date: 2019-11-02 15:28:42
tags:
- java
- mybatis
---

>软件开发工程师进阶，必须要学会阅读源码，以mybatis的源码为例进行源码阅读的训练和学习，重点有二，一是学习mybatis的设计模式和编码方式，二是学习阅读源码的方式。参照资料：[深入浅出MyBatis技术内幕](https://ke.qq.com/course/435473?tuin=7b49ac1d)

注：看源码的过程中，发现自己不熟悉的知识点： java泛型，lambda表达式，一些设计模式等。

# 1. 整体架构
![mybatis整体架构](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191105142603.png)

# 2. 日志模块
## 2.0 需求分析
需求：

- MyBatis没有提供日志的实现类，需要接入第三方的日志组件，但第三方日志组件都有各自的Log级别，且各
不相同，而MyBatis统一提供了trace、debug、warn、error四个级别。
- 自动扫描日志实现，并且第三方日志插件加载优先级如下：slf4J → commonsLoging → Log4J2 → Log4J
→ JdkLog;
- 日志的使用要优雅的嵌入到主体功能中；

解决方式：
- 功能点一：client和提供者的功能相近，只是接口不匹配（第三方日志组件和），可以使用适配器模式。
- 功能点二：通过静态代码块的形式设置日志加载优先级顺序。
- 功能点三：运用动态代理的方式实现 日志功能优雅的嵌入到主体功能中

## 2.1 日志的适配器
![MyBatis日志的适配器](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191107140626.png)

Mybatis的日志模块采用适配器模式来进行组织，做第三方插件与目标接口之间的适配工作。诸多适配器类，实现org.apache.ibatis.logging.Log接口, 在适配器内部做了函数功能的适配，举例如下：
```java
/**
 * Log4jjImpl 适配器类
 * @author Eduardo Macarron
 */
public class Log4jImpl implements Log {
  private static final String FQCN = Log4jImpl.class.getName();
  private final Logger log;   // 此处的Logger 是真实提供功能的 org.apache.log4j.Logger接口
  public Log4jImpl(String clazz) {
    log = Logger.getLogger(clazz);
  }
  @Override
  public boolean isDebugEnabled() {
    return log.isDebugEnabled();
  }
  @Override
  public boolean isTraceEnabled() {
    return log.isTraceEnabled();
  }

  //下面几个方法是主要转换部分，相当于把 org.apache.log4j.Logger的不同等级，
  // 转化为 org.apache.ibatis.logging.Log的 对应等级。其他的适配器也是类似。
  @Override
  public void error(String s, Throwable e) {
    log.log(FQCN, Level.ERROR, s, e);
  }
  @Override
  public void error(String s) {
    log.log(FQCN, Level.ERROR, s, null);
  }
  @Override
  public void debug(String s) {
    log.log(FQCN, Level.DEBUG, s, null);
  }
  @Override
  public void trace(String s) {
    log.log(FQCN, Level.TRACE, s, null);
  }
  @Override
  public void warn(String s) {
    log.log(FQCN, Level.WARN, s, null);
  }
}
```
Log4j2LoggerImpl应该是根据情况有两种转换方式，其中一种Log4j2AbstractLoggerImpl通过装饰者模式做了一些功能的增强，这里不做细致研究。

## 2.2 静态代码块实现加载的优先级
诸多第三方日志的加载顺序，是在MyBatis的代码中写死的！ 通过LogFactory的静态代码块来实现。 
- 静态代码块是给类初始化的，而构造代码块是给对象初始化的。
- 执行顺序： 静态代码块（JVM加载类时加载）> 构造块（创建对象时加载，给这个类所有的对象使用）> 构造方法（创建对象时加载，给特异性对象使用）

```java
// LogFactory类 静态代码块部分
  static {
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
  }
```
注意：

```java
    tryImplementation(LogFactory::useSlf4jLogging);
/*
*   上述的lamda的写法相当于一个匿名内部类（实现了Runnable的实现类），转化成下面的写法：
    (由于tryImplementation方法需要一个Runnable参数)
    tryImplementation(new Runnable() {
      @Override
      public void run() {
        LogFactory.useSlf4jLogging();
      }
    });
*/
//    tryImplementation(()->LogFactory.useSlf4jLogging());
```

## 2.3 动态代理嵌入日志功能
说道优雅的嵌入某功能，想到Spring AOP的思想，其本质是动态代理模式，参看<a href="https://ztxpp.cc/2019/10/21/designPattern2/#%E5%9B%9B-%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F#JDK_PROXY_PATTERN"> 设计模式-动态代理章节</a>， MyBatis使用的是JDK动态代理模式。

打印日志需求：
- 在创建prepareStatement时，打印执行的SQL语句
- 访问数据库时，打印参数的类型和值
- 查询出结构后，打印结果数据条数

MyBatis的jdbc包里，是这样设计的：

![MyBatis的动态代理模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191107163632.png)

四个Logger类负责了不同时期打印信息的功能，均实现了InvocationHandler接口。观察一个connect的方法：

```java
  public static Connection newInstance(Connection conn, Log statementLog, int queryStack) {
    InvocationHandler handler = new ConnectionLogger(conn, statementLog, queryStack);
    ClassLoader cl = Connection.class.getClassLoader();
    return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);
  }
```

实际生成的并不是原始的Connection接口的实现类，而是 **由JDK的 Proxy类动态生成的，实现了Connection接口的，聚合了InvocationHandler接口实现类** 的： $Proxy 类。

这四个类之间的关联顺序如下：

![关联](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191107173622.png)


# 3. 数据源模块
## 3.0 需求分析
数据源工厂 -> 数据源 -> 数据库连接

DataSourceFactory -> DataSource -> Connection

- 常见的数据源需要实现 javax.sql.DataSource接口
- MyBatis不但要能集成第三方的数据源组件，自身也提供了数据源的实现
- 一般情况下，数据源的初始化过程参数较多，比较复杂,客户端不应该关注这个创建数据源的过程

## 3.1 由数据源工厂产生数据源
- 工厂模式 or 简单工厂模式

1. 工厂模式的分析见这篇博客， MyBatis使用的是简单工厂模式，由数据源工厂产生数据源。只不过datasource包分为了两种产生连接的类型： Pool和 unPool。
    ```java
    // UnpooledDataSourceFactory类
    public UnpooledDataSourceFactory() {
      this.dataSource = new UnpooledDataSource();
    }
    @Override
    public void setProperties(Properties properties) {
    // 省略...
    }
    ```
2. 创建对象的几种方式及其特点：
- new对象： 创建简单对象使用，对象创建和对象使用 混杂在一起，违反单一职责原则； 业务扩展可能修改代码，违反开闭原则
- 反射创建对象： 同上。
- 工厂类创建对象：提取出创建对象的同一语句，方便修改和维和。扩展时只需要增加对应的工厂，符合开闭原则。


## 3.2 由数据源（连接池）产生连接
- 连接池的数据结构
- 获取连接的算法流程

1. 连接池的数据结构就是两个 ArrayList
```
数据源模块项目结构：
datasource
    ├─  pooled  使用连接池的包
    │    ├─ PooledDataSourceFactory     数据源工厂类
    │    ├─ PooledDataSource            一个简单、同步、安全的数据库连接池类
    │    ├─ PooledConnection            使用动态代理增强的数据库连接    
    │    └─ PoolState                   用于管理PooledConnection对象状态的组件，通过两个list分别 管理空闲状态的连接资源和活跃状态的连接资源
    │ 
    └─  unpooled  不使用连接池的包
         ├─ UnpooledDataSourceFactory   数据源工厂类
         └─ UnpooledDataSource          数据库连接池类  
```

PoolState中保存着两个List:
```java
//  空闲的连接
  protected final List<PooledConnection> idleConnections = new ArrayList<>();
// 活跃的，使用中的连接
  protected final List<PooledConnection> activeConnections = new ArrayList<>();
```
注意点：
- 使用List是因为具有先后顺序，可以记录connection的使用时间，判断是否超时，只需看队首的元素。
- 这两个List随PoolState对象一起保存在JVM堆中，线程公有，需要注意并发问题。

2. 获取连接的算法流程

    参考这里，先略过
    https://blog.csdn.net/luanlouis/article/details/37671851


# 4. 缓存模块
## 4.1 应用： MyBatis有两级缓存机制
- 一级缓存: SqlSession级别
- 二级缓存：Mapper级别

## 4.2 需求分析
- MyBatis的缓存底层是基于Map来实现的，核心功能是从缓存中读取数据
- 缓存模块附加了很多额外功能，如：防止缓存击穿，添加缓存清空策略（FIFO,LRU）,序列化， 日志能力，定时清空.这些功能如何添加。
- 附加功能如何任意组合使用。

## 4.3 装饰器模式实现缓存模块的附加功能及其任意叠加
- 装饰器模式，参考[这篇博客](https://ztxpp.cc/2019/10/21/designPattern2)。 
- 装饰器模式使用案例：IO中输入流和输出流的设计
    ```java
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(new FileInputStream("c://a.txt")))
    ```
- MyBatis的缓存模块的项目结构

    ![缓存模块：装饰者模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191109164827.png)

    MyBatis的缓存接口Cache,有很多实现，每一个实现都是一个装饰器，对原始的Cache进行包装和功能增强。 装饰器内部含有Cache对象（接口对象，由于多态能存下任意的实现对象）。所以可以将各种装饰者进行**任意叠加**。

    比如：
    ```java
    new BlockingCache(new FifoCache(new LoggingCache())); //伪代码
    ```

## 4.4 详细分析一个装饰者实现类：BlockingCache

用途：防止缓存雪崩
- 缓存雪崩：当缓存服务器重启或者大量缓存集中在某一个时间段失效，这样在失效的时候，也会给后端系统统造成很大的压力。
- 缓存穿透：如果key对应的value是一定不存在的，并且对该key并发请求量很大，就会导致大量的请求在缓存中拿不到数据而去请求数据库，就会对后端系统造成很大的压力。


```java
public class BlockingCache implements Cache {

  private long timeout; //设置过期时间
  private final Cache delegate;  // 装饰者模式中的concrete impl
  
  //用ConcurrentHashMap来当锁，key是缓存本身的值，value是一把重入锁
  // 比起直接用lock， 把缓存作为key,能减小锁粒度：
  //    访问同一个缓存的线程 用同一把锁; 访问不同缓存的线程，不相互影响。很聪明！
  private final ConcurrentHashMap<Object, ReentrantLock> locks;  

  public BlockingCache(Cache delegate) {
    this.delegate = delegate;
    this.locks = new ConcurrentHashMap<>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object value) {
    try {
      delegate.putObject(key, value);
    } finally {
      releaseLock(key);
    }
  }

//获取缓存：
//    逻辑是取缓存值需要锁，取到了缓存释放锁，取不到缓存继续等待。 以此防止大量线程访问数据库（缓存雪崩）
  @Override
  public Object getObject(Object key) {
    acquireLock(key);
    Object value = delegate.getObject(key);
    if (value != null) {
      releaseLock(key); //这里表示，第一个线程如果能在缓存中找到这个元素，则释放锁；没有找到，后面的线程就别想再找到这个元素了（拿不到锁）。
    }
    return value;
  }

  @Override
  public Object removeObject(Object key) {
    // despite of its name, this method is called only to release locks
    releaseLock(key);
    return null;
  }

  @Override
  public void clear() {
    delegate.clear();
  }

  private ReentrantLock getLockForKey(Object key) {
    //如果key存在，则找到这个lock；如果不存在，则创建一把新锁（第一次访问，锁不存在，需要创建）给这个key作为value
    return locks.computeIfAbsent(key, k ->  new ReentrantLock());
   // return locks.computeIfAbsent(key, k -> {return new ReentrantLock();});  
  }

  private void acquireLock(Object key) {
    Lock lock = getLockForKey(key);
    if (timeout > 0) {
      try {
        boolean acquired = lock.tryLock(timeout, TimeUnit.MILLISECONDS);
        if (!acquired) {
          throw new CacheException("Couldn't get a lock in " + timeout + " for the key " +  key + " at the cache " + delegate.getId());
        }
      } catch (InterruptedException e) {
        throw new CacheException("Got interrupted while trying to acquire lock for key " + key, e);
      }
    } else {
      lock.lock();
    }
  }

  private void releaseLock(Object key) {
    ReentrantLock lock = locks.get(key);
    if (lock.isHeldByCurrentThread()) {
      lock.unlock();
    }
  }

  public long getTimeout() {
    return timeout;
  }

  public void setTimeout(long timeout) {
    this.timeout = timeout;
  }
}

```

## 4.5 装饰器的添加
```java
//在CacheBuilder中找到例子
 private Cache setStandardDecorators(Cache cache) {
    try {
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      if (size != null && metaCache.hasSetter("size")) {
        metaCache.setValue("size", size);
      }
      if (clearInterval != null) {
        cache = new ScheduledCache(cache); //添加了ScheduledCache装饰器
        ((ScheduledCache) cache).setClearInterval(clearInterval);
      }
      if (readWrite) {
        cache = new SerializedCache(cache); //添加了SerializedCache装饰器
      }
      cache = new LoggingCache(cache);//添加了LoggingCache装饰器
      cache = new SynchronizedCache(cache);//添加了 SynchronizedCache装饰器
      //注意这里添加的SynchronizedCache装饰器，可以防止二级缓存的线程安全问题。
      if (blocking) {
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }
```


## 4.5 缓存与数据库同步机制
- 数据实时同步失效
    - 强一致性
    - 增：写库，更新缓存； 删：删库； 改； 查

- 数据准实时同步
    - 准一致性

- 任务调度更新
    - 最终一致性


# 5. 反射模块
## 5.1 反射模块概述

普通ORM框架查询数据的过程：

![orm框架的作用](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191116144917.png)

反射模块的作用是后面两个部分：
- 实例化目标对象
- 对象属性的赋值

反射核心类：
- ObjectFactory（接口）: 利用工厂模式创建对象的实例

- ReflectorFactory: 利用工厂模式创建 Reflector
  - Reflector： **类的元数据的封装**；对JDK提供的反射j进行了**功能增强、性能提升**

- ObjectWrapperFactory：ObjectWrapper 的工厂类，用于创建ObjectWrapper
  - ObjectWrapper： **对象的封装**，抽象了对象的属性信息，他定义了一系列查询对象属性信息的方法，以及更新属性的方法

## 5.2 ObjectFactory
- 主要作用：根据Class创建对象实例
- 反射创建实例对象的两种方式：
```java
//方法1：通过class创建对象：
Class classType = Class.forName("com.ztxpp.Person");
Object obj = classType.newInstance();

//方法2：通过构造器创建对象
Constructor<Person> con = Person.class.getConstructor(String.class,int.class); 
    //这里的参数是构造器的入参
Object obj = con.newInstance("lxf", 23);
```

DefaultObjectFactory, ObjectFactory接口的实现类相应代码：
```java
public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes,List<Object>constructorArgs) {
  Class<?> classToCreate = resolveInterface(type);
  // we know types are assignable
  return (T) instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
}

private  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes,List<Object> constructorArgs) {
  try {
    Constructor<T> constructor;
    //如果创建对象时没有传参数，直接用默认构造器创建对象：
    if (constructorArgTypes == null || constructorArgs == null) {
      constructor = type.getDeclaredConstructor();
      try {
        return constructor.newInstance();
      } catch (IllegalAccessException e) {
        if (Reflector.canControlMemberAccessible()) {
          //如果构造器是private，暴力方式变成 public：
          constructor.setAccessible(true);
          return constructor.newInstance();
        } else {
          throw e;
        }
      }
    }
    //如果创建对象时传了参数：
    constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Clas[constructorArgTypes.size()]));
    try {
      return constructor.newInstance(constructorArgs.toArray(new Object[constructorArgs.siz()]));
    } catch (IllegalAccessException e) {
      if (Reflector.canControlMemberAccessible()) {
        constructor.setAccessible(true);
        return constructor.newInstance(constructorArgs.toArray(new Objec[constructorArgs.size()]));
      } else {
        throw e;
      }
    }
  } catch (Exception e) {
    String argTypes = Optional.ofNullable(constructorArgTypes).orElseGe(Collections::emptyList)
        .stream().map(Class::getSimpleName).collect(Collectors.joining(","));
    String argValues = Optional.ofNullable(constructorArgs).orElseGe(Collections::emptyList)
        .stream().map(String::valueOf).collect(Collectors.joining(","));
    throw new ReflectionException("Error instantiating " + type + " with invalid types (" +argTypes + ") or values (" + argValues + "). Cause: " + e, e);
  }
}

// 解析接口类型，用对应的实现类类型替代
// 引用： 在面向对象的开发中我们会提倡面对接口而不是面向具体实现的编程原则，但是在创建对象时则必须指定一个具体的类，为了解决这个问题，mybatis对常用的集合超类指定了具体的实现类：
protected Class<?> resolveInterface(Class<?> type) {
  Class<?> classToCreate;
  if (type == List.class || type == Collection.class || type == Iterable.class) {
    classToCreate = ArrayList.class;
  } else if (type == Map.class) {
    classToCreate = HashMap.class;
  } else if (type == SortedSet.class) { // issue #510 Collections Support
    classToCreate = TreeSet.class;
  } else if (type == Set.class) {
    classToCreate = HashSet.class;
  } else {
    classToCreate = type;
  }
  return classToCreate;
}
```

测试类：
```java
@Test
public void reflectTest(){
  ObjectFactory objectFactory= new DefaultObjectFactory();
  TestUserPojo testUserPojo = objectFactory.create(TestUserPojo.class);
}
// 成功创建对象...
```

## 5.3 Reflector && ReflectorFactory
1. Reflector： **类的元数据的封装**；对JDK提供的反射j进行了**功能增强、性能提升**

```java 
public class Reflector {
//POJO类的元数据
  private final Class<?> type; //类型
  private final String[] readablePropertyNames; //属性有get方法
  private final String[] writablePropertyNames; //属性有set方法
  private final Map<String, Invoker> setMethods = new HashMap<>(); // 保存get方法： <id, getId()>
  private final Map<String, Invoker> getMethods = new HashMap<>(); // 保存set方法： <id, setId()>
  private final Map<String, Class<?>> setTypes = new HashMap<>();  // set方法的入参类型：<id, Long>
  private final Map<String, Class<?>> getTypes = new HashMap<>();  // get方法的返回参数：<id, Long>
  private Constructor<?> defaultConstructor; // 默认的构造方法
  
//...
}
```
上述代码可见，Reflector实际是对 bean的类的封装。 在MyBatis启动时，就已经将bean对象的元数据加载带 Reflector中了，当通过反射调用方法，或者属性时，其实是从Reflector的容器中获取的。这样提高了jdk原始反射的性能和效率。 -- “空间换时间”


2. ReflectorFactory: 的实现类 DefaultReflectorFactory

  - 工厂模式，将reflector的创建和使用分开了
  - 用线程安全的Map做缓存，在MyBatis刚启动的时候，将所有的bean用reflector封装起来，并且保存在这个ConcurrentHashMap中。

```java
//用这个线程安全的Map做缓存
private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<>();

public Reflector findForClass(Class<?> type) {
  if (classCacheEnabled) { //如果缓存打开
    //computeIfAbsent方法的含义： ConcurrentHashMap 中找到则返回；找不到则创建新的，填入type对应的value中，然后返回
    return reflectorMap.computeIfAbsent(type, Reflector::new);
  } else {
    return new Reflector(type);
  }
}
```

测试：
```java
  @Test
  public void reflectorFactoryTest(){
    ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
    Reflector reflector = reflectorFactory.findForClass(TestUserPojo.class);
    Constructor<?> constructor = reflector.getDefaultConstructor();
    String[] getablePropertyNames = reflector.getGetablePropertyNames();
    String[] setablePropertyNames = reflector.getSetablePropertyNames();
  }
```

## 5.4 ObjectWrapper
- 主要作用：为对象实例赋值
- 主要方式：采用反射方式，获取set方法，来进行赋值

## 5.5 MetaObject && SystemMetaObject
采用门面模式，包装上述核心类，提供给外部使用反射模块的功能

![门面模式](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191117160149.png)

SystemMetaObject中包含了MetaObject，将上述过程在内部完成，用静态方法的形式对外界提供简单易用的接口：

采用门面模式之前：

```java
@Test
public void objectWrapperTest(){
  ObjectFactory objectFactory= new DefaultObjectFactory();
  TestUserPojo testUserPojo = objectFactory.create(TestUserPojo.class);
  ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  MetaObject metaObject = MetaObject.forObject(testUserPojo,objectFactory,new DefaultObjectWrapperFactory(),reflectorFactory);
  ObjectWrapper wrapper = new BeanWrapper(metaObject,testUserPojo);

  PropertyTokenizer prop = new PropertyTokenizer("name");
  wrapper.set(prop,"ztxpp");
}
```

采用门面模式之后：
```java
@Test
public void systemMetaObjectTest(){
  TestUserPojo testUserPojo = SystemMetaObject.DEFAULT_OBJECT_FACTORY.create(TestUserPojo.class);
  MetaObject metaObject = SystemMetaObject.forObject(testUserPojo);
  metaObject.setValue("name","a'ming");
}
```

## 5.6 模拟Mybatis使用反射模块

```java
@Test
public void reflectTest(){
  TestUserPojo testUserPojo = SystemMetaObject.DEFAULT_OBJECT_FACTORY.creat(TestUserPojo.class);
  MetaObject metaObject = SystemMetaObject.forObject(testUserPojo);

  //1,从数据库获取数据(模拟)
  Map<String, Object> dbResult = new HashMap<>();
  dbResult.put("id",new Long(1));
  dbResult.put("name","ztxpp");
  dbResult.put("good_student",true);

  //2. 从xml文件获取映射规则（模拟）
  Map<String,String> mapper = new HashMap<>();
  mapper.put("id","id");
  mapper.put("name","name");
  mapper.put("goodStudent","good_student");

  //3. 使用反射模块将数据生成pojo对象
  for (String key : mapper.keySet()) {
    String propName = key;
    Object propValue = dbResult.get(mapper.get(key));
    metaObject.setValue(propName,propValue);
  }
  System.out.println(metaObject.getOriginalObject());
}
```

>后记： 
Mybatis的源码量较大，这次学习只涉及到：日志模块、数据源模块、缓存模块、反射模块 这四个模块，甚至没有涉及到源码的运行流程，只是停留在基础组件层。 学习源码的过程先看文档，再分层，然后梳理流程找入口，最后多调试。这只是开始，后续需要学习jdk并发部分和Spring的源码。