---
title: java泛型
date: 2019-11-06 14:59:01
tags:
- java
---


> 泛型的本质是 **参数化类型** 。主要作用是解决容器（类，数组）的继承关系。

# 1. 定义

- 类型参数：将类型作为可变的参数来表示。

- 有界类型参数：限制类型参数的类型种类范围。
    ```java
    T extends Number
    T extends Comparable<T>
    ```

- 类型通配符： 使用?代替具体的类型参数, 表示等所有<具体类型实参>的父类。

- 有界类型通配符
    ```java
    ? 
    ? extends Number
    ```

补充：
```
Difference between <？ extends A> and <T extends A> :

The reason for declaring a T is so that you can refer to it again, which can binding two parameter types, or a return type together.
```
# 2. 泛型方法


## 1. 用法
- 方法申明关键字： 所有泛型方法声明都有一个类型参数声明部分（由尖括号分隔），该类型参数声明部分在方法返回类型之前（在下面例子中的<E>）。

- 入参类型： 每一个类型参数声明部分包含一个或多个类型参数，参数间用逗号隔开。一个泛型参数，也被称为一个类型变量，是用于指定一个泛型类型名称的标识符。

- 返回值类型： 类型参数能被用来声明返回值类型，并且能作为泛型方法得到的实际参数类型的占位符。

- 限制：  泛型方法体的声明和其他方法一样。注意类型参数只能代表引用型类型，不能是基本数据类型。

## 2. 示例
```java
public class GenericMethodTest
{
    public static <E> E printArray(E[] inputArray)
    {
        for ( E element : inputArray ){
            System.out.printf( "%s ", element );
        }
        return inputArray[0];
    }

    public static void main( String args[] )
    {
        // 创建不同类型数组： Integer, Double 和 Character
        Integer[] intArray = { 1, 2, 3, 4, 5 };
        Double[] doubleArray = { 1.1, 2.2, 3.3, 4.4 };
        Character[] charArray = { 'H', 'E', 'L', 'L', 'O' };

        System.out.println( "整型数组元素为:" );
        printArray(intArray); // 传递一个整型数组

        System.out.println( "\n双精度型数组元素为:" );
        printArray(doubleArray); // 传递一个双精度型数组

        System.out.println( "\n字符型数组元素为:" );
        printArray(charArray); // 传递一个字符型数组
    }
}
```

# 3. 泛型类
## 1. 用法
- 和泛型方法一样，泛型类的类型参数声明部分也包含一个或多个类型参数，参数间用逗号隔开。
- 一个泛型参数，也被称为一个类型变量，是用于指定一个泛型类型名称的标识符。

## 2. 实例
```java
public class Box<T> {
   
  private T t;
 
  public void add(T t) {
    this.t = t;
  }
 
  public T get() {
    return t;
  }
 
  public static void main(String[] args) {
    Box<Integer> integerBox = new Box<Integer>();
    Box<String> stringBox = new Box<String>();
 
    integerBox.add(new Integer(10));
    stringBox.add(new String("ztxpp"));
 
    System.out.printf("整型值为 :%d\n\n", integerBox.get());
    System.out.printf("字符串为 :%s\n", stringBox.get());
  }
}
```

# 4. 类型通配符的PECS原则
## 1. 定义
- PESC: Producer Extends Consumer Super
```java
<? extends T> : 上界通配符
<? super T>: 下界通配符
```
## 2. 举例说明
![example1](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191106232816.png)

构造如上的类，继承关系如图所示。

### 1. 有界类型通配符：
下面两种表达对应下图所示：
```java
Plate<? extends Fruit>
Plate<? super Fruit>
```

![example2](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191106233502.png)

![example2](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20191106233655.png)

### 2. PECS原则：

```java
// situation1：
Plate<? extends Fruit> p1=new Plate<Apple>(new Apple()); 
p1.set(new Fruit()); //ERROR
p1.set(new Apple()); //ERROR

Fruit fruit = p1.get();  // OK
Food food = p1.get();  // OK
Apple apple = p1.get();  // ERROR

//situation2:
Plate<? super Fruit> p2 = new Plate<Fruit>(new Fruit());  
p2.set(new Fruit()) // OK
p2.set(new Apple()) // OK

Apple apple = p2.get();  //ERROR
Fruit fruit = p.get();    //ERROR
Object object = p2.get(); // OK
```

解释：
- situation1：
    - set难：编译器只知道？是Fruit或其子类，但不知道具体是哪一个，万一Plate里面定义的是装 low level的水果，但是用户希望set的是 high level的水果，则不允许。
    - get易：根据多态，只要申明的类型>=Fruit, 就都可以承载p1.get()出来的东西。
- situation2：
    - set： 根据多态，plate定义的是可以装下>=Fruit的东西，所以Fruit及其子类都可以set进入。
    - get: 由于编译器不知道>=Fruit的东西到底是大到什么类型，所以get出来的东西只能用 Object来承载。

PESC: 

频繁往外读取内容的，适合用上界Extends; 经常往里插入的，适合用下界Super。