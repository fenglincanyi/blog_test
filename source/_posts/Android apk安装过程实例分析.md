---
title: Android apk安装过程实例分析
date: 2016-11-17 21:08
categories: Android
tags: [Dalvik, ART]
---
> 注：本文不对apk安装之前，系统所做的复杂工作做分析，只针对普通的apk安装过程进行简单的过程理解

一般情况下，在apk安装时，系统一般会显示一个安装界面，获取用户同意之后进行安装，并且有一些apk预处理相关的操作，紧接着，会启动app界面，进入app。
在这一过程中，不同版本的Android系统在安装时，对包的处理是不太一样的，以下分Dalivk、ART两种虚拟机进行分析。这里主要采用原生模拟器，进行分析。

### 一、 Dalvik（JIT：just in time  即时编译）

- 过程分析：
 - 点击apk包后或apk下载后，安装之前，系统会启动安装界面PackageInstallerActivity，让用户进行授权选择，用户同意后，会调用ApplicationPackageManager 进行相关操作。
 - 调用ApplicationPackageManager，通过IPC来调用 PackageManagerService 中的 installPackage 方法，在这个方法中，又去调用 installPackageAsUser 方法，此时，要进行权限相关校验，完毕之后，会通过 handler 发送消息（类型：INIT_COPY，MCS_BOUND）
 - 然后 startCopy —> handleStartCopy 依次调用，这里，对安装位置、存储空间进行处理，之后会 拷贝apk 相关文件放在相应的目录下。这些文件主要包括：
	   - apk文件
	   - jar、so文件
	   - db文件（若有的话）
 - 拷贝完成之后，进行预处理，对于Dalvik来说，dexopt要进行dex文件的优化，生成odex文件，在app运行时，能加快其启动速度。
 - 另外，对于资源处理，是 AssetMananger 来进行资源的解析、加载。


- 实例考察：
	- 安装时logcat日志：拷贝工作，然后进行dex优化，dexopt执行操作

![](http://7xr1vo.com1.z0.glb.clouddn.com/apk%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B.png)

   
   拷贝后，目录如下：
	
![](http://7xr1vo.com1.z0.glb.clouddn.com/copy%20lib%E7%BB%93%E6%9E%9C.png)

 dex位置：
	
![](http://7xr1vo.com1.z0.glb.clouddn.com/dex%E4%BD%8D%E7%BD%AE.png)

如果把此位置的 dex 删除后，且杀掉已经运行的进程后，再次点击app启动后奔溃，如图：

![](http://7xr1vo.com1.z0.glb.clouddn.com/%E5%88%A0%E6%8E%89dalvik-cache%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84dex%E5%90%8E%EF%BC%8C%E4%B8%94%E6%9D%80%E6%8E%89%E5%B7%B2%E7%BB%8F%E8%BF%90%E8%A1%8C%E7%9A%84%E5%BA%94%E7%94%A8%E8%BF%9B%E7%A8%8B%EF%BC%8C%E5%86%8D%E6%AC%A1%E5%90%AF%E5%8A%A8.png)

由此可知，要启动app时，需要加载 /data/dalik-cache 目录下的dex文件

相关参考文献：
http://www.woaitqs.cc/android/2016/07/28/android-plugin-get-apk-info
http://blog.csdn.net/luoshengyang/article/details/8852432

### 二、ART（AOT：Ahead of time  预编译）

Android 4.4之后，对于Android虚拟机又继续做了优化，art 代替了 Dalvik，对于dex文件，优化工作做了改变。
- 大致的处理过程：
		
- 编译时，通过调用 dex2oat 对dex 进行预编译，这个编译器参考了LLVM框架，默认情况下，是采用了quick模式，但在6.0之后，LLVM被彻底去除了。关于LLVM细节，自行Google。
- 编译后生成的文件 .oat 文件，实际上是Android 私有的ELF文件。在Linux系统中，ELF文件主要分为3类：目标文件（ .o）、共享文件（ .so）、可执行文件。此处Android的 .oat文件属于可执行文件。
	oat 文件含有ELF文件正常的结构形态，里面既包含有dex 文件，也包含编译好的本地指令代码。被包含的dex文件也可以是多个。
	
	
- 实例考察
 - ART 安装apk时，日志记录：

![](http://7xr1vo.com1.z0.glb.clouddn.com/art%E5%AE%89%E8%A3%85apk%E7%9A%84log.png)
	 可以看到，有apk的拷贝，有dex2oat的dex优化过程。再追一下相关的目录：
	 

 ![](http://7xr1vo.com1.z0.glb.clouddn.com/local%E7%9B%AE%E5%BD%95%E4%B8%8B.png)
 
  ![](http://7xr1vo.com1.z0.glb.clouddn.com/dalvik-cache%E7%9B%AE%E5%BD%95%E4%B8%8B.png)
  
里面有一部分是系统框架层的相关文件。
	
安装后的目录结构也有变化：

![](http://7xr1vo.com1.z0.glb.clouddn.com/data_app%E7%9B%AE%E5%BD%95%E4%B8%8B.png)

生成相应的平台下的文件，此处我使用的是x86的模拟器

Android 在编译时用到的 dexopt、dex2oat、aapt 都在 /system/bin目录下：

![](http://7xr1vo.com1.z0.glb.clouddn.com/4.4%E4%B8%8Bsystem_bin%E4%B8%8B%E7%9A%84aapt.png.png)

当时对于 multidex 的包也进行了目录查找，如下，留下后续研究做参考。

![](http://7xr1vo.com1.z0.glb.clouddn.com/multidex.png)

相关参考文献：
https://mssun.me/blog/android-art-runtime-2-dex2oat.html
http://blog.csdn.net/luoshengyang/article/details/39307813