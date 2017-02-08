---
title: Spring 学习（二）
date: 2016-12-27 17:01
categories: Java后台
tags: Spring
---
> 本次学习内容：Spring 依赖注入相关，本次通过xml配置实现

## setXXX() 方法注入

``` xml
<bean id="book" class="com.geng.attr.Book">
    <property name="bookName" value="西游记"/>
</bean>
```
## 构造方法注入

``` xml
// 这里通过有参构造注入
<bean id="people" class="com.geng.attr.People">
    <constructor-arg name="name" value="小明"/>
</bean>
```
## 对象注入

``` xml
<bean id="userDao" class="com.geng.obj.UserDao"/>
<bean id="userService" class="com.geng.obj.UserService">
    <property name="userDao" ref="userDao"/>
</bean>
```

``` java
public class UserDao {

    public void showUserDao() {
        System.out.println("show user dao...");
    }
}
```

``` java
public class UserService {

    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void showService() {
        System.out.println("show Service....");

        userDao.showUserDao();
    }
}
```
测试：

``` java
public class UserServiceTest {

    private ApplicationContext context;

    @Before
    public void setUp() throws Exception {
        context = new ClassPathXmlApplicationContext("beans1.xml");
    }

    @Test
    public void showService() throws Exception {
        UserService userService = (UserService) context.getBean("userService");
        userService.showService();
    }

}
```
![](http://7xr1vo.com1.z0.glb.clouddn.com/40B10214-301C-4735-8CE0-8DB0F4CCCC8B.png)

## p命名空间注入
xml 头部加入：

``` xml
xmlns:p="http://www.springframework.org/schema/p"
```
配置：

``` xml
<bean id="person" class="com.geng.attr.Person" p:pName="呵呵哒”/>
```

``` java
public class Person {

    private String pName;

    public void setpName(String pName) {
        this.pName = pName;
    }

    public void test() {
        System.out.println("result: " + pName);
    }
}
```
![](http://7xr1vo.com1.z0.glb.clouddn.com/EA20EBAD-5360-4197-BA0C-4059ABF7636F.png)

## 复杂类型注入

``` java
public class MulitDemo {

    private String[] arrs;
    private List<String> list;
    private Map<String, String> map;
    private Properties properties;

    public void setArrs(String[] arrs) {
        this.arrs = arrs;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    public void setMap(Map<String, String> map) {
        this.map = map;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    public void showAll() {
        System.out.println(arrs);
        System.out.println(list);
        System.out.println(map);
        System.out.println(properties);
    }
}
```
xml 分别配置：
``` xml
<bean id="mulitDemo" class="com.geng.collection.MulitDemo">
    <property name="arrs">
        <array>
            <value>小红</value>
            <value>小白</value>
            <value>小绿</value>
        </array>
    </property>

    <property name="list">
        <list>
            <value>大白</value>
            <value>大春</value>
            <value>大花</value>
        </list>
    </property>

    <property name="map">
        <map>
            <entry key="001" value="小白" />
            <entry key="002" value="小刘" />
            <entry key="003" value="小东" />
        </map>
    </property>

    <property name="properties">
        <props>
            <prop key="driveClass">com.mysql.jdbc.Driver</prop>
            <prop key="userName">root</prop>
            <prop key="userPwd">root</prop>
        </props>
    </property>
</bean>
```
测试：
![](http://7xr1vo.com1.z0.glb.clouddn.com/76FF4DDF-95DD-4F5A-B43F-30946D328A30.png)

<br>
附录：
demo地址：
https://github.com/fenglincanyi/springdemo1