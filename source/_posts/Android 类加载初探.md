---
title: Android 类加载初探
date: 2016-11-17 21:08
categories: Android
tags: ClassLoader
---
> 源码路径(此版本：Android 5.0)：
android-5.0.0_r7\libcore\dalvik\src\main\java\dalvik\system

需要关注的类有：
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader1.png)
### 一、查找类的过程
对于一个Class，在Android中，是如何被ClassLoader查找的呢？我们先查看一下，在Android中最原始的ClassLoader: BaseDexClassLoader 中有直接的方法。直接看源码：
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassLoader2.png)

再继续追， pathList 的 findClass() 方法：
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassLoader3.png)

此时遍历每个dex，通过二进制名来查找类Class；若类未定义或类未找到，则将异常add至suppressed异常集合中，随后抛出。
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassLoader4.png)

### 二、3个类加载器的关系

我们先来看看这3个ClassLoader的定义

* BaseDexClassLoader

![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader5.png)

* DexClassLoader

![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader6.png)

* PathClassLoader

![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader7.png)

由上面可看出：BaseDexClassLoader 继承自最原始的 ClassLoader，DexClassLoader 和 PathClassLoader都继承自BaseClassLoader。依据传入的参数不同，来实现各自不同的 dex 加载功能

**ClassLoader 相关说明：**

从存储中加载类和资源。 在运行时安装一个或多个类加载器。 每当运行时系统需要在内存中尚不可用的特定类时，都会查询这些类。 通常，类加载器是一个树型结构，其中子类加载器将所有请求委托给父类加载器。 只有父类加载器无法满足请求，子类加载器才会尝试处理它（委托机制）

下面我们再继续分析，传入不同的 optimizedDirectory 参数，两者会有什么样的区别？

DexPathList 构造函数：
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader8.png)

makeDexElements过程：
![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader9.png)

加载DexFile，若optimizedDirectory 目录为空，则初始化一个DexFile，否则，直接加载dex文件：

![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader10.png)

初始化DexFile文件，通常是从一个文件对象中打开一个Dex file ，通常是一个 内容是 classes.dex 的 zip/jar文件，
虚拟机将在 /data/dalvik-cache 目录下生成相应名称的文件，且打开它，在系统权限允许的情况下尽可能的先创建或更新该文件 
不能传递 优化后的 文件名（/data/dalvik-cache），而是在 dexopt 之前的原始状态的文件名。

![](http://7xr1vo.com1.z0.glb.clouddn.com/AndroidClassloader11.png)

由上可知，PathClassLoader 的构造函数，与DexClassLoader相比较，optimizedDirectory 优化目录为null。
由于optimizedDirectory是用来缓存我们需要加载的dex文件的，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile对象。
optimizedDirectory必须是一个内部存储路径，无论哪种动态加载，加载的可执行文件一定要存放在内部存储。DexClassLoader可以指定自己的optimizedDirectory，所以它可以加载外部的dex，因为这个dex会被复制到内部路径的optimizedDirectory
所以DexClassLoader可以通过其他路径（内部存储路径）加载dex，而PathClassLoader没有optimizedDirectory，所以它只能加载内部的dex，这些大都是存在系统中已经安装过的apk里面的。

若想查看相关目录下的dex文件，可参考上一篇文章。



