---
title: Mybatis 学习（一）
date: 2017-01-11 12:01
categories: Java后台
tags: Mybatis
---
> 本次学习内容：MyBatis 基础

## 介绍
mybatis 是一个持久层框架，是apache下的开源项目
是一个半自动化框架，与Hibernate相比，需要开发者手动写sql，这样更为灵活配置
mybatis 可以将PreparedStatement 中输入的参数自动进行输入映射，将查询结果灵活映射成 Java 对象

## 基本原理
执行过程：

SqlMapConfig.xml/mapper.xml … —>
SqlSessionFactory —> 
SqlSession —> 
Executor —> 
mapper statement —> 
db

* 配置数据源，映射文件
* 创建SqlSession 交给执行器操作数据库
* 操作数据库
* 对操作数据库存储做封装，包括SQL语句，输入参数，输出结果

## 遇到问题
报错：
org.apache.ibatis.binding.BindingException: Invalid bound statement (not found):    ………
然后，发现target目录下，XxxMapper.xml 没有在 target 下相应的目录生成
![](http://7xr1vo.com1.z0.glb.clouddn.com/4B5E209A-0C22-4072-8B8D-AD3532042E98.png)

原因：

    在使用maven等构建工具时，默认会将源码编译后再加上资源目录的文件放到target目录下作为最后运行的文件（可以是war,jar,或者目录）。有时为了方便，我们会在src/main/java源码目录下放了资源文件，例如mybatis的mapper文件，方便我们编程时展开查看。这时，我们需要设置编译时也将这些配置文件放到target目录下，否则最后的target目录式没有这些文件的。

解决：
在maven中，在build元素中添加：

``` xml
<build>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
        </resource>
    </resources>
</build>
```

## 总结
* Mybatis 框架实际上 是对原来 jdbc 代码的封装，将繁琐重复的代码，交付给配置文件和接口类进行灵活的对象映射
* 动态sql 有点像 jstl，很容易理解，目的就是为了灵活的玩 sql 语句
* 对于各种类型的sql 语句，需要记住mybatis的处理方式即可
* Mybatis 是将 对象或简单类型输入，将结果以数据实体/简单类型输出，省去了 jdbc 的复杂操作
* 进行 事务性操作时，一定要 commit，否则不生效
sqlSession.commit();

## mybatis 插件
附录：
    使用：https://www.oschina.net/p/intellij-mybatis-plugin
    下载：https://github.com/CHN-Jaylin/Plugins-Cracked （破解的)


<br>

参考：
http://ask.csdn.net/questions/226091

附录
demo 地址：
https://github.com/fenglincanyi/mybaitsdemo1