---
title: 多线程与高并发（二）Synchronized关键字
date: 2020-04-30 01:12:05
tags: Java
---
> 马士兵老师的多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。

（二）Synchronized关键字

# 概念
synchronized 是 Java 中的关键字，是利用锁的机制来实现同步的。
锁机制有如下两种特性：

- 互斥性：即在同一时间只允许一个线程持有某个对象锁，通过这种特性来实现多线程中的协调机制，这样在同一时间只有一个线程对需同步的代码块(复合操作)进行访问。互斥性我们也往往称为操作的原子性。

- 可见性：必须确保在锁被释放之前，对共享变量所做的修改，对于随后获得该锁的另一个线程是可见的（即在获得锁时应获得最新共享变量的值），否则另一个线程可能是在本地缓存的某个副本上继续操作从而引起不一致。

# 用法
- 对象锁：
    - 目的：持有这个对象的线程争抢这把锁。
    - 用法：
        - 修饰非静态方法
        - 修饰代码块
- 类锁：
    - 目的：持有这个类的所有对象的线程，争抢同一把锁，因为T.class是唯一的。
    - 用法：
        - 修饰静态方法
        - 修饰代码块

# 用法实例
## 1. 对象锁
object 对象锁修饰代码块
```java
public class T1_Synchronized1{
    private int count =10;
    private Object object = new Object(); //使用object这个对象当锁

    public void m() {
        synchronized (object){
            count--;
            System.out.println(Thread.currentThread().getName()+" count = "+ count);
        }
    }

    public static void main(String[] args) {
        T1_Synchronized1 t = new T1_Synchronized1();
        for (int i = 0; i < 10; i++) {
            // 这句话如果放这，就相当于10个线程持有10个不同的对象，不是同一把锁
            // T1_Synchronized1 t = new T1_Synchronized1(); 
            new Thread(()->{
                t.m();
            }).start();
        }
    }
}
```

本对象修饰代码块, 和上述一样的功能，锁定的是某一个对象，不是代码
```java
public class T1_Synchronized2 {
    private int count =100;

    public void m() {
        synchronized (this){
            count--;
            System.out.println(Thread.currentThread().getName()+" count = "+ count);
        }
    }

    public static void main(String[] args) {
        T1_Synchronized2 t = new T1_Synchronized2();
        for (int i = 0; i < 100; i++) {
            new Thread(()->{
                t.m();
            }).start();
        }
    }
}
```
对象锁修饰非静态方法（如果是修饰静态方法，就是用的类锁了）
```java
public class T1_Synchronized3{
    private int count =100;

    public synchronized void m(){
        count--;
        System.out.println(Thread.currentThread().getName()+" count = "+ count);
    }
    public static void main(String[] args) {
        T1_Synchronized3 t = new T1_Synchronized3();
        for (int i = 0; i < 100; i++) {
            new Thread(()->{
                t.m();
            }).start();
        }
    }
}
```

## 2. 类锁

类锁的目的是持有这个类的所有对象的线程，争抢同一把锁，下面三种方式的作用相同：
- 修饰静态方法
- 修饰静态代码块
- 修饰非静态代码块

```java
public class T1_Synchronized4{
    private static int count =100;
    // 修饰静态方法
    public synchronized static void m1(){
        count--;
        System.out.println(Thread.currentThread().getName()+" count = "+count);
    }
    // 修饰静态代码块
    public static void m2(){
        synchronized (T1_Synchronized4.class){
            count--;
            System.out.println(Thread.currentThread().getName()+" count = "+count);
        }
    }
    // 修饰非静态代码块
    public void m3(){
        synchronized (T1_Synchronized4.class){
            count--;
            System.out.println(Thread.currentThread().getName()+" count = "+count);
        }
    }
    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
//            T1_Synchronized4 t = new T1_Synchronized4();
            new Thread(()->{
                T1_Synchronized4.m1();
//                T1_Synchronized4.m2();
//                t.m3();
            }).start();
        }
    }
}

```

# 脏读问题
## 1. 同步方法和非同步方法同时调用
同步方法和非同步方法是可以同时调用的，互不影响
```java
public class T2_Synchronized_noSynchronized {
    public synchronized void m1(){
        System.out.println(Thread.currentThread().getName()+" m1 start...");
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+" m1 end");
    }
    public void m2(){
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+" m2 ");
    }

    public static void main(String[] args) {
        T2_Synchronized_noSynchronized t = new T2_Synchronized_noSynchronized();
        new Thread(t::m1).start();
        new Thread(t::m2).start();

        /* 上述表达相当于
         new Thread(()->t.m1()).start();   
        */
        
        /*  jdk1.8以前的写法
        new Thread(new Runnable() {
            @Override
            public void run() {
                t.m1();
            }
        }).start();
        */
    }
}
```
同步方法和非同步方法同时调用，引申一下，可能导致下面的脏读问题。

## 2. 脏读
将上述的加锁的方法和不加锁的方法，转换成为对 Account的写方法和读方法，则会出现脏读问题，若给读方法加锁，则不会出现脏读问题。
- 给读方法加锁：读效率较低
- 不给读方法加锁：可能出现脏读

主要看业务是否允许脏读，隔离等级如何。

```java
public class T2_Account {
    double balance;

    public synchronized void set(double balance){
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.balance=balance;
    }
    public /* synchronized */ double getBalance(){
        return this.balance;
    }

    public static void main(String[] args) {
        T2_Account account = new T2_Account();
        new Thread(()-> account.set(100.0)).start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(account.getBalance());
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(account.getBalance());
    }
}
```

# 可重入特点
- 定义：当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功，否则阻塞。
- synchronized 是可重入锁
- 实例验证：

    ```java
    public class T3_ReentrantTest {
        synchronized void m(){
            System.out.println("father start...");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("father end");
        }

        public static void main(String[] args) {
            new T3_ReentrantTestChild().m();
        }
    }

    class T3_ReentrantTestChild extends T3_ReentrantTest{
        @Override
        synchronized void m(){
            System.out.println("child start...");
            super.m();
            System.out.println("child end");
        }
    }
    ```
    输出：
    ```
    child start...
    father start...
    father end
    child end
    ```
    如果不是可重入锁，会出现死锁。

- 实现原理概述：每一个锁关联一个线程持有者和计数器，当计数器为 0 时表示该锁没有被任何线程持有，那么任何线程都可能获得该锁而调用相应的方法；当某一线程请求成功后，JVM会记下锁的持有线程，并且将计数器置为 1；此时其它线程请求该锁，则必须等待；而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增；当线程退出同步代码块时，计数器会递减，如果计数器为 0，则释放该锁。

# 异常时，锁是否中断
- 程序执行过程中如果出现异常，默认情况下锁会被释放
- 在web app中，多个servlet访问同一个资源，如果异常没有处理合适，在第一个线程出现异常时，其他等待的线程会涌入同步代码区，有可能访问到异常的数据。
- 要非常小心处理 同步业务逻辑中的异常
- 实例验证：

```java
public class T4_Exception {
    static int count=0;
    synchronized static void m(){
        System.out.println(Thread.currentThread().getName()+" start...");
        while(true){
            count++;
            System.out.println(Thread.currentThread().getName()+ " count = " + count);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            if (count==5) {   
                int i=1/0;  //会出现异常的区域
            }
        }
    }

    public static void main(String[] args) {
        new Thread(T4_Exception::m,"t1").start();
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(T4_Exception::m,"t2").start();
    }
}
```

按照程序设计，t1线程在运行时，t2线程永远不会开始。但是由于t1线程出现了异常退出，t2乱入。输出如下：

```
t1 start...
t1 count = 1
t1 count = 2
t1 count = 3
t1 count = 4
t1 count = 5
Exception in thread "t1" t2 start...
t2 count = 6
java.lang.ArithmeticException: / by zero
	at c2.T4_Exception.m(T4_Exception.java:20)
	at java.lang.Thread.run(Thread.java:748)
t2 count = 7
t2 count = 8
t2 count = 9
```

改进方法是将可能出现异常的区域 int i=1/0， 使用try...catch包裹，之后t1的循环会继续。修改后输出如下：

```
t1 start...
t1 count = 1
t1 count = 2
t1 count = 3
t1 count = 4
t1 count = 5
t1 count = 6
java.lang.ArithmeticException: / by zero
	at c2.T4_Exception.m(T4_Exception.java:21)
	at java.lang.Thread.run(Thread.java:748)
t1 count = 7
t1 count = 8
```

# 以对象作锁，最好是final
```java
public class T6_ObjectLock_Final {
    /* final */ Object object = new Object();
    
    void m(){
        synchronized (object) {
            while (true) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }
        }
    }

    public static void main(String[] args) {
        T6_ObjectLock_Final t = new T6_ObjectLock_Final();
        new Thread(t::m,"t1").start();

        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        Thread t2 = new Thread(t::m,"t2");
        //如果锁对象是final的，这句话不通过，t2将没有机会运行
        t.object = new Object(); 
        t2.start();
    }
}
```
锁是由对象头上的两位来作标识的，修改了对象，就去访问别的对象的两位了，上一个对象的两位就没有参考意义了（锁就没用了）。以对象作为锁的时候，不要让其改变，加上final修饰。

# 锁升级

锁的底层实现，和锁的优化过程，可以参照 [这篇博客](https://blog.csdn.net/baidu_38083619/article/details/82527461)
和[这篇博客](https://blog.csdn.net/tongdanping/article/details/79647337)

# 锁优化

关于使用synchronized进行优化的内容，参照这篇博客

