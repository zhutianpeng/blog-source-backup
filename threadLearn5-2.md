---
title: 多线程与高并发（五）JUC同步工具(2)
date: 2020-05-07 23:32:21
tags: java
---

> 多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。

（五）JUC同步工具（2）

对于JUC同步工具的学习思路，首先从用法开始了解，之后再探讨原理。本文主要关注各同步工具的用法，后续文章逐渐深入原理。本文讨论的JUC同步工具包括：
- ReentrantLock
- CountDownLatch
- CyclicBarrier
- `Phaser`
- `ReadWriteLock`
- `Semaphore`
- `Exchanger`

# Phaser - 多阶段栅栏
## 用法：
- `Phaser`的功能与 CyclicBarrier和CountDownLatch有些类似，类似于一个多阶段的栅栏。是比较复杂的一种同步器。
- `Phaser`中的一些概念：
    - phase（阶段）：栅栏的名称叫做phase，在任意时间点，Phaser只处于某一个phase(阶段)，初始阶段为0，最大达到Integerr.MAX_VALUE，然后再次归零。当所有parties参与者都到达后，phase值会递增。
    - parties(参与者)：CyclicBarrier中的参与者在初始构造指定后就不能变更，而Phaser既可以在初始构造时指定参与者的数量，也可以中途通过register、bulkRegister、arriveAndDeregister等方法注册/注销参与者。
    - unarrive(未到达) / arrive(到达) / advance(进阶)：
        - Phaser注册完parties（参与者）之后，参与者的初始状态是unarrived
        - 当某个参与者到达（arrive）当前阶段（phase）后，该参与者状态就会变成arrived
        - 当阶段的到达参与者数满足条件后（注册的数量等于到达的数量），阶段就会发生进阶（advance）——也就是phase值+1。
- 比较：

| 同步器 | 特点 | 用法 | 
| :-----| :---- | :---- |  
| CountDownLatch | 倒数门栓 |初始时设定参与者个数，参与线程执行到`await`方法时等待。当计数器归零时，所有待线程放行; | 
| CyclicBarrier | 循环栅栏 | 当参与者线程到达`await`方法时等待。当到达线程数满足`parties`个数时放行。与数门栓相比，可以循环使用。| 
| Phaser | 多阶段栅栏 |可以在初始时设定参与线程数，也可以中途注册/注销参与者，当到达的参与者数量满足栅栏设的数量后，会进行阶段升级| 


## 用例：

本例使用`Phaser`来管理多线程的几个阶段。7个人（5个参与者线程，1个新郎线程，1个新娘线程）参与一项婚礼，分为4个步骤进行：到场，吃饭，离开，洞房。要求是所有线程全部完成某一阶段，才能进入下一阶段。（相当于分阶段控制）；而且不同的阶段，还能按照不同的参与者的身份进行控制（洞房阶段只有新郎新娘两人参与，其他人不能参与）。下面是具体的实现：

```java
public class T4_Phaser {
    static Random random = new Random();
    static MarriagePhaser phaser = new MarriagePhaser();

    static void milliSleep(int milli){
        try {
            TimeUnit.MILLISECONDS.sleep(milli);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        phaser.bulkRegister(7);
        for (int i = 0; i < 5; i++) {
            new Thread(new Person("p"+i)).start();
        }
        new Thread(new Person("新郎")).start();
        new Thread(new Person("新娘")).start();
    }

    static class MarriagePhaser extends Phaser {
        /*  Override了onAdvance：
        最后一个参与者到达时，会触发onAdvance方法,
        相当于CyclicBarrier中的barrierAction任务。*/
        @Override
        protected boolean onAdvance(int phaser, int registerParties){
            switch(phaser){
                case 0:
                    System.out.println("所有人到齐了，人数："+registerParties);
                    System.out.println();
                    return false;
                case 1:
                    System.out.println("所有人都吃完了，人数："+registerParties);
                    System.out.println();
                    return false;
                case 2:
                    System.out.println("所有人都离开了，人数："+ registerParties);
                    System.out.println();
                    return false;
                case 3:
                    System.out.println("婚礼结束，新郎新娘抱抱。人数："+ registerParties);
                    return true;
                default:
                    return true;
            }
        }
    }

    static class Person implements Runnable{
        String name;
        public Person(String name){
            this.name = name;
        }
        public void arrive(){
            milliSleep(random.nextInt(1000));
            System.out.printf("%s 到达现场 \n", name);
            phaser.arriveAndAwaitAdvance(); // 等待其它参与者线程到达
        }
        public void eat(){
            milliSleep(random.nextInt(1000));
            System.out.printf("%s 吃完 \n",name);
            phaser.arriveAndAwaitAdvance(); // 等待其它参与者线程到达
        }
        public void leave(){
            milliSleep(random.nextInt(1000));
            System.out.printf("%s 离开 \n",name);
            phaser.arriveAndAwaitAdvance(); // 等待其它参与者线程到达
        }
        public void hug(){
            if(name.equals("新郎")||name.equals("新娘")){
                milliSleep(random.nextInt(1000));
                System.out.printf("%s 洞房 \n",name);
                phaser.arriveAndAwaitAdvance(); // 等待其它参与者线程到达
            }else{
                phaser.arriveAndDeregister();
            }
        }
        @Override
        public void run() {
            arrive();;
            eat();
            leave();
            hug();
        }
    }
}
```

<details>
<summary>输出</summary>
<pre>
p1 到达现场 
新郎 到达现场 
p4 到达现场 
p3 到达现场 
新娘 到达现场 
p0 到达现场 
p2 到达现场 
所有人到齐了，人数：7

新郎 吃完 
p1 吃完 
新娘 吃完 
p0 吃完 
p3 吃完 
p4 吃完 
p2 吃完 
所有人都吃完了，人数：7

新娘 离开 
p2 离开 
p4 离开 
p1 离开 
新郎 离开 
p3 离开 
p0 离开 
所有人都离开了，人数：7 

新郎 洞房 
新娘 洞房 
婚礼结束，新郎新娘抱抱。人数：2
</pre>
</details>

# ReadWriteLock - 读写锁
## 锁的特性
- 可重入(Reentrant)：`synchronized`和`ReentrantLock`具有可重入性
- 可中断/不可中断：`synchronized`不可中断，`Lock`接口可中断
- 公平/非公平：`synchronized`是非公平锁。`ReentrantLock`和`ReentrantReadWriteLock`，在默认情况下是非公平锁，可以设置为公平锁。
- 读写锁/互斥锁：为了解决锁的互斥与共享引入的概念。`synchronized`获取锁的线程一律互斥。`ReentrantReadWriteLock`具有读与读之间共享的特点。

## 用法
### ReadWriteLock接口：
- 定义：ReadWriteLock接口是一个单独的接口（未继承Lock接口），该接口提供了获取读锁和写锁的方法: `readLock()`和`writeLock()`。

- 特点：

|  | 读锁 | 写锁 |
| :----:| :----: | :----: | 
| 读锁| 非阻塞 | 阻塞 | 
| 写锁|  阻塞 |  阻塞 | 


- 使用场景：读多写少、读较为耗时

### ReentrantReadWriteLock 实现类
- 定义： ReentrantReadWriteLock类是通过定义内部类实现AQS框架的API来实现独占/共享的功能。从而实现ReadWriteLock接口。

- 特点：
    - 支持公平/非公平策略
    -  支持锁重入
        - 同一读线程在获取了读锁后还可以获取读锁；
        - 同一写线程在获取了写锁之后既可以再次获取写锁又可以获取读锁；
    - 支持锁降级
        - 先获取写锁，然后获取读锁，最后释放写锁，这样写锁就降级成了读锁。
        - 但是，读锁不能升级到写锁。
    - Condition条件支持

## 用例
本例中，主要比较读写锁与互斥锁的执行效率：

```java
public class T5_ReadWriteLock {
    private static int value;
    // 1. 直接使用互斥锁
    static Lock lock = new ReentrantLock(); 
    // 2. 使用读写锁
    static ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    static Lock readLock = readWriteLock.readLock();
    static Lock writeLock = readWriteLock.writeLock();

    public static void read(Lock lock){
        try {
            lock.lock();
            Thread.sleep(1000);
            System.out.println("read over, value="+value);
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public static void write(Lock lock, int v){
        try {
            lock.lock();
            Thread.sleep(1000);
            value = v;
            System.out.println("write over, set value to: "+value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        //1. 直接使用互斥锁
        // Runnable readR = ()->read(lock);
        // Runnable writeR = ()->write(lock,new Random().nextInt());

        //2. 使用读写锁
        Runnable readR = ()->read(readLock);
        Runnable writeR = ()->write(writeLock,new Random().nextInt());

        for (int i = 0; i < 2; i++) new Thread(writeR).start();
        for (int i = 0; i < 18; i++) new Thread(readR).start();
    }
}
```

效率比较：

使用读写锁效率更高，18个读线程可以在1秒完成，另外两个写线程各占1秒；使用互斥锁，所有线程相互互斥，各占1秒，一般需要20秒完成。读写锁在这个应用场景中，效率更高。

# semaphore - 限流信号量

## 用法：
- 读音： [ˈseməfɔːr] 
- Semaphore维护了一个许可集，其实就是一定数量的“许可证”。
- 当有线程想要访问共享资源时，需要先获取(acquire)的许可；如果许可不够了，线程需要一直等待，直到许可可用。当线程使用完共享资源后，可以归还(release)许可，以供其它需要的线程使用。
- 另外，Semaphore支持公平/非公平策略，这和ReentrantLock类似

## 构造器与接口申明
- 构造器：
    ```java
    // 创建一个包含permits个许可令的、非公平（默认）的信号量
    public Semaphore(int permits){}
    // 创建一个包含permits个许可令的、公平/非公平的信号量
    public Semaphore(int permits, boolean fair) {}
    ```
-  常用接口申明：
    ```java
    //获取令牌
    void acquire()
    void acquire(int permits)
    //释放令牌
    void release()
    ```

## 用例：
本例可以看出`Semaphore`具有限流的作用。当然了，当初始化为1时可以当做锁来用。
```java
public class T6_Semaphore {
    public static void main(String[] args) {
        Semaphore s = new Semaphore(2);
        new Thread(()->{
            try {
                s.acquire();
                System.out.println("t1 is running...");
                Thread.sleep(200);
                System.out.println("t1 is running...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                s.release();
            }
        },"t1").start();
        new Thread(()->{
            try {
                s.acquire();
                System.out.println("t2 is running...");
                Thread.sleep(200);
                System.out.println("t2 is running...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                s.release();
            }
        },"t2").start();
    }
}
```

# Exchanger - 交换器
## 用法
- 这个类主要作用是两个线程之间的交换数据。
- 先到的线程会阻塞等待后到的线程，交换完数据后，再同时放行。
- 应用:游戏中交换装备、可以实现简单的生产者-消费者模式。

## 用例

本例可以看出，先到的线程会阻塞等待后到的线程。
```java
public class T7_Exchanger {
    static Exchanger<String> exchanger = new Exchanger<>();

    public static void main(String[] args) {
        new Thread(()->{
            String s = "T1";
            try {
                System.out.println("t1 begins running");
                s = exchanger.exchange(s);
                System.out.println("t1 continue runs");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" "+s);
        },"t1").start();

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("main threads sleep");

        new Thread(()->{
            String s = "T2";
            try {
                System.out.println("t2 begins running...");
                s = exchanger.exchange(s);
                System.out.println("t2 continue runs.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" "+s);
        },"t2").start();
    }
}
```
<details>
<summary>输出</summary>
<pre>
t1 begins running
main threads sleep
t2 begins running...
t2 continue runs.
t1 continue runs
t2 T1
t1 T2
</pre>
</details>

# 总结
- ReentrantLock： 可重入锁
    - `lock.tryLock()`
    - `lock.lockInterruptibly()`
    - 可设置“公平/非公平”

- CountDownLatch： 倒数门栓
    - `await()` + `countDown()`

- CyclicBarrier - 循环栅栏
    - `CyclicBarrier(int parties, Runnable barrierAction)` + `await()`

- Phaser - 多阶段的栅栏
    - 参与者个数可变，达到数量后会进行阶段升级
    - `bulkRegister(int x)` + `arriveAndAwaitAdvance()` or `arriveAndDeregister()`

- ReadWriteLock - 读写锁
    - 读写锁 与 互斥锁的 概念
    - 实现类`ReentrantReadWriteLock`的特点

- Semophare - 限流信号量
    - `acquire()` + `release()`

- Exchanger - 交换器
    - 两个线程之间的阻塞交换
    - `exchanger(V x)`


# 参考资料：
- [透彻理解Java并发编程系列](https://segmentfault.com/a/1190000015558984)

> 后记：本着先实践后原理的原则，JUC同步工具（1）、（2）两文，不求甚解的把所有的J.U.C同步工具列举出来，重点放在了用法和用例上。后面的文章，会在几道题目中加深理解。对于其原理，这些同步工具大都是使用了AQS进行实现，后面会就AQS的源码和其数据结构进行分析。