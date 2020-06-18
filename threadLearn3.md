---
title: 多线程与高并发（三）Volatile关键字
date: 2020-05-01 00:01:09
tags: Java
---

> 马士兵老师的多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。

（三）Volatile关键字

# java内存模型的承诺
- 原子性（Atomicity）
    - 由Java内存模型来直接保证的原子性变量操作包括read、load、use、assign、store和write六个，大致可以认为基础数据类型的访问和读写是具备原子性的。
    - 如果应用场景需要一个更大范围的原子性保证，Java内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock与unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐匿地使用这两个操作，这两个字节码指令反映到Java代码中就是同步块---synchronized关键字，因此在synchronized块之间的操作也具备原子性。

- 可见性（Visibility）
    - 可见性就是指当一个线程修改了线程共享变量的值，其它线程能够立即得知这个修改。Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方法来实现可见性的
    - 普通变量与volatile变量的区别是volatile的特殊规则保证了新值能立即同步到主内存，以及每使用前立即从内存刷新。因为我们可以说volatile保证了线程操作时变量的可见性，而普通变量则不能保证这一点。
    - 除了volatile之外，Java还有两个关键字能实现可见性，它们是synchronized和final。
        - synchronized同步块的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中(执行store和write操作)”这条规则获得的。
        - final关键字的可见性是指：被final修饰的字段是构造器一旦初始化完成，并且构造器没有把“this”引用传递出去，那么在其它线程中就能看见final字段的值。

- 有序性（Ordering）
    - Java内存模型中的程序天然有序性可以总结为一句话：如果在本线程内观察，所有操作都是有序的；如果在一个线程中观察另一个线程，所有操作都是无序的。前半句是指“线程内表现为串行语义”，后半句是指“指令重排序”现象和“工作内存主主内存同步延迟”现象。
    - Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性
        - volatile关键字本身就包含了禁止指令重排序的语义
        - synchronized则是由“一个变量在同一时刻只允许一条线程对其进行lock操作”这条规则来获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入。

# Volatile的两特性
## 可见性
- java内存模型：

    - Java内存模型(即Java Memory Model，简称JMM)本身是一种抽象的概念，并不真实存在，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式。
    - JVM运行的实体是线程，线程创建时JVM会为其创建一个工作内存，而JMM规定所有变量都保存在主内存中，供所有线程访问，属于公共空间。线程自己的工作空间对其他线程不可见。
    - 线程对工作变量的一次操作过程：从主内存拷贝到自己的工作内存，操作变量，再写回主内存中。过程如下：

        ![JMM](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E7%BA%BF%E7%A8%8B/JMM1.png)

    - 注意不要弄混java内存模型（虚拟概念）和硬件内存架构（真实硬件）之间的关系：多线程的执行最终都会映射到硬件处理器上进行执行，但Java内存模型和硬件内存架构并不完全一致，不管是工作内存的数据还是主内存的数据，对于计算机硬件来说都会存储在计算机主内存中，当然也有可能存储到CPU缓存或者寄存器中。下图表示是一种交叉的关系。
        ![java内存模型（虚拟概念）和硬件内存架构（真实硬件）之间的关系](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E7%BA%BF%E7%A8%8B/JMM%E4%B8%8E%E7%A1%AC%E4%BB%B6%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)

- Volatile保证可见性的原理：
    1. 使用volatile关键字会强制将修改的值立即写入主存；
    2. 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量   缓存行无效；
    3. 由于线程1的工作内存中缓存变量的缓存行无效，所以线程1再次去主存读取变量的值。

- 例证：
    ```java
    public class T1_Visibility {
        /* volatile */ boolean running =true;
        void m(){
            System.out.println("m start...");
            while (running) { }
            System.out.println("m end");
        }

        public static void main(String[] args) {
            T1_Visibility t = new T1_Visibility();
            new Thread(t::m,"t1").start();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            t.running=false;
        }
    }
    ```
    上述例子，在不给running添加volatile关键字时，线程不会停止运行，原因是main线程   修改running的值后，t1线程感知不到（没有去更新缓存）。当添加volatile关键字后，t1线程可终止。

## 有序性
举例：在单例模式的双重锁校验过程中的volatile分析：
```java
public class T2_Ordering {
    private /* volatile */ static T2_Ordering INSTANCE;

    private T2_Ordering(){}
    public static T2_Ordering getInstance() {
        if (INSTANCE == null) {                     //(1)
            synchronized (T2_Ordering.class) {      //(2)
                if (INSTANCE == null) {             //(3)
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    INSTANCE = new T2_Ordering();   //(4)
                }
            }
        }
        return INSTANCE;                            //(5)
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new Thread(()-> System.out.println(T2_Ordering.getInstance().hashCode())).start();
        }

    }
}
```
单例模式的 DCL（double check lock）是在 线程安全的懒汉式的基础上，将方法上的synchronized关键字，范围缩小到单例对象初始化的代码块中，提高了运行效率。

上述的DCL的写法中，添加了volatile的正常流程是：

- 线程t1与线程t2同时到达（1），单例对象还未初始化，同时竞争锁。
- 线程t1抢到锁，到（3），（4），初始化单例对象，后释放锁。
- 线程t2立马进入（2），但是被拦在（3）处，也释放锁，顺利拿到t1创建的对象。

但是在没有添加volatile的情况下，单例对象的初始化过程由于不是原子性操作，分为了下面三个步骤：
```
a.获取对象地址 
b.在对象地址上初始化一个T2_Ordering对象；
c.将INSTANCE引用指向对象地址
```
由于指令重排可能出现 a->b->c, 或者 a->c->b两种情况。当出现后面一种情况时，可能出错的情形如下：
- 线程t1到达（4）处，且经过a->c->b的步骤，刚刚进行到c, 此时INSTANCE指向的对象还没有被初始化，地址是1750862512（但其实这个位置还没有被初始化出对象）；
- 在这一瞬间，线程t2到达（1）处，INSTANCE不为空，是1750862512，则直接跳出到（5）拿对象。这时发现在对应的地址中，没有对象 -> 空指针异常。

添加了volatile则可以防止这种指令重排问题。（注：本例在实验环境中很难测试出出错的情形）。

## 原子性（无）
volatile不保证原子性，在一些非原子性操作的时候，还需要添加synchronized来保护线程安全，反例如下。这个例子输出结果可能会小于10000，分析如下：
```java
public class T3_Atomicity {
    public volatile int count = 0;

    public void increase() {
        count++;
    }
    public static void main(String[] args) {
        final T3_Atomicity t = new T3_Atomicity();
        for(int i=0;i<1000;i++){
            new Thread(()->{
                for(int j=0;j<10;j++){
                    t.increase();
                }
            }).start();
        }

        try { //这里写的不好,需要确认上面的步骤全部进行完毕
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(t.count);
    }
}
```
这是典型的count++的问题，count++不是原子性操作，包含了下面三个步骤：
- a.从主内存中读取，并拷贝到线程的工作内存中
- b.修改count值
- c.写回主内存

由于没有对increase方法加锁，可能会出现下面的错误情形：
- 线程t1从主内存中读取count的值，为10,准备进行修改操作。此时线程调度器激活线程t2,t2从主内存中读取到count的值为10。
- 线程t1进行count修改的操作，并写回主内存。
- 其他线程t3,t4再读到count的时候，均是11。但是t2已经将count读进自己的工作内存，不会重复在主内存中读一次，所以t2接着进行修改操作，写回主内存的还是count=11。

对比例1和例3：
- 例子1中，因为是while语句，线程会不断读取running的值来判断是否为false，每一次判断都是一个操作。这里是从缓存中读取。单个读取操作是具有原子性的，所以当例子1中的主线程修改了running时，由于volatile变量的可见性，线程1的 **下一个操作** 中再读取running时是最新的值，为true。
- 而例子3中，为什么自增操作会出现那样的结果呢？可以知道自增操作是三个原子操作组合而成的复合操作。 **在一个操作中**，读取了count变量后，是不会再读取的inc的，所以它的值还是之前读的10，它的下一步是自增操作。


>总结: volatile保证了可见性和有序性。可见性在CPU层级，使用的是缓存一致性来实现；有序性是在虚拟机层级，通过添加读写屏障实现。

参考资料：
- [深入理解Java虚拟机笔记---原子性、可见性、有序性](https://blog.csdn.net/xtayfjpk/article/details/41969915)
- [全面理解Java内存模型(JMM)及volatile关键字](https://blog.csdn.net/javazejian/article/details/72772461)