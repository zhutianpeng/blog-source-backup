---
title: 多线程与高并发（四）CAS与Atomic
date: 2020-05-01 23:51:44
tags: Java
---

> 多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。

（四）CAS与Atomic

# CAS是什么
线程安全性设计上，包括四种方式:
- 不可变：
    - 变量不可变，则不需要考虑线程安全问题：
    - 举例：final修饰、String、枚举类型、Number部分子类：Long，Double，BigInteger, BigDecimal 等
- 阻塞同步（互斥同步） （悲观锁）
    - synchronized
    - ReentranLock （重入锁）
- 非阻塞同步 （乐观锁）
    - 先进行操作，判断没有其他线程共享资源，则操作成功；否则采取补偿措施（重试直到成功），此之谓非阻塞。
    - 乐观锁需要 硬件支持原子性操作：`CAS(compare and set)`
- 无同步方案
    - 栈封闭（多线程访问方法的局部变量，属于线程私有的，没有安全性问题）
    - 线程本地存储（将需要共享数据的 多块代码，放进一个线程里面，一次性搞定，例如“web的request就是一个thread”）

可以看出乐观锁的实现方式需要基于CAS，(Compare-and-Swap)，又叫“无锁”，是**原子性**的操作。

这里对CAS的底层实现原理不做深入分析,只做简单描述。

- 在java语言层面：

    对于`AtomicInteger`的`incrementAndGet()`方法而言，采用了CAS。`AtomicInteger`类调用了`Unsafe`类的如下方法。最终调用的是`native`方法 `compareAndSwapInt`。

    ```java
    // class AtomicInteger
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    ```
    ```java
    // class Unsafe
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
    ```
    ```java
    // class Unsafe
    public final native boolean compareAndSwapInt(Object var1, long var2,int var4, int var5);
    ```
    将上述方法做一个转换即是下面的表达，意思是：如果obj内的value和expect相等，就证明没有其他线程改变过这个变量，那么就更新它为update；反之，如果进入compareAndSwapInt方法后，发现obj内的value已经和expect不一致了（被其他线程修改过了），那就采用自旋的方式继续进行CAS操作（补偿一次操作，是乐观锁的思想）。
    ```java
    // compareAndSwapInt（var1, var2, var5, var5 + var4）
    // 转换之后的代码
    compareAndSwapInt（obj, offset, expect, update）
    ```
- JDK源码层：
    
    可以参考[这篇博客](https://juejin.im/post/5a73cbbff265da4e807783f5)。

# CAS的应用 - Atomic类

`concurrent.atomic`包下的工具类都用到了CAS。下面的例子是典型的`count++`的操作，如果用悲观锁来实现，需要在`m()`方法加上`synchronized`关键字，有可能升级为重量级的锁，没有本例中的`AtomicInteger`的执行效率高。

```java
//atomic类方法本身是原子性的，但是不能保证多个方法连续调用的原子性。
public class T1_AtomicInteger {
    AtomicInteger count = new AtomicInteger(0);

    void m(){
        for (int i = 0; i < 10000; i++) {
            count.incrementAndGet();
        }
    }

    public static void main(String[] args) {
        T1_AtomicInteger t = new T1_AtomicInteger();
        List<Thread> threadList = new ArrayList<Thread>();
        for (int i = 0; i < 10; i++) {
            threadList.add(new Thread(t::m,"thread-"+i));
        }
        threadList.forEach((o)->o.start());

        threadList.forEach((o)->{
            try {
                o.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        System.out.println(t.count);
    }
}
```

# CAS的问题
- ABA问题
    - 起因：在执行CAS操作时，目标值被其他线程 修改了两次 A->B->A, 对于基本类型没有影响。对于引用类型，就是中间经历了别的对象。
    - 解决方式：添加版本号：`atomic`中的`AtomicStampedReference`类可以用于解决此问题。

- 自旋时间开销大问题
    - CAS执行不成功，如果长时间自旋会给cpu带来较大的执行开销（在大量线程的情况下）。下面的例子`T2_Sync_Atomic_LongAddr`可以看出。

# 多线程中自增的三种写法
多线程中的自增是最常见的一种情况，比如说秒杀，有如下三种线程安全的实现方式：
- synchronized
- atomic类
- LongAdder类

```java
public class T2_Atomic_Sync_LongAddr {
    static long count1 = 0;
    static AtomicLong count2 = new AtomicLong(0L);
    static LongAdder count3 = new LongAdder();

    static long m(int testType,int testNum) {
        Thread[] threads = new Thread[1000];

        switch (testType) {
            case 1:
                Object object = new Object();
                for (int i = 0; i < threads.length; i++) {
                    threads[i] = new Thread(() -> {
                        for (int j = 0; j < testNum; j++) {
                            synchronized (object) {
                                count1++;
                            }
                        }
                    });
                }
                break;
            case 2:
                for (int i = 0; i < threads.length; i++) {
                    threads[i] = new Thread(() -> {
                        for (int j = 0; j < testNum; j++) {
                            count2.incrementAndGet();
                        }
                    });
                }
                break;
            case 3:
                for (int i = 0; i < threads.length; i++) {
                    threads[i] = new Thread(() -> {
                        for (int j = 0; j < testNum; j++) {
                            count3.increment();
                        }
                    });
                }
        }

        long startTime = System.currentTimeMillis();
        for (Thread t : threads) t.start();
        for (Thread t : threads) {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        long endTime = System.currentTimeMillis();

        return endTime - startTime;
    }

    public static void main(String[] args) {
//        System.out.println("m1: " + m(1,1000));
//        System.out.println("m2: " + m(2,1000));
//        System.out.println("m3: " + m(3,1000));

        System.out.println("m1: " + m(1,10000));
        System.out.println("m2: " + m(2,10000));
        System.out.println("m3: " + m(3,10000));

//        System.out.println("m1: " + m(1,100000));
//        System.out.println("m2: " + m(2,100000));
//        System.out.println("m3: " + m(3,100000));
    }
```
上述例子，分别在三种CASE中，对1000个线程，每个线程中进行`若干次`自增操作，来模拟较高的并发情况。三种CASE分别由`synchronized`、`atomic`和`LongAdder`来实现。经过粗糙的测试有如下的结果：
```
//1000次
m1: 93
m2: 73
m3: 75

//10000次
m1: 553
m2: 219
m3: 95

//100000次
m1: 1800
m2: 2006 
m3: 246  // 可以发现在大量线程进行测试情况下，LongAddr类效率较高。
```
上述时间单位为ms，可以发现：
- 1000次及以下的时候：三者差异不大，执行效率：atomic ≈ LongAdder > synchronized
- 10000次：执行效率：atomic ≈ LongAdder > synchronized
- 100000次：执行效率：LongAdder > synchronized > atomic 

究其原因：
- 在并发量较小的情况下：乐观锁的执行效率高于悲观锁，因为乐观锁的执行成功率较高。乐观锁，线程较少，自旋的情况较少；而悲观锁synchronized可能锁升级，去操作系统申请重量级锁，这种情况下较为耗时。
- 在并发量较大的情况下：在竞争激烈的情况下，CAS操作不断的失败，就会有大量的线程不断的自旋尝试 CAS 会造成 CPU 的极大的消耗。CAS此时暴露出“自旋时间开销大”的问题。
- LongAddr整体表现比较优秀，其原因是使用了分段锁。

# LongAddr类
- 特点：

    参看`LongAddr`的javadoc：

    > This class is usually preferable to {@link AtomicLong} when multiple threads update a common sum that is used for purposes such as collecting statistics, not for fine-grained synchronization control.  Under low update contention, the two classes have similar characteristics. But under high contention, expected throughput of this class is significantly higher, at the expense of higher space consumption.

    LongAddr在在低更新争用情况下，和AtomicLong类相比具有相似的特性；高争用的情况下，LongAddr类的预期吞吐量会显著提高，但会牺牲更高的空间消耗。与AtomicLong相比，LongAddr更适合去做 统计数据的收集，而不是细粒度的锁控制。

    总结来看，LongAddr的优缺点如下：
    - 优点：解决了 AtomicLong 类，在极高竞争下的性能问题。
    - 缺点：对于自增ID的生成，不适合再使用LongAddr,因为LongAdder 在统计的时候如果   有并发更新，可能导致统计的数据有误差。此时首选AtomicLong。

- 底层实现：
    - 使用了一个 cell 列表去承接并发的 cas, 分散竞争，逐步升级竞争。
    - 详细实现可以参考：[从 LongAdder 中窥见并发组件的设计思路](https://xilidou.com/2018/11/27/LongAdder/)