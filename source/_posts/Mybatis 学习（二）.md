---
title: Mybatis 学习（二）
date: 2017-01-15 14:32
categories: Java后台
tags: Mybatis
---
> 本次学习内容：MyBatis 高级
> 结合demo进行学习

## 2张表关联查询，一对一查询
* 需求
查询订单的信息及用户信息

``` sql
select
orders.*,user.username,user.sex, user.address
from orders, user
where orders.user_id=user.id
```
结果集：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/0A6D42AE-B4E6-4960-8313-FAE563CBA426.png)

主要对 结果集 进行分析，再书写 mapper.xml

* resultType实现
新建一个 po , 继承自 Orders（因为此sql 中 orders表的字段较多），再增加 user 的几个字段即可.

``` java
public class OrdersCustom extends Orders{

    // 需求：
    // select orders.*,user.username,user.sex, user.address
    // from orders, user
    // where orders.user_id=user.id

    private String username;
    private String sex;
    private String address;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```

``` java
public interface OrdersCustomMapper {

    // 一对一查询 使用 resultType实现
    List<OrdersCustom> queryOrdersUser();
}
```

``` xml
<select id="queryOrdersUser" resultType="com.gjr.po.OrdersCustom">
    select
    orders.*,user.username,user.sex, user.address
    from orders, user
    where orders.user_id=user.id
</select>
```

* resultMap实现
需要使用 association 进行关联，在原来的 Orders 类中 添加 user 属性

``` java
/**
* 一对一查询：使用 resultMap实现，优点：可实现懒加载
*/
private User user;

public User getUser() {
    return user;
}

public void setUser(User user) {
    this.user = user;
}
```

``` xml
<select id="queryOrdersUser1" resultMap="OrdersUserResultMap">
    select
    orders.*,user.username,user.sex, user.address
    from orders, user
    where orders.user_id=user.id
</select>
```

``` xml
<resultMap id="OrdersUserResultMap" type="com.gjr.po.Orders">
    <!-- 配置订单信息 -->
    <id column="id" property="id"/>
    <result column="user_id" property="userId"/>
    <result column="number" property="number"/>
    <result column="createtime" property="createtime"/>
    <result column="note" property="note"/>

    <!-- 配置用户信息 -->
    <association property="user" javaType="com.gjr.po.User"> <!-- 这个 property="user" 就是 com.gjr.po.Orders 中增加的那个 user 属性-->
        <id column="user_id" property="id"/> <!-- column="user_id" 是查出来的结果中 用户的 唯一标示 -->
        <result column="username" property="username"/>
        <result column="sex" property="sex"/>
        <result column="address" property="address"/>
    </association>
</resultMap>
```
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/CB9AFFD6-B061-41CC-BC0B-0376C263DC0F.png)


## 3张表关联查询，一对多查询

**确定主表，对其 po 添加所需字段**
* 需求
-- 查询用户订单详情

``` sql
select
orders.*,
user.username,user.sex, user.address,
orderdetail.id orderdetail_id,
orderdetail.items_id,
orderdetail.items_num,
orderdetail.orders_id
from orders, user, orderdetail
where orders.user_id = user.id and orders_id = orders.id
```
结果集：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/937F9118-45DC-4D2F-83A3-8ED359C2CD7F.png)

* 实现

``` xml
<select id="queryOrdersDetailUser" resultMap="OrdersDetailUserMap">
select
    orders.*,
    user.username,user.sex, user.address,
    orderdetail.id orderdetail_id,
    orderdetail.items_id,
    orderdetail.items_num,
    orderdetail.orders_id
from orders, user, orderdetail
where orders.user_id = user.id and orders_id = orders.id
</select>

<resultMap id="OrdersDetailUserMap" type="com.gjr.po.Orders" extends="OrdersUserResultMap">
    <!-- 用户信息、订单信息、复用前面的-->
    
    <collection property="orderdetailList" ofType="com.gjr.po.Orderdetail"> <!--使用 ofType-->
        <id column="orderdetail_id" property="id"/>
        <result column="items_id" property="itemsId"/>
        <result column="items_num" property="itemsNum"/>
        <result column="orders_id" property="ordersId"/>
    </collection>
</resultMap>
```
在查询主表对应的po类， Orders 中增加字段 

``` java
private List<Orderdetail> orderdetailList;
```

## 3张表联合查询，多对多查询
* 需求
查询用户订单商品明细

``` sql
SELECT
  orders.*,
  user.username,
  user.sex,
  user.address,
  orderdetail.id orderdetail_id,
  orderdetail.items_id,
  orderdetail.items_num,
  orderdetail.orders_id,
  items.name items_name,
  items.detail items_detail,
  items.price items_price
FROM
  orders,
  user,
  orderdetail,
  items
WHERE orders.user_id = user.id AND orderdetail.orders_id=orders.id AND orderdetail.items_id = items.id
```
结果集：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/C9339305-3D30-4A4F-B349-73DC6B6C9D50.png)

* 实现

``` xml
<select id="queryOrdersDetailUserItems" resultMap="OrdersDetailUserItemsMap">
--     查询用户订单商品明细
        SELECT
          orders.*,
          user.username,
          user.sex,
          user.address,
          orderdetail.id orderdetail_id,
          orderdetail.items_id,
          orderdetail.items_num,
          orderdetail.orders_id,
          items.name items_name,
          items.detail items_detail,
          items.price items_price
        FROM
          orders,
          user,
          orderdetail,
          items
        WHERE orders.user_id = user.id AND orderdetail.orders_id=orders.id AND orderdetail.items_id = items.id
    </select>


<resultMap id="OrdersDetailUserItemsMap" type="com.gjr.po.User">
    <!-- 用户信息-->
    <id column="user_id" property="id"/> <!-- 根据查询结果 写 column !!! -->
    <result column="username" property="username"/>
    <result column="sex" property="sex"/>
    <result column="address" property="address"/>


    <!-- 订单信息: 一个用户对应多个订单-->
    <collection property="ordersList" ofType="com.gjr.po.Orders">
        <id column="id" property="id"/>
        <result column="user_id" property="userId"/>
        <result column="number" property="number"/>
        <result column="createtime" property="createtime"/>
        <result column="note" property="note"/>


        <!-- 订单明细：一个订单对应多个明细 -->
        <collection property="orderdetailList" ofType="com.gjr.po.Orderdetail">
            <id column="orderdetail_id" property="id"/>
            <result column="items_id" property="itemsId"/>
            <result column="items_num" property="itemsNum"/>
            <result column="orders_id" property="ordersId"/>


            <!-- 商品信息：一个明细对应一个商品 -->
            <association property="items" javaType="com.gjr.po.Items">
                <id column="items_id" property="id"/>
                <result column="items_name" property="name"/>
                <result column="items_detail" property="detail"/>
                <result column="items_price" property="price"/>
            </association>
        </collection>
    </collection>
</resultMap>
```
各个 po 类需要分别增加相关字段即可，安装关联的类型和字段

## 懒加载
配置 mybatisConfig.xml 相关参数：

``` xml
<settings>
    <!-- 打开 延迟加载 的开关-->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 设置按需加载 -->
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

``` xml
    <select id="findOrdersUserLazyLoading" resultMap="OrdersUserLazyLoadingResultMap">
--         懒加载测试
        SELECT * FROM orders
    </select>
```
用 association 实现延迟加载：

``` xml
<resultMap id="OrdersUserLazyLoadingResultMap" type="com.gjr.po.Orders">
    <!--对订单信息进行映射配置  -->
    <id column="id" property="id"/>
    <result column="user_id" property="userId"/>
    <result column="number" property="number"/>
    <result column="createtime" property="createtime"/>
    <result column="note" property="note"/>
    
    <association property="user" javaType="com.gjr.po.User"
                 select="com.gjr.mapper.UserMapper.findUserById"
                 column="user_id">
        <!-- 实现对用户信息进行延迟加载 -->

    </association>
</resultMap>
```
测试：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/51C2ECFE-8AD9-4354-B9D1-3E0B7DC66FE8.png)

## mybatis 缓存相关
* mybatis默认开启一级缓存，二级缓存
* Mybatis查询得数据，缓存至一级缓存，再缓存至二级缓存中，二级缓存是 mapper 级别的，凡是同一个mapper，都会有自己的二级缓存区
* 多个sqlsession可以同享同一个 mapper的二级缓存

## 总结
* resultMap可以实现高级映射（使用association、collection实现一对一及一对多映射），association、collection具备延迟加载功能。
* 延迟加载：先从单表查询、需要时再从关联表去关联查询，大大提高数据库性能，因为查询单表要比关联查询多张表速度要快。
* 查询结果 必须要和 pojo 类型保持一致
* 联系实际问题的关系类型（如：一个订单对应一个用户，一个订单对应多个明细），进行 po 的修改或扩展
* association, collection 分别是 关系类型为：1对1，1对多   配置时分别是 javaType 、ofType

<br>

附录
demo 地址：
https://github.com/fenglincanyi/mybaitsdemo2