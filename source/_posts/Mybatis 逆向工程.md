---
title: Mybatis 逆向工程
date: 2017-01-17 10:11
categories: Java后台
tags: Mybatis
---

## pom 配置
pom.xml 中 加入：

``` xml
<plugins>
    <plugin>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-maven-plugin</artifactId>
        <version>1.3.5</version>
        <configuration>
            <verbose>true</verbose>
            <overwrite>true</overwrite>
        </configuration>
    </plugin>
</plugins>
```
## maven配置
![](http://7xr1vo.com1.z0.glb.clouddn.com/5709477C-1A4B-4AEF-B0E6-5EBE20688677.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/8C4EFD37-2282-4BEA-81C7-4C4E11D2A91C.png)

命名：
mybatis-generator:generate -e

![](http://7xr1vo.com1.z0.glb.clouddn.com/C97FA1F0-DEDF-4A13-9810-6029D03D97DE.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/4EAE9177-A8D0-411D-869D-2167D54CB026.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/E3C00FBA-21F9-4B6B-80BE-74D553270C6A.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/D0FC7201-E1D6-4D16-90E6-87D079213CA3.png)

注意：
文件名必须是：generatorConfig.xml
否则报错： configfile /Users/geng/ssm/Demo/ssmdemo/src/main/resources/generatorConfig.xml does not exist

## mybatis config.xml 文件配置
拷贝驱动包的绝对路径
![](http://7xr1vo.com1.z0.glb.clouddn.com/68C3A3B3-DB05-4786-A412-C808F0A8AA17.png)

xml 配置：

``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE generatorConfiguration PUBLIC
        "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>

    <!-- !!!! Driver Class Path !!!! —>
    <classPathEntry location="/Users/geng/ssm/Demo/mybaitsdemo2/src/main/resources/mysql-connector-java-5.1.7-bin.jar"/>

    <context id="context" targetRuntime="MyBatis3">
        <commentGenerator>
            <property name="suppressAllComments" value="false"/>
            <property name="suppressDate" value="true"/>
        </commentGenerator>

        <!-- !!!! Database Configurations !!!! -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/mybatis_demo2_db?characterEncoding=utf-8"
                        userId="root"
                        password="root"/>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- !!!! Model Configurations !!!! -->
        <javaModelGenerator targetPackage="com.gjr.testgenerator.po" targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- !!!! Mapper XML Configurations !!!! -->
        <sqlMapGenerator targetPackage="com.gjr.testgenerator.mapper" targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!-- !!!! Mapper Interface Configurations !!!! -->
        <javaClientGenerator targetPackage="com.gjr.testgenerator.mapper" targetProject="src/main/java" type="XMLMAPPER">
            <property name="enableSubPackages" value="false"/>
        </javaClientGenerator>

        <!-- !!!! Table Configurations !!!! -->
        <table tableName="items" enableCountByExample="false" enableDeleteByExample="false" enableSelectByExample="false"
               enableUpdateByExample="false"/>
        <table tableName="orders" enableCountByExample="false" enableDeleteByExample="false" enableSelectByExample="false"
               enableUpdateByExample="false"/>
        <table tableName="orderdetail" enableCountByExample="false" enableDeleteByExample="false" enableSelectByExample="false"
               enableUpdateByExample="false"/>
        <table tableName="user" enableCountByExample="false" enableDeleteByExample="false" enableSelectByExample="false"
               enableUpdateByExample="false"/>

    </context>
</generatorConfiguration>
```

## 运行逆向生成代码

![](http://7xr1vo.com1.z0.glb.clouddn.com/90C77448-2475-4EE3-B0D3-BF49D2A79606.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/617ABCDF-11AB-45AF-BA9C-1150393FEDC9.png)

运行成功

<br>

附录
demo 地址：
https://github.com/fenglincanyi/mybaitsdemo2