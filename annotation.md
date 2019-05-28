---
title: Java 注解 （Annotation）
date: 2019-05-19 18:30:02
tags:
- java
---

> annotation: Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。Java 注解是从 Java5 开始添加到 Java 的。下文为关于 annotation的学习笔记。

# 1. annotation的地位  
- annotation是一种类型，跟Class, Interface一个层面上的定义。
- annotation在 rt.jar 中的 lang.annotation 中定义，其使用与 lang.reflect 紧密相关。

# 2. annotation的语法 
## 2.1 理解：
- Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。Java 注解是从 Java5 开始添加到 Java 的。  
- 注解就是一种标签，一种盖章。可以给事物打上这种标签，使得事物具有某种属性。

## 2.2 定义：
注解通过 @interface关键字进行定义。
```java 
public @interface Login {} // 理解：创建了一张名字为 Login 的标签。
```

## 2.3 元注解：
- 元注解是一种基本注解，能给注解加注解。
- 元标签有 @Retention、@Documented、@Target、@Inherited、@Repeatable 5 种。
    ```java
    //保留期:当 @Retention 应用到一个注解上的时候，它解释说明了这个注解的的存活时间。
    //分为：源码期，编译期，运行期：
    //      SOURCE(源码阶段保留，编译时丢弃)，
    //      CLASS(保留到编译期，不被加载到JVM)，
    //      RUNTIME（保留到程序运行，它会被加载进入到 JVM 中，在程序运行时可以获取到）
    @Retention() 

    // 文档:作用是能够将注解中的元素包含到 Javadoc 中去。
    @Documented

    //目标: @Target 指定了注解运用的场景。
    //分为： 注解，构造方法，属性，局部变量，方法，包，方法内参数，类：
    //      ANNOTATION_TYPE: 可以给注解进行注解
    //      CONSTRUCTOR ：给构造方法进行注解
    //      FIELD ：给属性进行注解
    //      LOCAL_VARIABLE： 给局部变量进行注解
    //      METHOD ：给方法进行注解
    //      PACKAGE ：给一个包进行注解
    //      PARAMETER ：给方法内的参数进行注解
    //      TYPE ：可以给一个类型进行注解，比如类、接口、枚举
    @Target()
    
    // 继承, 超类被一个 “被 @Inherited注解过的注解 @testA” 注解，则子类也拥有这个注解 @testA。 
    @Inherited()
        //eg：
            @Inherited
            @Retention(RetentionPolicy.RUNTIME)
            @interface Test {}

            @Test
            public class A {}

            public class B extends A {} // B也具有了 @Test 的注解

    // 可重复, 同一个注解可以用来加在同一个事物上多次
    @Repeatable()
        //eg：
            @interface Persons {
	            Person[]  value();
            }

            @Repeatable(Persons.class)
            @interface Person{
            	String role default "";
            }

            @Person(role="artist")
            @Person(role="coder")
            @Person(role="PM")
            public class SuperMan{}
    ```
## 2.4 注解的属性
- 在注解中定义属性时它的类型必须是 8 种基本数据类型（char,int,short,long,boolean,byte,double,float）外加String、 class、 enum、接口、注解及它们的数组
- 示例：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {

	public int id() default -1;

	public String msg() default "Hi";

}
```
```java
@TestAnnotation(id=3,msg="hello annotation")
public class Test {}
```

## 2.5 Java 预置的注解
- @Deprecated ：过时的元素
- @Override: override
- @SuppressWarnings:关闭不当的编译器警告。
- @SafeVarargs：参数安全类型注解。它的目的是提醒开发者不要用参数做一些不安全的操作。
- @FunctionalInterface: 函数式编程接口

## 2.6 注解的提取 - 反射
注解是元素的一个标签，想要获取注解，先获取元素。所以获取 class, method, field等就需要用到 reflect的方法。   
基本的使用方法：
```java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
public Annotation[] getAnnotations() {}
```
获取到annotation后，再获取相应的属性即可。

# 3. annotation的应用  
annotation的应用或者作用不能一概而论，分类讨论，注解的分类如下：  

## 3.1 按照运行机制划分：
1. 源码注解:    
    提供信息给编译器，编译器可以利用注解来探测错误和警告信息

2. 编译时注解：   
    软件工具可以用来利用注解信息来生成代码、Html文档或者做其它相应处理。   
    eg：@Override、@Deprecated、@SuppressWarnings

3. 运行时注解：    
    某些注解可以在程序运行的时候接受代码的提取  
    eg：Spring的 @Autowired自动注入

## 3.2 按照来源划分：

1. 来自JDK的注解  
@Retention、@Documented、@Target、@Inherited、@Repeatable

2. 来自第三方的注解  
举例：   
![来自第三方的注解](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E9%97%B2%E6%95%A3/%E6%B3%A8%E8%A7%A3.png)  

3. 自定义注解  

## 3.3 元注解：
注解的注解。


# 4. 自定义注解实战

## 4.1 利用自定义注解 标记 “登陆后方法”，并结合拦截器检查
定义@Login注解：   
Login
```java
/**
 * 登录效验
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Login {
}
```

结合Interceptor设置登录状态判断：  
AuthorizationInterceptor.java  
```java
/**
 * 权限(Token)验证
 */
@Component
public class AuthorizationInterceptor extends HandlerInterceptorAdapter {
    @Autowired
    private TokenService tokenService;

    public static final String USER_KEY = "userId";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //TODO: 判断该方法是否需要进行登录状态的校验。
        Login annotation;
        if(handler instanceof HandlerMethod) {
            annotation = ((HandlerMethod) handler).getMethodAnnotation(Login.class);
        }else{
            return true; //不需要登录状态校验，直接返回。
        }

        if(annotation == null){
            return true; //不需要登录状态校验，直接返回。
        }

        //下面的登录校验的业务逻辑不需要关注。
        //从header中获取token
        String token = request.getHeader("token");
        //如果header中不存在token，则从参数中获取token
        if(StringUtils.isBlank(token)){
            token = request.getParameter("token");
        }

        //token为空
        if(StringUtils.isBlank(token)){
            RRException exception = new RRException("token不能为空");
            exception.setCode(401);
            throw exception;
        }

        //查询token信息
        TokenEntity tokenEntity = tokenService.queryByToken(token);
        if(tokenEntity == null || tokenEntity.getExpireTime().getTime() < System.currentTimeMillis()){
            RRException exception = new RRException("token失效，请重新刷新token");
            exception.setCode(401);
            throw exception;
        }

        //设置userId到request里，后续根据userId，获取用户信息
        request.setAttribute(USER_KEY, tokenEntity.getUserId());

        return true;
    }
}
```

具体使用：  
EvaluationRealtimeController.java
```java
/**
 * @author ztp
 * @date 2018-5-29
 * 列出一段时间内的，某个用户的评估结果，list形式
 */
@Login //这句话表示下面的方法，需要进过登录状态判断，必须是登录后才可以访问到的方法
@PostMapping("/list/duringTime/{startTime}/{endTime}")
@ApiOperation("列出一段时间内的，某个用户的评估结果")
public R listDuringTime(@ApiParam(value = "YYYY-MM-DD HH:mm:SS", required = true)@PathVariable("startTime") String startTime,
                         @ApiParam(value = "YYYY-MM-DD HH:mm:SS", required = true)@PathVariable("endTime") String endTime,
                         @ApiIgnore @RequestAttribute("userId") Long userId){
    List<TrainRealtimeInstanceEntity> list =evaluationRealtimeService.queryDuringTimeAndUserId(userId,startTime,endTime);
    return R.ok().put("list", list);
}
```


## 4.2 利用自定义注解实现 ORM 映射
