---
title: java OOM异常
date: 2019-03-23 16:58:47
tags:
	- java
    - OOM
---

java OOM 异常处理 （OutOfMemoryError）

# 1. Java堆溢出
## 1.1 定义：
Java堆用于存储对象实例，只要不断地创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，对象数量达到最大堆容量限制，则发生溢出。(有点类似于“内存泄露”)

## 1.2 举例：
Java堆溢出
```java
public static void main(String[] args) {
    List<OOMObject> list = new ArrayList<OOMObject>();
    while(true) {
        // list保留引用，避免Full GC 回收 
        list.add(new OOMObject());
    }
}

static class OOMObject {}
```

报错信息：
```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:261)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
	at java.util.ArrayList.add(ArrayList.java:458)
	at OOM.HeapOOM.main(HeapOOM.java:14)
```

## 1.3 解决方式：
需要确认是 **内存泄露** 还是 **内存溢出**。

- 内存泄露 ：查看泄露对象到GC Roots的引用链，定位泄露代码位置。
- 内存溢出 ：如果不存在泄露，即内存中的对象确实都还必须活着，检查JVM堆参数（-Xmx与-Xms），调大参数，检查代码是否存在某些对象生命周期过长，持有状态过长的情况，减少程序运行期的内存消耗。

# 2. 虚拟机栈和本地方法栈溢出
## 2.1 定义：
HotSpot不区分虚拟机栈和本地方法栈，栈容量只能由-Xss参数设定。

- StackOverFlow：线程申请的栈深度超过允许的最大深度
- OutOfMemoryError： 虚拟机扩展时无法申请到足够的内存空间

## 2.2 举例
Java栈 stackoutflow: (递归调用)
```java
public class JavaVMStackStackOverFlow {

/**
 * 虚拟机栈和本地方法栈溢出 StackOverFlow
 * VM Args：-Xss128k
 */
    private int stackLength = 1;

    // 递归调用方法，定义大量的本地变量，增大此方法帧中本地变量表的长度
    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackStackOverFlow sof = new JavaVMStackStackOverFlow();
        try {
            sof.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length: " + sof.stackLength);
            throw e;
        }
    }
```

报错信息：
```java 
Exception in thread "main" java.lang.StackOverflowError
stack length: 19471
    at OOM.JavaVMStackStackOverFlow.stackLeak(JavaVMStackStackOverFlow.java:17)
    at OOM.JavaVMStackStackOverFlow.stackLeak(JavaVMStackStackOverFlow.java:17)
    ...
```

java栈 OutOfMemoryError：多线程下的内存溢出，与栈空间是否足够大并不存在任何联系。为每个线程的栈分配的内存越大（参数-Xss），那么可以建立的线程数量就越少，建立线程时就越容易把剩下的内存耗尽，越容易内存溢出。在这种情况下，如果不能减少线程数目或者更换64位虚拟机时，减少最大堆和减少栈容量能够换区更多的线程。

```java 
//注意：Windows平台虚拟机中，Java的线程映射到操作系统的内核线程上执行，下面代码执行可能造成系统假死！
//不要执行，不要执行，不要执行！
public class JavaVMStackOOM {
    private void dontStop() {
        while(true) {
        }
    }

    // 多线程方式造成栈内存溢出 OutOfMemoryError
    public void stackLeakByThread() {
        while(true) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}
```

执行结果：
```
windows死机
```

# 3. 运行时常量池溢出
## 3.1 定义
JDK1.7开始逐步“去永久代”，下面的讨论可以测试下实际影响。

String.intern()是一个Native方法，它的作用是：如果运行时常量池中已经包含一个等于此String对象内容的字符串，则返回常量池中该字符串的引用；如果没有，则在常量池中创建与此String内容相同的字符串，并返回常量池中创建的字符串的引用。JDK7的intern()方法的实现有所不同，当常量池中没有该字符串时，不再是在常量池中创建与此String内容相同的字符串，而改为在常量池中记录堆中首次出现的该字符串的引用，并返回该引用。

在JDK1.6之前，常量池分配在永久代，以下代码在JDK1.6下运行才回发生内存溢出，如果在JDK1.7及其之后的版本运行，则是死循环

## 3.2 举例
```java
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        int i = 0;
        while(true) {
            // list保留引用，避免Full GC 回收
            list.add(String.valueOf(i++).intern());
        }
    }
}
```
报错信息：
```
jdk1.7后 程序陷入死循环
```

# 4.方法区溢出
## 4.1 定义：
方法区用于存放Class的相关信息，如果运行时产生大量的类去填满方法区，就可能发生方法区的内存溢出。 例如主流框架Spring、Hibernate对大量的类进行增强时，利用CGLib字节码生成动态类；大量JSP或动态JSP(JSP第一次运行时需要编译为Java类）。

## 4.2 举例：
```java
/**
     * 方法区溢出
     * -XX:PermSize=10M -XX:MaxPermSize=10M
     */
    public static void main(String[] args) {
        while(true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(HeapOOM.OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object obj, Method m, Object[] objs, MethodProxy proxy) throws Throwable {
                    // TODO Auto-generated method stub
                    return proxy.invokeSuper(obj, objs);
                }
            });
            enhancer.create();
        }
    }
```

# 5. 本地直接内存溢出
## 5.1 定义：
Java虚拟机可以通过参数-XX:MaxDirectMemorySize设定本机直接内存可用大小，如果不指定，则默认与java堆内存大小相同。JDK中可以通过反射获取Unsafe类(Unsafe的getUnsafe()方法只有启动类加载器Bootstrap才能返回实例)直接操作本机直接内存。通过使用-XX:MaxDirectMemorySize=10M，限制最大可使用的本机直接内存大小为10MB。

## 5.2 举例：
```java

public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024 * 1024;
    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe . class .getDeclaredFields()[0];     
        unsafeField.setAccessible( true );
        Unsafe unsafe = ( Unsafe ) unsafeField.get( null );

        while ( true ) {
            // unsafe 直接想操作系统申请内存
            unsafe.allocateMemory( _1MB );
        }
    }
}
```
报错信息：
```
Exception in thread "main" java.lang.OutOfMemoryError
	at sun.misc.Unsafe.allocateMemory(Native Method)
	at OOM.DirectMemoryOOM.main(DirectMemoryOOM.java:20)
```

参考：https://blog.csdn.net/u011080472/article/details/51322119
