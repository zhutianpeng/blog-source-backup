---
title: 多线程与高并发（五）JUC同步工具(1)
date: 2020-05-06 17:37:56
tags: Java
---

> 多线程与高并发 的系列学习笔记，不仅仅停留在理论层面，尽量落到实地，以代码的形式实践、总结。


（五）JUC同步工具（1）

对于JUC同步工具的学习思路，首先从用法开始了解，之后再探讨原理。本文主要关注各同步工具的用法，后续文章逐渐深入原理。本文讨论的JUC同步工具包括：
- `ReentrantLock`
- `CountDownLatch`
- `CyclicBarrier`
- Phaser
- ReadWriteLock
- Semaphore
- Exchanger

# `ReentrantLock` - 可重入锁
## 用法
- 基本用法：能完成与Synchronized的可重入锁的功能，但是需要手动释放锁:`unlock()`。（Synchronized锁定会由JVM管理，出现异常情况会自动释放锁，ReentrantLock需要手动释放锁）
- 特性：
    1. `lock.tryLock()`：为了防止在长时间获取不到锁的情况下，出现阻塞的情况，可以使用`tryLock()`进行尝试锁定。无论是否成功，方法将继续执行下去。
    2. `lock.lockInterruptibly()`:可以被打断的锁。后序可以通过`t.interrupt();`将正在运行的`线程t`，进行打断。 （就是叫`线程t`别跑了，也别等了。）
    3. 可以设定`公平锁和非公平锁`。公平锁（true）对于等待的线程讲究先来后到，满足FIFO；非公平锁(false)就看哪个线程能争抢上，没有先后顺序。ReentrantLock默认是非公平锁。
        - 公平锁每次获取到锁为同步队列中的第一个节点，保证请求资源时间上的绝对顺序，而非公平锁有可能刚释放锁的线程下次继续获取该锁，则有可能导致其他线程永远无法获取到锁，造成“饥饿”现象。
        - 公平锁为了保证时间上的绝对顺序，需要频繁的上下文切换，而非公平锁会降低一定的上下文切换，降低性能开销。因此，ReentrantLock默认选择的是非公平锁，则是为了减少一部分上下文切换，保证了系统更大的吞吐量。


## 用例
- 基本用法：
    ```java
    public class T1_ReentrantLock1 {
        Lock lock = new ReentrantLock();
    //  基础用法：
        void m1(){
            try {
                lock.lock();  //手动上锁
                for (int i = 0; i < 10; i++) {
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(i);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                //手动解锁，一定要都放在try...finally块中，保证异常情况下也能解锁
                lock.unlock();
            }
        }
        void m2(){
            try {
                lock.lock();
                System.out.println("m2 ...");
            } finally {
                lock.unlock();
            }
        }

        public static void main(String[] args) {
            T1_ReentrantLock1 t = new T1_ReentrantLock1();
            new Thread(t::m1).start();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            new Thread(t::m2).start();
        }
        //输出结果为m1方法全部执行完之后，m2方法才开始执行，表示锁生效了。
    }
    ```
- `lock.tryLock()`:
    ```java
    public class T1_ReentrantLock2 {
        Lock lock = new ReentrantLock();
        void m1(){
            try {
                lock.lock();
                //此处m1执行时间的长短，表示占有锁的时间长短
                for (int i = 0; i < 10; i++) {
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(i);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
        void m2(){
            boolean locked = false;
            try {
                locked = lock.tryLock(5,TimeUnit.SECONDS);
                if (locked){
                    System.out.println("m2: get lock");
                    // do something...
                }else{
                    System.out.println("m2: don't get lock");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if(locked) lock.unlock();
            }
        }

        public static void main(String[] args) {
            T1_ReentrantLock2 t = new T1_ReentrantLock2();
            new Thread(t::m1,"t1").start();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            new Thread(t::m2,"t2").start();
        }
    }
    ```

    当t1执行时间较短（在t2的等待时间之内），t2线程拿到了锁，可以do something...
    
    当t1执行时间较长（超过了t2的等待时间），t2线程即使拿不到锁，也不会一直阻塞下去，可以执行拿不到锁的else代码块。

    由此可见代码的灵活性更高，可以自行决定是否wait下去，如果是synchronized拿不到锁，就会一直阻塞下去。

- `lock.lockInterruptibly()`
    ```java
    public class T1_ReentrantLock3 {
        Lock lock = new ReentrantLock();
        void m1(){
            try {
                lock.lock();
                System.out.println("t1 start");
                TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
                System.out.println("t1 end");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
            }
        }
        void m2(){
            try {
                lock.lockInterruptibly(); //允许被打断的锁
                System.out.println("t2 start");
                TimeUnit.SECONDS.sleep(5);
                System.out.println("t2 end");
            } catch (InterruptedException e) {
                e.printStackTrace();
                System.out.println("interrupyed!");
            } finally {
                lock.unlock();
            }
        }

        public static void main(String[] args) {
            T1_ReentrantLock3 t = new T1_ReentrantLock3();
            Thread t1 = new Thread(t::m1);
            Thread t2 = new Thread(t::m2);
            t1.start();
            t2.start();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            t2.interrupt(); //这里可以手动终止t2的线程运行。
        }
    }
    ```

- 设置公平与非公平锁
    ```java
    public class T1_ReentrantLock4 extends Thread{
        private static ReentrantLock lock = new ReentrantLock(false);
        public void run(){
            for (int i = 0; i < 100; i++) {
                try {
                    lock.lock();
                    System.out.println(Thread.currentThread().getName() + "获得锁"+" 循环次数： "+i);
                    Thread.sleep(50);
                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }

        public static void main(String[] args) {
            T1_ReentrantLock4 t = new T1_ReentrantLock4();
            Thread t1 = new Thread(t,"t1");
            Thread t2 = new Thread(t,"t2");
            t1.start();
            t2.start();
        }
    }
    ```

    上述例子中，公平锁与非公平锁的设置，执行结果不一样：非公平锁情况下,t1,t2会相互争抢着进行；公平锁情况下，t1,t2会交替进行。

# `CountDownLatch` - 倒数门栓
## 用法：
用`CountDownLatch`可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。

CountDownLatch类只提供了一个构造器：
```java
public CountDownLatch(int count) {};  //参数count为计数值
```
然后下面这3个方法是CountDownLatch类中最重要的方法：
```java
//调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public void await() throws InterruptedException { };   

 //和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { }

//将count值减1
public void countDown() { };  
```

## 用例：
有100个线程Ai,以及一个线程B。要使线程B在线程Ai全部执行完之后再执行，可以用`CountDownLatch`:
```java
public class T2_CountDownLatch {
    public static void main(String[] args) {
        usingCountDownLatch();
    }
    private static void usingCountDownLatch(){
        Thread[] threadList = new Thread[100];
        CountDownLatch latch = new CountDownLatch(threadList.length);
        for (int i = 0; i < threadList.length; i++) {
            threadList[i] = new Thread(()->{
               int result = 0;
                for (int j = 0; j < 10000; j++) {
                    result+=j;
                }
                System.out.println(Thread.currentThread().getName()+" "+ result);
                latch.countDown();
            },"A"+i);
        }
        for (int i = 0; i < threadList.length; i++) {
            threadList[i].start();
        }
        try {
            System.out.println("等待子线程运行中...");
            latch.await();
            System.out.println("所有子线程运行完毕");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()-> System.out.println(Thread.currentThread().getName()+" run"),"B").start();
    }
}
```

# `CyclicBarrier` - 循环栅栏
## 用法
- 构造器：
    ```java
    // 参数parties指让多少个线程或者任务等待至barrier状态；
    // 参数barrierAction为当这些线程都达到barrier状态时会执行的内容。
    public CyclicBarrier(int parties, Runnable barrierAction) {}
    public CyclicBarrier(int parties) {}
    ```

- 主要方法：
    ```java
    // 当前线程挂起，直到有parties个数的线程到达barrier状态，即可继续执行
    public int await() throws InterruptedException, BrokenBarrierException { }  ;

    //加上了等待时间，防止阻塞
    public int await(long timeout, TimeUnit unit)throws InterruptedException,   BrokenBarrierException,TimeoutException { };
    ```

- 特点：与`CountDownLatch`相比，二者本质上都是对线程的一种控制器（门栓、栅栏），判断是否执行完毕。但是`CyclicBarrier`可以重复使用，所以叫“循环”

## 用例

例1：（重复使用特点）

 本例是102个线程，每满20个线程，就放行；如果不满20个，满足等待时间，也放行。
```java
public class T3_CyclicBarrier {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(20,()-> System.out.println("人满发车"));
        for (int i = 0; i < 102; i++) {
            new Thread(()->{
                try {
                    cyclicBarrier.await(5, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }catch (TimeoutException e) {
                    System.out.println("等待时间到，发车");
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

<details>
<summary>输出</summary>
<pre>
人满发车
人满发车
人满发车
人满发车
人满发车
等待时间到，发车
java.util.concurrent.BrokenBarrierException
	at java.util.concurrent.CyclicBarrier.dowait(CyclicBarrier.java:250)
	at java.util.concurrent.CyclicBarrier.await(CyclicBarrier.java:435)
	at c5.T3_CyclicBarrier.lambda$main$1(T3_CyclicBarrier.java:14)
	at java.lang.Thread.run(Thread.java:748)
java.util.concurrent.TimeoutException
	at java.util.concurrent.CyclicBarrier.dowait(CyclicBarrier.java:257)
	at java.util.concurrent.CyclicBarrier.await(CyclicBarrier.java:435)
	at c5.T3_CyclicBarrier.lambda$main$1(T3_CyclicBarrier.java:14)
	at java.lang.Thread.run(Thread.java:748)
</pre>
</details>

例2: （具体业务场景）

本例是要进行一个复杂的业务逻辑`nextBusinessLogic`前，必须先获取来自不同数据源的数据（这是耗时操作），如果顺序执行：`读数据库数据-> 读网络数据-> 读文件数据`，则需要较长时间，可以使用并行操作。那么可以使用`CyclicBarrier`来保证：这三个线程执行完毕，才能进行后续操作。

```java
public class T3_CyclicBarrier2 {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3,()->{
            nextBusinessLogic();
        });
        List<Thread> threads = new ArrayList<Thread>();
        businessLogic(threads,"读数据库数据",2000,cyclicBarrier);
        businessLogic(threads,"读网络数据",4000,cyclicBarrier);
        businessLogic(threads,"读文件数据",3000,cyclicBarrier);
        for (int i = 0; i < 3; i++) {
            threads.get(i).start();
        }
    }

    public static void businessLogic( List<Thread> threadList, String threadName, int waitTime, CyclicBarrier barrier){
        threadList.add(new Thread(()->{
            System.out.println("线程"+Thread.currentThread().getName()+"正在执行...");
            try {
                Thread.sleep(waitTime);
                System.out.println("线程"+Thread.currentThread().getName()+"获取数据完毕，等待其他线程获取完毕");
                barrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        },threadName));
    }

    public static void nextBusinessLogic(){
        System.out.println("数据库,网络,文件的数据都获取完毕，执行下面的业务...");
    }
}
```

<details>
<summary>输出</summary>
<pre>
线程读数据库数据正在执行...
线程读文件数据正在执行...
线程读网络数据正在执行...
线程读数据库数据获取数据完毕，等待其他线程获取完毕
线程读文件数据获取数据完毕，等待其他线程获取完毕
线程读网络数据获取数据完毕，等待其他线程获取完毕
数据库,网络,文件的数据都获取完毕，执行下面的业务...
</pre>
</details>

