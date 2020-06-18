---
title: 多线程与高并发（六）Wait-Notify与LockSupport
date: 2020-05-09 17:03:58
tags: java
---

> 前文讲述了JUC同步器的用法，JUC同步器大都使用AQS作为实现方式。而java锁和同步器框架的核心AQS，就是通过LockSupport的`park()`、`unpark()`方法，来实现线程的阻塞和唤醒的。因此，本文关注`LockSupport`类的用法，以及其与`Object`类的`Wait()`、`Notify()`方法之间的区别（因为两者都能实现最基本的对线程的阻塞和唤醒）。最后比较了基于`synchronized`的`wait-notify`与基于`reentrantLock`的`Condition`的`await-signal`之间的区别。

（六）Wait-Notify与LockSupport

# Wait-Notify
## 用法
- `wait()`、`notify()`方法都要在`synchronized`块中使用，否则会抛出异常。三者使用的对象必须一致。
- `wait()`会释放锁。
- `notify()`并不会释放锁，得等`synchronized`块结束，才会释放锁。
- **线程A处于`wait()`状态时，其他线程调用了`notify()`方法，并不会立刻唤醒线程A，线程A首先需要拿到锁，才能接着`wait()`的地方运行。**

## 用例
- 例1：

    在本例中，线程t1希望打印出线程t2对count累加100次的结果。但是下面的这种写法事与愿违。因为线程t2，在`notify()`后，并未交还自己的`object锁`，而是继续进行。

    ```java
    public class T1_WaitNotify {
        static int count;
        static Object object = new Object();
        public static void main(String[] args) {
            new Thread(()->{
                synchronized (object){
                    System.out.println("waiting");
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("count is:"+ count);
                }
            },"t1").start();

            new Thread(()->{
                synchronized (object){
                    for (int i = 0; i < 100; i++) {
                        count++;
                    }
                    object.notify();
                    for (int i = 0; i < 100; i++) {
                        count++;
                    }
                }
            },"t2").start();
        }
    }
    ```
    输出：
    ```
    waiting
    count is:200
    ```

- 例2：

    在例2中对上述代码加以改进即可实现在线程t2累加100次时，让线程t1打印结果。方法是让线程t2在累加完100次时，交出`object锁`。

    ```java
    public class T1_WaitNotify {
        static int count;
        final static Object object = new Object();
        public static void main(String[] args) {
            new Thread(()->{
                synchronized (object){
                    System.out.println("waiting");
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("count is:"+ count);
                    object.notify();
                }
            },"t1").start();

            new Thread(()->{
                synchronized (object){
                    for (int i = 0; i < 100; i++) {
                        count++;
                    }
                    object.notify();
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    for (int i = 0; i < 100; i++) {
                        count++;
                    }
                }
            },"t2").start();
        }
    }
    ```

    输出
    ```
    waiting
    count is:100
    ```

# LockSupport
## 用法：
- 定义：
    - LockSupport类，是JUC包中的一个工具类，是用来创建锁和其他同步类的基本线程阻塞原语
    - LockSupport可以用于对线程进行阻塞和唤醒。核心方法：`park()`和`unpark()`，其中`park()`方法用来阻塞当前调用线程，`unpark()`方法用于唤醒指定线程。
    - 原理是使用Permit（许可）概念来实现阻塞和唤醒。
- 特点：
    - LockSupport不需要synchronized加锁就可以实现线程的阻塞和唤醒。
    - 可以先使用`unpark()`,再使用`park()`。（相当于先赋予一张令牌，再阻塞的时候就直接放行）
    - `park()`让线程进入`waiting`状态，`unpark()`让线程重新进入`Runnable`状态。
    - 调用`park()`让线程进入`waiting`状态后，如果再连续调用一次`park()`，线程将永远无法唤醒。

## 用例：
- 例1：

    先赋予一张令牌，再调用`park()`,则不阻塞，直接放行：

    ```java
    public class T2_LockSupport {
        public static void main(String[] args) {
            Thread t = new Thread(()->{
                for (int i = 0; i < 10; i++) {
                    System.out.println(i);
                    if (i==5){
                        LockSupport.park(); // 此处5s后执行
                    }
                    try {
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start(); 
            LockSupport.unpark(t);  //此处先执行完
        }
    }
    ```
- 例2：

    在例2中，如果不加注释2处，先`park()`后`unpark(t)`，线程正常阻塞与唤醒；加上了注释两处后，即是在线程`waitting`状态下继续调用`park()`,即使调用两次`unpark(t)`,线程也永远无法被唤醒。

    ```java
    public class T2_LockSupport {
        public static void main(String[] args) {
            Thread t = new Thread(()->{
                for (int i = 0; i < 10; i++) {
                    System.out.println(i);
                    if (i==5){
                        LockSupport.park();  //首次调用，线程已经处于waitting状态
                        // LockSupport.park();  //再次调用，线程将永远无法被唤醒。
                    }
                    try {
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start();
            try {
                TimeUnit.SECONDS.sleep(10);
                LockSupport.unpark(t);
                // LockSupport.unpark(t);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    ```

# 两道面试题

LockSupport相比于Wait-Notify，不需要使用`synchronized`加锁就能实现线程的阻塞与唤醒。下面在两道题目中，让二者实现同样的功能，进行比较。

## 1. 实时监控，即时打印
- 要求：实现一个容器，提供两个方法`add`,`size`。需要有两个线程。线程1：不停添加10个元素到容器中；线程2：实时监控容器的size，个数到5时，及时打印，结束线程2.

- 使用wait-notify实现：

    ```java
    public class T3_NotifyHoldingLock {
        volatile List list = new ArrayList();
        public void add(Object o){
            list.add(o);
        }
        public int size(){
            return list.size();
        }

        public static void main(String[] args) {
            T3_NotifyHoldingLock t = new T3_NotifyHoldingLock();
            final Object lock = new Object();
            new Thread(()->{
                synchronized (lock){
                    System.out.println("t2启动");
                    if (t.size()!=5){
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println("t2 结束");
                    }
                    lock.notify();
                }
            },"t2").start();

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            new Thread(()->{
                System.out.println("t1 开始");
                synchronized (lock){
                    for (int i = 0; i < 10; i++) {
                        t.add(new Object());
                        System.out.println("add "+ i);
                        if (t.size()==5){
                            lock.notify();
                            try {
                                lock.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                }
            },"t1").start();
        }
    }
    ```

- 使用LockSupport实现：

    ```java
    public class T3_LockSupport {
        volatile List list = new ArrayList();
        public void add(Object o){
            list.add(o);
        }
        public int size(){
            return list.size();
        }

        static Thread t1=null, t2=null;
        public static void main(String[] args) {
            T3_LockSupport t = new T3_LockSupport();
            t2 = new Thread(()->{
                System.out.println("t2 启动");
                if (t.size()!=5){
                    LockSupport.park();
                }
                System.out.println("t2 结束");
                LockSupport.unpark(t1);
            },"t2");
            t2.start();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            t1 = new Thread(()->{
                System.out.println("t1 开始");
                for (int i = 0; i < 10; i++) {
                    t.add(new Object());
                    System.out.println("add "+ i);
                    if (t.size()==5){
                        LockSupport.unpark(t2);
                        LockSupport.park();
                    }
                }
            });
            t1.start();
        }
    }
    ```

- 其他实现方式：还可以用`CountDownLatch(1)`来实现锁的功能。

## 2. 生产者-消费者 （固定容量的同步容器）
- 要求：实现一个固定容量的同步容器，拥有get,put方法，具有支持2个生产者线程和10个消费者线程的同步调用。

- 使用`synchronized`的`wait-notify`方法实现线程的阻塞与唤醒：
    ```java
    public class T4_MyContainer_waitNotify<T> {
        final private LinkedList<T> lists = new LinkedList<>();
        final private int MAX = 10;

    //    生产者
        public synchronized void put(T t){
            while (lists.size()==MAX){ // 思考此处为什么不用 if
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            lists.add(t);
             // 此处唤醒的是所有阻塞的线程，而不针对消费者线程，是一个缺点。
            this.notifyAll();
        }
    //    消费者
        public synchronized T get(){
            T t = null;
            while(lists.size()==0){
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            t = lists.removeFirst();
            this.notifyAll();
            return t;
        }

        public static void main(String[] args) {
            T4_MyContainer_waitNotify<String> c = new   T4_MyContainer_waitNotify<>();
            for (int i = 0; i < 10; i++) {
                new Thread(()->{
                    for (int j = 0; j < 5; j++) {
                        System.out.println(c.get());
                    }
                },"Consumer"+i).start();
            }
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            for (int i = 0; i < 2; i++) {
                new Thread(()->{
                    for (int j = 0; j < 25; j++) {
                        c.put(Thread.currentThread().getName()+" "+j);
                    }
                },"Producer"+i).start();
            }
        }
    }
    ```

    上面的实现方式，有两个需要思考的地方：
    - 为什么使用`while`循环来检查，而不使用`if`检查：当容器满时，生产者进入wait等待，只有当其他线程唤醒它时，才能继续`add`元素。试想此时恰好其他生产者在同时将容器又填满了，那么本线程再`add`就会出错。所以在`add`之时需要再次检查。
    - 本例中，生产者线程，会使用`notifyAll()`唤醒所有线程，而然只需要唤醒消费者线程即可（如果唤醒的是生产者线程，则又会重复`wait`,浪费资源。），同理，消费者线程也会使用`notifyAll()`唤醒所有线程。

- 使用`reentrantLock`的`condition`加以改进
    ```java
    public class T4_MyContainer_ReentrantLock<T> {
        final private LinkedList<T> lists = new LinkedList<>();
        final private int MAX =10;
        ReentrantLock reentrantLock = new ReentrantLock();
        Condition consumer = reentrantLock.newCondition();
        Condition producer = reentrantLock.newCondition();

        //生产者
        public void put(T t){
            try {
                reentrantLock.lock();
                while(lists.size()==MAX){
                    producer.await();
                }
                lists.add(t);
                consumer.signalAll();  //此处只唤醒消费者
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                reentrantLock.unlock();
            }
        }
        //消费者
        public T get(){
            T t= null;
            try {
                reentrantLock.lock();
                while(lists.size()==0){
                    consumer.await();
                }
                t=lists.removeFirst();
                producer.signalAll(); //此处只唤醒生产者
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                reentrantLock.unlock();
            }
            return t;
        }

        public static void main(String[] args) {
            T4_MyContainer_ReentrantLock c = new T4_MyContainer_ReentrantLock   ();
            for (int i = 0; i < 10; i++) {
                new Thread(()->{
                    for (int j = 0; j < 5; j++) {
                        System.out.println("消息内容： "+ c.get());
                    }
                },"Consumer"+i).start();
            }
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            for (int i = 0; i < 2; i++) {
                new Thread(()->{
                    for (int j = 0; j < 25; j++) {
                        c.put(Thread.currentThread().getName()+" "+j);
                    }
                },"Producer"+i).start();
            }
        }
    }
    ```

    上面的例子可以看出，`reentantLock`比`synchronized`更加精细，可以控制哪些线程需要唤醒。
    - `Condition`本质就是一个等待队列。上面的例子中包含了两个等待队列。比起`synchronized`只有一个等待队列，更加精确。


> 后记：本文介绍了`LockSupport`,和`object`的`wait-notify`。`wait-notify`需要在`sychronized`锁中使用，`LockSupport`可以直接使用。 同样在锁中，又比较了`synchronized`与`reentrantLock`之间的区别，前者只包含一个等待队列，后者则可以使用`condition`创建多个等待队列，对线程的唤醒，更加精确。后文则从AQS的原理和数据结构入手进行分析。