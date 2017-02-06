---
title: Spring 学习（一）
date: 2016-12-26 14:06
categories: Java后台
tags: Spring
---
## Spring的2个基本概念
* aop
面向切面编程
在原有的基础上进行扩展，而不是进行修改。符合 开闭原则
* ioc
控制反转
不通过手动 new 方式，来创建对象，而是交给Spring 容器根据配置，进行创建。从而将类的对象交给Spring 进行控制管理

	* ioc 的2种方式来创建对象：
	（1）通过配置文件
	（2）通过注解
<br>
	* ioc 实现原理
	
	通过dom4j 解析 xml 文件，拿到类的全路径，然后通过反射的技术创建该类的对象，结合工厂模式返给调用方
伪代码实现说明：

```
<bean id=“userService” class=“com.geng.UserService” />
public class Factory {
    public static UserService getUserService() {
        String classValue = dom4j.getValue(“userService”);
        Class clazz = Class.forName(“classValue”);
        return clazz.newInstance();
    }
}
```
## Spring 运行时
![](http://7xr1vo.com1.z0.glb.clouddn.com/spring-overview.png)
Spring 提供了全面的Java web 服务框架，从web层到业务层，再到持久层，都有着相关模块的实现。
对于Spring 最基本使用，必须尹若 Core Container中的4个重要的部分，方可使用。
若只引用某部分的库，maven也会自动引用最核心的jar包到你的应用中。

## Spring 第一个Demo 开发
在 idea 中，创建 maven 项目，然后，编辑 pom.xml 文件。
引入：Spring 主要的几个库，log4j，junit

```
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.3.5.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
        <version>1.2.17</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
```

在resource 目录下，创建 xml 配置文件，如图：
![](http://7xr1vo.com1.z0.glb.clouddn.com/A5CEBFD6-CFE5-404A-A42F-CF521BAB4B47.png)

Idea 在你编辑时候，会提示相关的属性，方便书写配置

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd”>

    <bean id="user" class="com.geng.ioc.User"/>

</beans>
```
![](http://7xr1vo.com1.z0.glb.clouddn.com/9FEC554D-7DDC-48C7-BA89-3FAAAC181312.png)

创建测试用例：

```
public class UserTest {

    private ApplicationContext context;

    @Before
    public void setUp() throws Exception {
        context = new ClassPathXmlApplicationContext("beans1.xml");
    }

    @Test
    public void add() throws Exception {
        User user = (User) context.getBean("user");
        user.add();
    }

}
```
![](http://7xr1vo.com1.z0.glb.clouddn.com/979F1DF8-ADF4-499F-8119-62CDDE033AD4.png)

至此，我们完成了第一个 Spring demo
## bean 的管理
* 通过无参构造创建（前面第一个demo）
* 通过静态工厂实现对象创建
代码示例：

```
public class BeanFactory {

    public static User2 createUser2() {
        return new User2();
    }
}
```

```
public class User2 {

    public void add() {
        System.out.println("user2.....”);
    }
}
```
xml中配置：

```
<!--用静态工厂创建-->
<bean id="user2" class="com.geng.ioc.BeanFactory" factory-method="createUser2”/>
```
测试：

```
public class User2Test {

    private ApplicationContext context;

    @Before
    public void setUp() throws Exception {
        context = new ClassPathXmlApplicationContext("beans1.xml");
    }

    @Test
    public void add() throws Exception {
        User2 user2 = (User2) context.getBean("user2");
        System.out.println(user2);
        user2.add();
    }

}
```
![](http://7xr1vo.com1.z0.glb.clouddn.com/0815D096-5833-4AA4-9BD8-EE493BE15F32.png)
* 通过实例工厂创建对象

```
<!--用实例工厂创建—>
<bean id="bean2Factory" class="com.geng.ioc.Bean2Factory" />
<bean id="user2" factory-bean="bean2Factory" factory-method="getBean"/>
```
这里不再验证。。。

## Spring配置文件中的几个重要属性

id：不能还有特殊符号，“_”是可以的
name：可以含有特殊符号，如：#，历史版本中使用的，后期不推荐使用。context.getBean() 方法可以获取这两种属性的值
class：类的全路径
scope：作用域  

	singleton(默认)  
	prototype(多例)
	request 创建对象放在request域
	session 创建对象放在session域
	globalSession 一次登陆，任何地方都保存有登录状态

## 附录：IDEA Resource 目录下，存放的文件类型
![](http://7xr1vo.com1.z0.glb.clouddn.com/23395798-F73A-4EBB-95E2-1641C5EF8A24.png)

（IDEA 官网：https://www.jetbrains.com/help/idea/2016.3/resource-files.html ）

所以，在此目录下，我们一般存放配置文件，或一些必要的资源文件。

Demo 地址：
https://github.com/fenglincanyi/springdemo1