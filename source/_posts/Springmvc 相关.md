---
title: Springmvc 相关
date: 2017-01-21 14:55
categories: Java后台
tags: Springmvc
---
## springmvc 框架原理
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/mvc.png)

springmvc执行流程：
1. 前端控制器（DispatcherServlet）, 接受请求，然后请求 处理器映射器（HanderMapping）
2. HanderMapping 根据 xml / 注解 进行查找相关的 handler，并返回给 前端控制器
3. 前端控制器 调用 处理器适配器（HanderAdapter）按照一定规则去执行 handler
4. handler执行完毕后，返回给 HanderAdapter ModelAndView，HandlerAdapter 再返回给 前端控制器
5. 前端控制器 将 ModelAndView 发送给 视图解析器（ViewResolver），试图解析器根据试图名解析为真正的视图
6. 视图解析器解析后，将view 返回给 前端控制器，前端控制器 进行视图渲染，视图渲染将模型数据(在ModelAndView对象中)填充到request域
7. 前端控制器 向用户相应结果

重要的组件：
* 前端控制器 DispatherServlet：接受请求，相应结果，转发器的作用
* 处理器映射器 HandlerMapping: 根据配置查找 handler
* 处理器适配器 HandlerAdapter: 按照规则去执行 handler
* 处理器 Handler: 业务处理
* 视图解析器 ViewResolver: 进行视图解析，将ModelAndView解析为真正的view


## springmvc 基本配置
如果不在 springmvc.xml 配置相关的映射器、适配器，spring会使用默认的，默认的配置 在 DispatcherServlet.properties 中：
这些默认的都不建议使用了。
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/B77F53FF-BBEF-40EC-BD0A-E176EB7AD8DA.png)


``` java
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
   org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
   org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
   org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
   org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
   org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

Spring 3.1之后 使用的 映射器 和 适配器：

org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter


springmvc.xml 中的约束配置，参考文档进行配置：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/5665FF5D-517B-45F0-819D-3707A9534C8B.png)

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans         
        http://www.springframework.org/schema/beans/spring-beans.xsd         
        http://www.springframework.org/schema/mvc         
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
</beans>
```
基本配置：
web.xml：

``` xml
<!DOCTYPE web-app PUBLIC
        "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
    <display-name>springmvcdemo1</display-name>

    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

        <!-- 设置 springmvc 的配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvcConfig.xml</param-value>
        </init-param>
    </servlet>

    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <!-- 访问 action 结尾的，由 DispatcherServlet 来解析-->

        <!--http://localhost:8080/springmvcdemo1/queryItems.action-->
        <!--<url-pattern>*.action</url-pattern>-->

        <!--http://localhost:8080/springmvcdemo1/queryItems-->
        <url-pattern>/</url-pattern>

    </servlet-mapping>

</web-app>
```
springmvc.xml 基本配置： 

``` xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
      http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
      http://www.springframework.org/schema/mvc
      http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd">

    <!-- 注解的 映射器、适配器-->
    <!--<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>-->
    <!--<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"/>-->


    <!-- 此配置默认 设置了 映射器、适配器，而且加载了许多的参数绑定，如json的自动转换
        开发时候使用它 ：   -->
    <mvc:annotation-driven />

    <!-- 视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <!-- 使用扫描controller,service等等 -->
    <context:component-scan base-package="com.gjr.controller"/>
</beans>
```

## 参数绑定
此处举例，具体看项目
@RequestMap中的简单类型，直接安装相关参数进行设置即可
对于包装类型 pojo, 客户端请求时候，相关的参数必须要和Pojo的属性名称一致

如：

``` java
public class ItemsQueryVo {

    // 商品信息
    private Items items;

    // 为了扩展性，对生成的 po 进行扩展
    private ItemsCustom itemsCustom;
}
```

``` java
@RequestMapping(value = "/editItemsSubmit")
public ModelAndView editItemsSubmit(Integer id, ItemsQueryVo itemsQueryVo) throws Exception {

    return modelAndView;
}
```
前端页面请求时候，表单信息中：

``` xml
<input name=“itemsCustom.name” …/>
```

## Spring 校验相关

使用的是 Hibernate 的校验框架：

``` xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>5.3.4.Final</version>
</dependency>
```

![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/BD15A0C3-C8A8-4C0E-84AB-656C19A1C1B0.png)

具体看项目。。。

<br>

附录
demo 地址：
https://github.com/fenglincanyi/ssmdemo