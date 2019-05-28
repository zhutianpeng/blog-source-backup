---
title: java并发
date: 2019-03-27  16:58:47
tags:
	- java
    - 并发
---

java多线程实例

# 1. 守护线程
## 定义：
通过setDaemon(true)来设置线程为“守护线程”,守护进程（Daemon）是运行在后台的一种特殊进程。它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。也就是说守护线程不依赖于终端，但是依赖于系统，与系统“同生共死”。

- 当JVM中所有的线程都是守护线程的时候，JVM就可以退出了；
- 如果还有一个或以上的非守护线程则JVM不会退出。

## 实例：
```java
public class DaemonThread extends Thread{
    @Override
    public void run() {            //永真循环线程
        for(int i=0;;i++){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ex) {   }
            System.out.println(i);
        }
    }

    public static void main(String [] args){
        DaemonThread test = new DaemonThread();
        test.setDaemon(true);    
        //调试时可以设置为false，那么这个程序是个死循环，没有退出条件。设置为true，即可主线程结束，test线程也结束。
        test.start();
        System.out.println("isDaemon = " + test.isDaemon());
        try {
            System.in.read();   // 接受输入，使程序在此停顿，一旦接收到用户输入，main线程结束，守护线程自动结束
        } catch (IOException ex) {
        }
    }
}
```
## 补充
垃圾回收线程就是一个经典的守护线程，当我们的程序中不再有任何运行的Thread,程序就不会再产生垃圾，垃圾回收器也就无事可做，所以当垃圾回收线程是JVM上仅剩的线程时，垃圾回收线程会自动离开。它始终在低级别的状态中运行，用于实时监控和管理系统中的可回收资源。

# 2. Executors
## 1. 定义：
java.util.concurrent包中的ExecutorService, ExecutorService就是Java中对线程池的实现
```java 
1. newCachedThreadPool 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
2. newFixedThreadPool 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
3. newScheduledThreadPool 创建一个定长线程池，支持定时及周期性任务执行。
4. newSingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
``` 

## 2.用法实例：
基本使用和创建
```java 
ExecutorService exec = Executors.newFixedThreadPool(10);
exec.execute(new Runnable() {
    @Override
    public void run() {
        System.out.println("Asynchronous task");
    }
});
exec.shutdown();
```
```java
Future future = exec.submit(new Runnable() {
    @Override
    public void run() {
        System.out.println("Asynchronous task");
    }
});
    //如果任务结束执行则返回 null
System.out.println("future.get()=" + future.get());
```
```java
ExecutorService executorService = Executors.newSingleThreadExecutor();

Set<Callable<String>> callables = new HashSet<Callable<String>>();

callables.add(new Callable<String>() {
public String call() throws Exception {
    return "Task 1";
}
});
callables.add(new Callable<String>() {
public String call() throws Exception {
    return "Task 2";
}
});
callables.add(new Callable<String>() {
    public String call() throws Exception {
    return "Task 3";
}
});

String result = executorService.invokeAny(callables);
System.out.println("result = " + result);
executorService.shutdown();
```

- invokeAny(…)方法接收的是一个Callable的集合，执行这个方法不会返回Future，但是会返回所有Callable任务中其中一个任务的执行结果。这个方法也无法保证返回的是哪个任务的执行结果.

- invokeAll(…)与 invokeAny(…)类似也是接收一个Callable集合，但是前者执行之后会返回一个Future的List，其中对应着每个Callable任务执行后的Future对象。

# 3. N个线程循环打印累加数字
## 题目：
```
1、补充如下程序通过N个线程顺序循环打印从0至100，如给定N=3则输出： 
thread0: 0
thread1: 1
thread2: 2
thread0: 3
thread1: 4
...
注意线程号与输出顺序间的关系。
```
## 解法：
使用 锁
```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class NumberCount {
    public static void main(String[] args) {
        new NumberCount().printNumber(5);
    }

    public void printNumber(int n){

        PrintThead first = new PrintThead(0, n,100);
        first.start();

        PrintThead pre = first;
        PrintThead printThead = null;

        for(int i = 1; i<n; i++) {
            printThead = new PrintThead(i, n,100);
            pre.setNextThreadCondition(printThead.getCondition());
            pre = printThead;
            printThead.start();
        }
        if(pre!=first) {
            pre.setNextThreadCondition(first.getCondition());
        }
    }
}


class PrintThead extends Thread{
    int num;
    int max;
    int order;
    static ReentrantLock lock = new ReentrantLock();

    // 关键点1： Condition 的强大之处在于它可以为多个线程间建立不同的 Condition
    // 而：      notify 会提醒其他的任意 一个线程
    Condition condition = lock.newCondition();    
    static int count = 0;
    private Condition nextThreadCondition;

    PrintThead(int order, int num, int max){
        this.order = order;
        this.num = num;
        this.max = max;
    }

    public void setNextThreadCondition(Condition nextThreadCondition){
        this.nextThreadCondition = nextThreadCondition;
    }

    public Condition getCondition() {
        return condition;
    }

    @Override
    public void run() {
        while (true) {
            lock.lock();
            if (count > max) {
                nextThreadCondition.signal();     
                lock.unlock();                    //关键点2：最后一次，销毁线程前，需要手动释放锁
                break;
            }
            if (count % num == order) {
                System.out.println(Thread.currentThread().getName() + ":" + count);
                count++;
                nextThreadCondition.signal();    // signal 可以提醒一个指定的condition 
                try {
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else{
                try{
                    condition.await();
                }catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            lock.unlock();
        }
    }
}
``` 

## 考点： 线程之间的协作
1. join

    在线程中调用另一个线程的 join() 方法，会将当前线程挂起，直到目标线程结束后恢复。也可以带一个超时参数，保证目标线程在超时后，join方法可以返回

2. wait(), notify(), notifyAll()
```java 
public class TestThread implements Runnable {
    int i = 1;

    @Override
    public void run() {
        while(true){
        /*指代的为 t,因为使用的是implements方式。若使用继承Thread类的方式,使用this*/
            synchronized (this) {
            /*唤醒另外一个线程，注意是 this对象的方法，而不是 Thread 类的方法 */
            notify(); 
            try {
            /*使其休眠100毫秒，放大线程差异*/
                Thread.currentThread().sleep(100);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
                if (i<=100) {

                    System.out.println(Thread.currentThread().getName() + ":"+ i);
                    i++;
                    try {
                    /*放弃资源，等待*/
                        wait();
                    } catch (InterruptedException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
        }
    }
    }

    public static void main(String[] args) {
        /*只有一个TestThread对象*/
        TestThread t = new TestThread();
        Thread t1 = new Thread(t);
        Thread t2 = new Thread(t);

        t1.setName("线程1");
        t2.setName("线程2");

        t1.start();
        t2.start();

    }
}
```

3. await(), signal(), signalAll()

    java.util.concurrent 类库中提供了 Condition 类来实现线程之间的协调，可以在 Condition 上调用 await() 方法使线程等待，其它线程调用 signal() 或 signalAll() 方法唤醒等待的线程。相比于 wait() 这种等待方式，await() 可以指定等待的条件，因此更加灵活。

# 4. 并发编程 原理（给volatile做铺垫）
## 1. 内存模型的相关概念（计算机的内存模型）
计算机执行程序的时候，cpu（高速缓存）和 内存 之间存在 缓存数据不一致的问题，有两种解决方法：

- 通过在总线加LOCK#锁的方式；
- 通过缓存一致性协议

通过在总线加LOCK#锁的方式： 加锁会影响性能

=>

通过缓存一致性协议： 当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

## 2.并发编程的三个概念：

**要想并发程序正确地执行，必须要保证原子性、可见性以及有序性。只要有一个没有被保证，就有可能会导致程序运行不正确。** 

1. 原子性：  

即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。一个很经典的例子就是银行账户转账问题：

2. 可见性：(重要，需要理解)  

可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

```java
//线程1执行的代码
int i = 0;
i = 10;
 
//线程2执行的代码
j = i;
```

- 假若执行线程1的是CPU1，执行线程2的是CPU2。由上面的分析可知，当线程1执行 i =10这句时，会先把i的初始值加载到CPU1的高速缓存中，然后赋值为10，那么在CPU1的高速缓存当中i的值变为10了，却没有立即写入到主存当中。

    此时线程2执行 j = i，它会先去主存读取i的值并加载到CPU2的缓存当中，注意此时内存当中i的值还是0，那么就会使得j的值为0，而不是10.

3. 有序性 （涉及到jvm的指令重排）
- 有序性：即程序执行的顺序按照代码的先后顺序执行。
- 指令重排序：一般来说，处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。（ztp:有依赖的地方会保证是有序的，当然了，仅限单线程）

- 指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。

### 3. Java内存模型
引申：Java内存模型规定所有的变量都是存在主存当中（类似于前面说的物理内存），每个线程都有自己的工作内存（类似于前面的高速缓存）。线程对变量的所有操作都必须在工作内存中进行，而不能直接对主存进行操作。并且每个线程不能访问其他线程的工作内存。

1. 原子性：

    Java内存模型只保证了基本读取和赋值是原子性操作，如果要实现更大范围操作的原子性，可以通过synchronized和Lock来实现。synchronized和Lock能够保证任一时刻只有一个线程执行该代码块。

2. 可见性：

    Java提供了volatile关键字来保证可见性。

    解释：　当一个共享变量被volatile修饰时，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值。

    而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。

    另外，通过synchronized和Lock也能够保证可见性，synchronized和Lock能保证同一时刻只有一个线程获取锁然后执行同步代码，并且在释放锁之前会将对变量的修改刷新到主存当中。因此可以保证可见性

3. 有序性：

    happens-before原则（先行发生原则）：这8条原则摘自《深入理解Java虚拟机》。

- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
- 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作
- volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、 Thread.isAlive()的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始
 
 重要需要理解的是 volatile变量规则： 直观地解释就是，如果一个线程先去写一个变量，然后一个线程去进行读取，那么写入操作肯定会先行发生于读操作。

# 5. 线程安全
## 1. 不可变
变量不可变，则不需要考虑线程安全问题：

- final修饰
- String
- 枚举类型
- Number部分子类：Long，Double，BigInteger, BigDecimal 等

## 2. 阻塞同步（互斥同步） （悲观锁）
- synchronized
- ReentranLock （重入锁）

## 3. 非阻塞同步 （乐观锁）
先进行操作，判断没有其他线程共享资源，则操作成功；否则采取补偿措施（重试直到成功），此之谓非阻塞。

乐观锁需要 硬件支持原子性操作：CAS(compare and set)

## 4. 无同步方案
- 栈封闭（多线程访问方法的局部变量，属于线程私有的，没有安全性问题）
- 线程本地存储（将需要共享数据的 多块代码，放进一个线程里面，一次性搞定，例如“web的request就是一个thread”）
- 可重入代码？

# 6. 关键词分析：
## 1. synchronized
## 2. volatile （理解 见java内存模型）
原理：
- 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。

- 禁止进行指令重排序。

实现：  

“观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令”   

—- 《深入理解Java虚拟机》   

lock前缀相当于 内存屏障 会提供 3 个功能：    

- 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

- 它会强制将对缓存的修改操作立即写入主存；

- 如果是写操作，它会导致其他CPU中对应的缓存行无效。

面试答题思路：
```java 
volatile保证了 可见性； 禁止了指令重排；
    解释可见性（主内存 和 工作内存）
    解释有序性（jvm指令重排）

实现方式是通过加入lock前缀指令（内存屏障）
    解释内存屏障（禁止重拍，先写后读，缓存失效）
```
举例：
```java
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}
 
//线程2
stop = true;
```

该方案不能很好的利用线程2终止线程1的工作，需要加上 volatile关键字给stop变量,解释如下：

- 使用volatile关键字会强制将修改的值立即写入主存；

- 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）；

- 由于线程1的工作内存中缓存变量stop的缓存行无效，所以线程1再次读取变量stop的值时会去主存读取。

引申： （ztp的理解）  

“当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）”  

解决mysql 和 redis的缓存一致性的问题，可以从这一点学到：  

- 增：先写mysql，redis缓存失效

- 删：删除mysql,redis缓存失效

- 改：先改mysql，redis缓存失效

- 查：先查redis，有就返回；redis没有，则读库，然后更新redis缓存


## 3. ReentranLock