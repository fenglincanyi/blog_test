---
title: Spring 学习（三）
date: 2017-01-07 12:01
categories: Java后台
tags: Spring
---
> 本次学习内容：Spring AOP相关

# 问题引入

``` java
public class User {

    public void addUser() {
        // ...
    }
}
```
—— 需求，在上面原有的功能中，加入用户日志，记录用户信息？如何做？
—— 方法尝试：

``` java
public class BaseUser {
    
    public void writeLog() {
        // 写日志，记录用户信息
    }
}
```

``` java
public class User extends BaseUser{

    public void addUser() {
        // ...
        super.writeLog();
    }
}
```
这种 纵向 的抽取，并不能友好的解决问题，一旦父类的名称改变，后面也要接着更改

# AOP 原理
## 有接口

``` java
public interface Dao {
    
    void add();
}

public class DaoImpl implements Dao {
    
    public void add() {
        
    }
}
```
使用 jdk 动态代理，生成 接口实现类的代理对象，来增相应方法的功能
## 没有接口

``` java
public class User {

    public void addUser() {
    }
}
```
使用 cglib 动态代理，生成其子类的代理对象，调用父类的方法，来增强相应功能

# 几个重要 术语
* 连接点 JoinPoint：类中哪些方法可以被 增强，这些方法就是 连接点
* 切入点 PointCut：类中实际增强类那些方法（如：add,update)，这些方法称为 切入点
* 通知／增强 advice: 增强的逻辑，如：要扩展日志功能，则日志功能为 增强／通知
分为：前置通知(方法之前执行），后置通知，异常通知(方法出现异常)，最终通知(后置之后执行)，环绕通知(方法之前和之后执行)
* 切面 aspect：把增强功能应用到具体方法上，这个过程叫切面。即：把增强应用到切入点的过程
* 目标对象 target: 要增强的类，代理的目标对象
* 织入 weaving: 把增强应用到目标的过程，把advice 应用到target的过程
* 代理 Proxy: 一个类被aop织入增强后，就产生一个结果代理类

# 实现AOP
## xml 配置实现
* 切入点配置表达式：
**execution(<访问修饰符> <返回类型><方法名>(参数)<异常>）**
e.g.

        execution(* com.gjr.aop.Book.add(..))
        execution(* com.gjr.aop.Book.*(..))          book 类下的所有方法加强
        execution(* *.*(..))                         所有类下的所有方法加强
        execution(* xxx*(..))                        所有xxx开头的方法加强

<br>
demo 示例：

``` java
public class Book {

    public void show() {
        System.out.println("show book ....");
    }
}
public class MyBook {

    public void addFunctionBefore() {
        System.out.println("在前面，加点功能 。。。");
    }

    public void addFunctionAfter() {
        System.out.println("在后面，加点功能 。。。");
    }

    public void arroundFunction(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        System.out.println("之前环绕 。。。");
        proceedingJoinPoint.proceed();
        System.out.println("之后环绕 。。。");
        // 注意 环绕通知 的参数：ProceedingJoinPoint
    }
}
```
xml配置：

``` xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/aop
            http://www.springframework.org/schema/aop/spring-aop.xsd">
    <!-- 加入 aop 约束，xml 配置aop 来实现功能—>

    <bean id="book" class="com.gjr.aspect.Book"/>
    <bean id="myBook" class="com.gjr.aspect.MyBook"/>

    <aop:config>
        <!-- 切点: 需要被增强功能的方法 -->
        <aop:pointcut id="bookPoint" expression="execution(* com.gjr.aspect.Book.show(..))"/>
        <!-- 切面: 把增强方法加到需要增强的方法上 -->
        <aop:aspect ref="myBook">
            <aop:before method="addFunctionBefore" pointcut-ref="bookPoint"/>
            <aop:after method="addFunctionAfter" pointcut-ref="bookPoint"/>
            <aop:around method="arroundFunction" pointcut-ref="bookPoint"/>
        </aop:aspect>
    </aop:config>
</beans>
```
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/DE37AB73-E35C-4515-8EE9-377A606E3956.png)

3种 切面类型同时存在时：before最先调用，after最后调用
## 注解实现 aop（只列出@Before，其他的相同）

``` java
public class Order {

    public void show() {
        System.out.println("show order ...");
    }
}
@Aspect
public class MyOrder {

    @Before(value = "execution(* Order.show(..))")
    public void addFunction() {
        System.out.println("add function ...");
    }
}
```

``` xml
<bean id="order" class="com.gjr.annaop.Order"/>
<bean id="myOrder" class="com.gjr.annaop.MyOrder"/>

<!-- 注解 实现 aop-->
<aop:aspectj-autoproxy/>
```
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/83664DD4-196B-44DB-BF22-16A94F0FFA01.png)


# Spring 相关的 pom 配置
| GroupId      |     ArtifactId |   Description   |
| :-------- | --------:| :------: |
| org.springframework    |   spring-aop |  Proxy-based AOP support  |
| org.springframework    |   spring-aspects |  AspectJ based aspects  |
| org.springframework    |   spring-beans |  Beans support, including Groovy |
| org.springframework    |   spring-context |  Application context runtime, including scheduling and remoting abstractions |
| org.springframework    |   spring-context-support |  Support classes for integrating common third-party libraries into a Spring application context  |
| org.springframework    |   spring-core |  Core utilities, used by many other Spring modules  |
| org.springframework    |   spring-expression |  Spring Expression Language (SpEL)  |
| org.springframework    |   spring-instrument |  Instrumentation agent for JVM bootstrapping  |
| org.springframework    |   spring-instrument-tomcat |  Instrumentation agent for Tomcat  |
| org.springframework    |   spring-jdbc |  JDBC support package, including DataSource setup and JDBC access support  |
| org.springframework    |   spring-jms |  JMS support package, including helper classes to send and receive JMS messages  |
| org.springframework    |   spring-messaging |  Support for messaging architectures and protocols  |
| org.springframework    |   spring-orm |  Object/Relational Mapping, including JPA and Hibernate support  |
| org.springframework    |   spring-oxm |  Object/XML Mapping  |
| org.springframework    |   spring-test |  Support for unit testing and integration testing Spring components  |
| org.springframework    |   spring-tx |  Transaction infrastructure, including DAO support and JCA integration  |
| org.springframework    |   spring-web |  Web support packages, including client and web remoting  |
| org.springframework    |   spring-webmvc |  REST Web Services and model-view-controller implementation for web applications  |
| org.springframework    |   spring-webmvc-portlet |  MVC implementation to be used in a Portlet environment  |
| org.springframework    |   spring-websocket |  WebSocket and SockJS implementations, including STOMP support  |




<br>

附录
demo 地址：
https://github.com/fenglincanyi/springdemo2