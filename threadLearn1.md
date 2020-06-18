---
title: 多线程与高并发（一）线程的基本概念
date: 2020-04-29 00:56:24
tags: Java 
---

> 多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。

（一）线程的基本概念

# 创建线程的5种方式
```java
public class T1_HowToCreateThread {
    static class MyThread extends Thread {
        public void run() {
            System.out.println("MyThread");
        }
    }

    static class MyRun implements Runnable {
        public void run() {
            System.out.println("MyRun");
        }
    }

    static class MyCall implements Callable<String> {
        public String call() throws Exception {
            System.out.println("MyCall");
            return "success";
        }
    }

    //    启动线程的5种方式
    public static void main(String[] args) {
//        1. 继承Thread
        new MyThread().start();
//        2. 实现Runnable
        new Thread(new MyRun()).start();
//        3. Lamda表达式匿名内部类
        new Thread(()->{
            System.out.println("Thread3");
        }).start();
//        4. 带返回值的 FutureTask<>(Callable())
        new Thread(new FutureTask<String>(new MyCall())).start();
//        5. 线程池 创建的方式
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(()->{
            System.out.println("ThreadPool");
        });
        executorService.shutdown();
    }
}
```

# 线程的基本方法 sleep yield join
```java
public class T2_Sleep_Yield_Join {
    public static void main(String[] args){
//        testSleep();
//        testYield();
        testJoin();
    }

//  sleep:当前线程休眠一段时间，留给其他线程运行。睡眠时间到线程复活。
    static void testSleep(){
        new Thread(()->{
            for(int i=0;i<100;i++){
                System.out.println("A"+i);
                try{
                    Thread.sleep(500);
                }catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

//  yield: 当前线程从Running->Ready，然后和其他线程同时开始争夺资源，让出CPU，重新竞争。
//  可能是当前线程继续运行，更大可能是重等待队列中选出一个线程来运行。
    static void testYield(){
        new Thread(()->{
            for(int i=0;i<100;i++){
                System.out.println("A"+i);
                if(i%10==0){
                    Thread.yield();
                }
            }
        }).start();

        new Thread(()->{
            for(int i=0;i<100;i++){
                System.out.println("-------B"+i);
                if(i%10==0){
                    Thread.yield();
                }
            }
        }).start();
    }

//    join: t1正在运行，t2.join(),这时t1会等t2运行完再执行。（自己join自己没有意义）
    static void testJoin(){
        Thread t1 = new Thread(()->{
           for(int i=0;i<20;i++){
               System.out.println("A"+i);
               try{
                   Thread.sleep(50);
               }catch (InterruptedException e){
                   e.printStackTrace();
               }
           }
        });

        Thread t2 = new Thread(()->{
            try{
                t1.join();
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            for(int i=0;i<25;i++){
                System.out.println("B"+i);
                try{
                    Thread.sleep(50);
                }catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

# 线程的六种状态
![线程的六种状态](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E7%BA%BF%E7%A8%8B/%E7%BA%BF%E7%A8%8B%E8%BF%90%E8%A1%8C%E7%8A%B6%E6%80%81.png)
```java
public class T3_ThreadState {
    public static void main(String[] args) {
        Thread t = new Thread(()->{
           System.out.println(Thread.currentThread().getName() + ": "+ Thread.currentThread().getState());  // RUNNABLE
           for(int i=0;i<10; i++){
               try{
                   Thread.sleep(500);
               }catch (InterruptedException e){
                   e.printStackTrace();
               }
               System.out.println(i);
           }
        }，“0”);
        System.out.println("new线程的状态: "+t.getState());
        t.start();
        for(int i=0;i<100;i++){
            try{
                Thread.sleep(500); 
            }catch (InterruptedException e){
                e.printStackTrace();
            }
             System.out.println("线程0的状态: "+t.getState()+"  currentThread:"+Thread.currentThread().getName() + " currentThreadState: "+ Thread.currentThread().getState());
            //此处时而显示  线程0的状态: RUNNABLE ， 时而显示 线程0的状态: TIMED_WAITING
            //最终将显示： 线程0的状态: TERMINATED
        }
    }
}
```
输出：

![状态输出](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E7%BA%BF%E7%A8%8B/c1%E8%BE%93%E5%87%BA.png)

- New: Thread t = new Thread()。 线程被创建出来就是New状态。
- RUNNABLE: 线程在线程调度器执行的状态。（可以看到1处，主线程和0线程同时处于RUNNABLE状态，如果是单核CPU,那么可以猜想得知：0线程是处于Ready，main线程是Running；我的测试机是多核CPU, 两者的状态未知，可以都处于Running。）
- TIMED_WAITING: Thread.sleep(xx)。 在0线程sleep的时候，0线程即处于TIMED_WAITING状态，其状态与main线程是否sleep无关。

> 可见多线程的运行，跟机器的CPU的核数有关系。单核CPU采用时间片轮转方式来实现并发，而多核CPU能实现真正的并行。

# 并发、并行
- 并发（concurrency ）：时间段内有很多的线程或进程在执行，但何时间点上都只有一个在执行，多个线程或进程争抢时间片轮流执行，使其在宏观上具有多个线程同步执行的效果。
- 并行（parallelism ）：时间段和同一时间点上都有多个线程或进程在执行。

# 单核、多核
都是一个CPU，不同的是每个CPU上的核心数。一个核心只能同时执行一个线程。

- 单核：（并发）
    
    单核CPU上运行的多线程程序, 同一时间只能一个线程在跑, 系统帮你切换线程而已, 系统给每个线程分配时间片来执行, 每个时间片大概10ms左右, 看起来像是同时跑, 但实际上是每个线程跑一点点就换到其它线程继续跑，效率不会有提高，切换线程反倒会增加开销。

- 多核：（并行）

    多核CPU是多个CPU的替代方案，同时也减少了功耗。可以真正意义上做到并行。
    
    - 物理核：物理核心数量=CPU数(机器实装的CPU数)*每个CPU的核心数
    - 虚拟核：所谓的4核8线程，4核指的是物理核心，通过超线程技术，用一个物理核模拟两个虚拟核，每个核两个线程，总数为8线程（逻辑处理器）。
 
 windows下通过指令查看CPU个数和物理核心与线程数：
 ```
 # 查看cpu个数
systeminfo
 ```
 ```
 # 查看细节信息
wmic
cpu get *
# 或者：cpu get NumberOfCores, NumberOfLogicalProcessors
 ```

<div align="center">
<img src="https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E7%BA%BF%E7%A8%8B/win%E6%9F%A5%E7%9C%8B%E6%A0%B8%E5%BF%83.png" ><br>
<sup>NumberOfCores 物理核心,  NumberOfLogicalProcessors 线程数（逻辑处理器）</sup>
</div>

不同核数和任务适合的线程数：
- 多核CPU——计算密集型任务:此时要尽量使用多线程，可以提高任务执行效率，例如加密解密，数据压缩解压缩（视频、音频、普通数据），否则只能使一个核心满载，而其他核心闲置。
- 单核CPU——计算密集型任务：此时的任务已经把CPU资源100%消耗了，就没必要也不可能使用多线程来提高计算效率了；如果要做人机交互，最好还是要用多线程，避免用户没法对计算机进行操作。
- 单核CPU——IO密集型任务，使用多线程还是为了人机交互方便。
- 多核CPU——IO密集型任务，这就更不用说了，跟单核时候原因一样。（防止单线程由于IO阻塞）



参考资料：

- [进程和线程、并行和并发、单核和多核CPU](https://www.jianshu.com/p/8132ebc8e99d)
- [Windows下查看电脑的CPU个数，核心数，线程数](https://blog.csdn.net/ksws0292756/article/details/79119961)