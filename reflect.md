---
title: java反射
date: 2019-03-18 16:58:47
tags:
	- java
    - reflect
---

总结一下java中的反射的基础知识，以及反射的应用（Spring）

# 1.java的反射机制

Java 反射机制在程序运行时，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性。这种 动态的获取信息 以及 动态调用对象的方法 的功能称为 java 的反射机制。举例如下： 

父类：FatherClass
```java
public class FatherClass {
    public String mFatherName;
    public int mFatherAge;
    private int testa;

    public void printFatherMsg(){}
}
```

子类：SonClass
```java
public class SonClass extends FatherClass {

    private String mSonName;
    protected int mSonAge;
    public String mSonBirthday;

    public void printSonMsg(){
        System.out.println("Son Msg - name : "
                + mSonName + "; age : " + mSonAge);
    }

    private void setSonName(String name){
        mSonName = name;
    }

    private void setSonAge(int age){
        mSonAge = age;
    }

    private int getSonAge(){
        return mSonAge;
    }

    private String getSonName(){
        return mSonName;
    }
}
```

## 1. 使用反射获取类的信息
```java
/**
 * 通过反射获取类的所有变量
 */
private static void printFields() throws NoSuchFieldException, IllegalAccessException {
    //1.获取并输出类的名称
    Class mClass = SonClass.class;
    System.out.println("类的名称：" + mClass.getName());

    //2.1 获取所有 public 访问权限的变量
    // 包括本类声明的和从父类继承的
    Field[] fields = mClass.getFields();

    //2.2 获取所有本类声明的变量（不问访问权限）
    // Field[] fields = mClass.getDeclaredFields();

    //2.3 获取父类的private方变量
    // Field fi = mClass.getSuperclass().getDeclaredField("testa");
    // System.out.println("1:"+Modifier.toString(fi.getModifiers()) + " "+fi.getType().getName() + " " + fi.getName());


    //3. 遍历变量并输出变量信息
    for (Field field : fields) {
        //获取访问权限并输出
        int modifiers = field.getModifiers();
        System.out.print(Modifier.toString(modifiers) + " ");
        //输出变量的类型及变量名
        System.out.println(field.getType().getName() + " " + field.getName());
    }
}
```  
输出：2.1 getFields() 包括了 子类的public属性和继承自父类的所有属性（当然继承的public的） 
```
类的名称：fanshe.SonClass
public java.lang.String mSonBirthday
public java.lang.String mFatherName
public int mFatherAge
```
输出：2.2 getDeclaredFields()方法： 包括了 子类所有的属性，不问访问权限
```
类的名称：fanshe.SonClass
private java.lang.String mSonName
protected int mSonAge
public java.lang.String mSonBirthday
```
输出2.3 想获取父类的信息，先getSuperclass(), 然后再 getDeclaredField(), 即可拿到private 属性  
```
类的名称：fanshe.SonClass
1:private int testa
```

## 2. 获取类的所有方法信息
```java
private static void printMethods(){
        //1.获取并输出类的名称
        Class mClass = SonClass.class;
        System.out.println("类的名称：" + mClass.getName());

        //2.1 获取所有 public 访问权限的方法
        //包括自己声明和从父类继承的
       Method[] mMethods = mClass.getMethods();

        //2.2 获取所有本类的的方法（不问访问权限）
        // Method[] mMethods = mClass.getDeclaredMethods();

        //3.遍历所有方法
        for (Method method : mMethods) {
            //获取并输出方法的访问权限（Modifiers：修饰符）
            int modifiers = method.getModifiers();
            System.out.print(Modifier.toString(modifiers) + " ");
            //获取并输出方法的返回值类型
            Class returnType = method.getReturnType();
            System.out.print(returnType.getName() + " " + method.getName() + "( ");
            //获取并输出方法的所有参数
            Parameter[] parameters = method.getParameters();
            for (Parameter parameter: parameters) {
                System.out.print(parameter.getType().getName() + " " + parameter.getName() + ",");
            }
            //获取并输出方法抛出的异常
            Class[] exceptionTypes = method.getExceptionTypes();
            if (exceptionTypes.length == 0){
                System.out.println(" )");
            }
            else {
                for (Class c : exceptionTypes) {
                    System.out.println(" ) throws " + c.getName());
                }
            }
        }
    }
```  

输出: 2.1 getMethods() 拿到了本类的所有public方法，包括从父类继承来的:
```
类的名称：fanshe.SonClass
public void printSonMsg(  )         //自己的方法
public void printFatherMsg(  )      //从父类继承来的方法                                
public final void wait(  ) throws java.lang.InterruptedException // 下面的方法来自 Object 类。
public final void wait( long arg0,int arg1, ) throws java.lang.InterruptedException
public final native void wait( long arg0, ) throws java.lang.InterruptedException
public boolean equals( java.lang.Object arg0, )
public java.lang.String toString(  )
public native int hashCode(  )
public final native java.lang.Class getClass(  )
public final native void notify(  )
public final native void notifyAll(  )
```

输出： 2.2 getDeclaredMethods(); 拿到了这个类中定义的所有方法，不包括继承来的，但是包含private方法
```
类的名称：fanshe.SonClass
private void setSonName( java.lang.String arg0, )
private int getSonAge(  )
private void setSonAge( int arg0, )
public void printSonMsg(  )
private java.lang.String getSonName(  )
``` 

## 3. 访问或操作类的私有变量和方法
定义类：TestClass
```java
public class TestClass {

    private String MSG = "Original";                       //private变量

    private void privateMethod(String head , int tail){    //private方法
        System.out.print(head + tail);
    }

    public String getMsg(){
        return MSG;
    }
}
```

- 利用 getPrivateMethod() 获取私有方法:
```java
/**
     * 访问对象的私有方法
     * 为简洁代码，在方法上抛出总的异常，实际开发别这样
     */
    private static void getPrivateMethod() throws Exception{
        //1. 获取 Class 类实例
        TestClass testClass = new TestClass();
        Class mClass = testClass.getClass();

        //2. 获取私有方法
        //第一个参数为要获取的私有方法的名称
        //第二个为要获取方法的参数的类型，参数为 Class，没有参数就是null
        //方法参数也可这么写 ：new Class[]{String.class , int.class}
        Method privateMethod = mClass.getDeclaredMethod("privateMethod", String.class, int.class);

        //3. 开始操作方法
        if (privateMethod != null) {
            //获取私有方法的访问权
            //只是获取访问权，并不是修改实际权限
            privateMethod.setAccessible(true);

            //使用 invoke 反射调用私有方法
            //privateMethod 是获取到的私有方法
            //testClass 要操作的对象
            //后面两个参数传实参
            privateMethod.invoke(testClass, "Java Reflect ", 666);
        }
    }
```

输出： 调用私有方法成功
```
Java Reflect 666
```

- 利用modifyPrivateFiled() 修改私有变量：
```java

/**
 * 修改对象私有变量的值
 * 为简洁代码，在方法上抛出总的异常
 */
private static void modifyPrivateFiled() throws Exception {
    //1. 获取 Class 类实例
    TestClass testClass = new TestClass();
    Class mClass = testClass.getClass();

    //2. 获取私有变量
    Field privateField = mClass.getDeclaredField("MSG");

    //3. 操作私有变量
    if (privateField != null) {
        //获取私有变量的访问权
        privateField.setAccessible(true);

        //修改私有变量，并输出以测试
        System.out.println("Before Modify：MSG = " + testClass.getMsg());

        //调用 set(object , value) 修改变量的值
        //privateField 是获取到的私有变量
        privateField.set(testClass, "Modified"); //testClass 要操作的对象, "Modified" 为要修改成的值
        System.out.println("After Modify：MSG = " + testClass.getMsg());
    }
}
```

输出：

```
Before Modify：MSG = Original
After Modify：MSG = Modified
```
反射基础部分借鉴：https://juejin.im/post/598ea9116fb9a03c335a99a4

# 2. java反射在项目中的应用
## 2.1 工厂模式
例子：国家电网能力开放平台 项目 之中的 “计费” 使用的是 工厂模式。

接口类： （计费）   
```java
@Component("chargeService")
public interface ChargeServiceI {
    public void charge(JSONObject j);
    public boolean checkStatus(JSONObject j) throws Exception;
}
```

实现类1：ChargeByQueryServiceImpl（按照请求次数计费） 不用管具体实现，知道有两个方法为charge(), checkStatus() 即可

```java
@Component("chargeByQueryServiceImpl")
public class ChargeByQueryServiceImpl implements ChargeServiceI {
//    @Autowired
//    ChargeAccountServiceI chargeAccountService;
    @Autowired
    private SystemService systemService;
    @Autowired
    private AlarmService alarmService;
    @Autowired
    private SearchMethod searchMethod;

    @Override
    public void charge(JSONObject j) {
        ChargeAccountEntity c = new ChargeAccountEntity();
        c.setId(j.get("chargeAccountId").toString());
        c.setTypename(j.get("typename").toString());
        c.setRestState(String.valueOf(Integer.parseInt(j.get("restState").toString())-1));
        try {
            ChargeAccountServiceI chargeAccountService = (ChargeAccountServiceI) SpringContextUtils.getBean("chargeAccountService");
            chargeAccountService.saveOrUpdate(c);
        } catch (Exception e) {
            systemService.addLog("计费失败", Globals.Log_Type_UPDATE, Globals.Log_Leavel_ERROR,"1");
            e.printStackTrace();
        }
    }

    @Override
    public boolean checkStatus(JSONObject j) throws ControlException{
        if (Integer.parseInt(j.get("restState").toString())>0){
            return true;
        }else {
            //alarmService.writeDatabase("剩余次数不足！","1",apiId,appName);
            throw new ControlException("剩余次数不足，请充值！");
        }
    }
}
```

实现类2：ChargeByFlowServiceImpl（按照流量计费）
```java
public class ChargeByFlowServiceImpl implements ChargeServiceI {
    @Autowired
    ChargeAccountServiceI chargeAccountService;
    @Autowired
    private SystemService systemService;
    @Autowired
    private AlarmService alarmService;
    @Autowired
    private SearchMethod searchMethod;

    @Override
    public void charge(JSONObject j) throws ControlException {
        int restState = Integer.parseInt(j.get("restState").toString()) - Integer.parseInt(j.get("flowAmount").toString());
        ChargeAccountEntity c = new ChargeAccountEntity();
        if (restState < 0)
            throw new ControlException("流量超出限额");
        c.setId(j.get("chargeAccountId").toString());
        c.setTypename(j.get("typename").toString());
        c.setRestState(String.valueOf(restState));
        try {
            chargeAccountService.saveOrUpdate(c);
        } catch (Exception e) {
            systemService.addLog("计费失败", Globals.Log_Type_UPDATE, Globals.Log_Leavel_ERROR, "1");
            e.printStackTrace();
        }
    }

    @Override
    public boolean checkStatus(JSONObject j) throws ControlException {
        if (Integer.parseInt(j.get("restState").toString()) > 0) {
            return true;
        } else {
            throw new ControlException("剩余流量不足，请充值！");
        }
    }
```

实现类3：ChargeByPeriodServiceImpl（按照时间计费）
```java
public class ChargeByPeriodServiceImpl implements ChargeServiceI {
    @Autowired
    private AlarmService alarmService;
    @Autowired
    private SearchMethod searchMethod;

    @Override
    public void charge(JSONObject j) {
        
    }

    @Override
    public boolean checkStatus(JSONObject j) throws Exception {
        DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        Date expireDate = dateFormat.parse(j.get("restState").toString());
        Date today = new Date();
        if (today.before(expireDate) || DateUtils.isSameDay(expireDate, today)) {
            return true;
        } else {
            //alarmService.writeDatabase("订购已经到期！","2",apiId,appName);
            throw new Exception("订购已经到期，请续订！");
        }
    }
}
```   

工厂类：

```java
@Component("chargeFactory")
public class ChargeFactory {
    @Autowired
    ChargeAccountServiceI chargeAccountService;

    public boolean checkStatus(JSONObject param) throws ControlException {
        boolean status = false;
        Class l = null;
        try {
            l = Class.forName(param.get("typename").toString());
            Object obj1 = l.newInstance();
            Method m = l.getMethod("checkStatus", param.getClass());
            status = (boolean) m.invoke(obj1, param);
            return status;
        } catch (ControlException e) {
            throw new ControlException(e.msg);
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            throw new ControlException(e.getTargetException().getMessage());
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return status;
    }

    public void charge(JSONObject param) {

        Class l = null;
        try {
            l = Class.forName(param.get("typename").toString());
            Object classObj = l.newInstance();
            Method m = l.getMethod("charge", param.getClass());
            m.invoke(classObj,param);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (ControlException e) {
            throw new ControlException(e.msg);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }

    }
```

- 工厂模式示意图：  
![工厂模式结构](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/java-reflect1.png)

- 反射相关
```java
try {
    Class Son = Class.forName("fanshe.SonClass");
    Object son = Son.newInstance();
    Method method = Son.getMethod("printSonMsg");
    method.invoke(son);
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```  

## 2.2 Spring的IOC中的反射
只要在代码或配置文件中看到类的完整路径（包.类），其底层原理基本上使用的就是Java的反射机制。对于Spring，主要关注IOC底层实现的原理（反射），Bean容器的实现。

- 示例1： 基于 的 Spring 构造器的注入：
```xml
<bean id="courseDao" class="com.qcjy.learning.Dao.impl.CourseDaoImpl"></bean>
```

伪代码解释：
```xml
//解析<bean .../>元素的id属性得到该字符串值为“courseDao”
String idStr = "courseDao";
//解析<bean .../>元素的class属性得到该字符串值为“com.qcjy.learning.Dao.impl.CourseDaoImpl”
String classStr = "com.qcjy.learning.Dao.impl.CourseDaoImpl";
//利用反射知识，通过classStr获取Class类对象
Class<?> cls = Class.forName(classStr);
//实例化对象
Object obj = cls.newInstance();
//container表示Spring容器
container.put(idStr, obj);
```

- 示例2：含属性的bean 通过setter注入：

当一个类里面需要应用另一类的对象时，Spring的配置如下所示：
```xml
<bean id="courseService" class="com.qcjy.learning.service.impl.CourseServiceImpl">
     <!-- 控制调用setCourseDao()方法，将容器中的courseDao bean作为传入参数 -->
     <property name="courseDao" ref="courseDao"></property>
</bean>
```   

伪代码解释：
```xml
//解析<property .../>元素的name属性得到该字符串值为“courseDao”
String nameStr = "courseDao";
//解析<property .../>元素的ref属性得到该字符串值为“courseDao”
String refStr = "courseDao";
//生成将要调用setter方法名
String setterName = "set" + nameStr.substring(0, 1).toUpperCase()
		+ nameStr.substring(1);
//获取spring容器中名为refStr的Bean，该Bean将会作为传入参数
Object paramBean = container.get(refStr);
//获取setter方法的Method类，此处的cls是刚才反射代码得到的Class对象
Method setter = cls.getMethod(setterName, paramBean.getClass());
//调用invoke()方法，此处的obj是刚才反射代码得到的Object对象
setter.invoke(obj, paramBean);
```

Spring的IOC部分借鉴：https://blog.csdn.net/mlc1218559742/article/details/52774805

## 2.3 Spring的AOP中的反射(使用动态代理模式实现)

- 代理模式：静态代理和动态代理，静态代理，顾名思义，就是你自己写代理对象，动态代理，则是在运行期，生成一个代理对象。
- AOP：Spring AOP就是基于动态代理的，如果要代理的对象，实现了某个接口，那么Spring AOP会使用JDK Proxy，去创建代理对象。
对于没有实现接口的对象，就无法使用JDK Proxy去进行代理了，Spring AOP会使用Cglib，生成一个被代理对象的子类，来作为代理。

简单的示例：

Interface: IOperation
```java
public interface IOperation {
    /**
     * 方法执行之前的操作
     * @param method
     */
    void start(Method method);
    /**
     * 方法执行之后的操作
     * @param method
     */
    void end(Method method);
}
```
实现类：LoggerOperation
```java
public class LoggerOperation implements IOperation {
    public void end(Method method) {
        Logger.logging(Level.DEBUGE, method.getName() + " Method end .");
    }
    public void start(Method method) {
        Logger.logging(Level.INFO, method.getName() + " Method Start!");
    }
}
```

代理类:DynaProxyHello
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
public class DynaProxyHello implements InvocationHandler {
    /**
     * 操作者
     */
    private Object proxy;
    /**
     * 要处理的对象(也就是我们要在方法的前后加上业务逻辑的对象,如例子中的Hello)
     */
    private Object delegate;
 
 
    /**
     * 动态生成方法被处理过后的对象 (写法固定)
     * 
     * @param delegate
     * @param proxy
     * @return
     */
    public Object bind(Object delegate,Object proxy) {
       
        this.proxy = proxy;
        this.delegate = delegate;
        return Proxy.newProxyInstance(
                this.delegate.getClass().getClassLoader(), this.delegate
                        .getClass().getInterfaces(), this);
    }
    /**
     * 要处理的对象中的每个方法会被此方法送去JVM调用,也就是说,要处理的对象的方法只能通过此方法调用
     * 此方法是动态的,不是手动调用的
    */
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        try {
        //反射得到操作者的实例
            Class clazz = this.proxy.getClass();
            //反射得到操作者的Start方法
            Method start = clazz.getDeclaredMethod("start",new Class[] { Method.class });


            //反射执行start方法
            start.invoke(this.proxy, new Object[] { method });
            //执行要处理对象的原本方法
            result = method.invoke(this.delegate, args);
            //反射得到操作者的end方法
            Method end = clazz.getDeclaredMethod("end",new Class[] { Method.class });
            //反射执行end方法
            end.invoke(this.proxy, new Object[] { method });


        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }
}
```
Spring的AOP部分借鉴：https://blog.csdn.net/fuzhongmin05/article/details/61615149