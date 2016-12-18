---
title: RxJava 初探（一）
date: 2016-10-06 00:45
categories: RxJava
tags: RxJava
---
### 一、几个基本概念
* 由来
	Rx(Reactive Extensions)，最初是LINQ的一个扩展，后由微软团队开发，在2012年11月开源。ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。
* Rx 模式
 * 创建：Rx可以方便的创建事件流和数据流
 * 组合：Rx使用查询式的操作符组合和变换数据流
 * 监听：Rx可以订阅任何可观察的数据流并执行操作
* 名词定义
	* Iterable： 可迭代对象，支持以迭代器的形式遍历
	* Observable：可观察对象，在Rx中定义为更强大的Iterable，在观察者模式中是被观察的对象，一旦数据产生或发生变化，会通过某种方式通知观察者或订阅者
	* Observer： 观察者对象，监听Observable发射的数据并做出响应，Subscriber是它的一个特殊实现
	* emit：直译为发射，发布，发出，含义是Observable在数据产生或变化时发送通知给Observer，调用Observer对应的方法。文章统一翻译为发射
	* items：直译为项目，条目，在Rx里是指Observable发射的数据项，文章统一译为数据，数据项

### 二、响应式编程模式
* 下图采自官方文档，基本阐述了数据流和数据变换的过程：
	
	![这里写图片描述](http://7xr1vo.com1.z0.glb.clouddn.com/rxjava%E7%AE%80%E5%8D%95%E5%9B%BE%E8%A7%A3.png)
* 冷热观察者
	* 热 观察者：可能一创建完就开始发射数据，因此所有后续订阅它的观察者可能从序列中间的某个位置开始接受数据（有一些数据错过了）
	* 冷 观察者：一个"冷"的Observable会一直等待，直到有观察者订阅它才开始发射数据，因此这个观察者可以确保会收到整个数据序列

### 三、操作分类
* 创建操作： Create, Defer, Empty/Never/Throw, From, Interval, Just, Range, Repeat, Start, Timer
* 变换操作：Buffer, FlatMap, GroupBy, Map, Scan和Window
* 过滤操作：Debounce, Distinct, ElementAt, Filter, First, IgnoreElements, Last, Sample, Skip, SkipLast, Take, TakeLast
* 组合操作：And/Then/When, CombineLatest, Join, Merge, StartWith, Switch, Zip
* 错误处理：Catch和Retry
* 辅助操作：Delay, Do, Materialize/Dematerialize, ObserveOn, Serialize, Subscribe, SubscribeOn, TimeInterval, Timeout, Timestamp, Using
* 条件和布尔操作：All, Amb, Contains, DefaultIfEmpty, SequenceEqual, SkipUntil, SkipWhile, TakeUntil, TakeWhile
* 算术和集合操作：Average, Concat, Count, Max, Min, Reduce, Sum
* 转换操作：To
* 连接操作：Connect, Publish, RefCount, Replay
* 反压操作：用于增加特殊的流程控制策略的操作符

> 官方文档翻译版：https://mcxiaoke.gitbooks.io/rxdocs/content/Intro.html