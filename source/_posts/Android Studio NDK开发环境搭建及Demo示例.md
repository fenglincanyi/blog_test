---
title: Android Studio NDK开发环境搭建及Demo示例
date: 2016-01-16 19:34
comments: true
categories: Android
tags: [JNI, NDK]
---
>  说明：Android Studio 1.4后支持C/C++开发，1.3之前的版本坑点较多，所以使用1.4后的版本较为容易

### 所用工具版本
* Android Studio 1.5
* android-ndk-r10e-windows-x86_64.exe

### 配置NDK环境
* 自行下载 **ndk** 工具，在你的要安装 ndk 的目录下，直接双击 此安装包，就会自动安装至此目录下，过程中会打开cmd窗口，命令行不断执行命令，需等待一会即可

* cmd窗口自动关闭后，会出现 **ndk** 的文件夹，如图： 

 ![](http://7xr1vo.com1.z0.glb.clouddn.com/1.jpg)![](http://7xr1vo.com1.z0.glb.clouddn.com/2.png)

* 配置**ndk**环境变量，path 中添加即可，如我的路径如下：

	> **PATH： D:\android-ndk-r10e**
* 保存后，打开cmd,输入命令:
	> **ndk-build** 

	出现如下显示，则安装成功：

	![](http://7xr1vo.com1.z0.glb.clouddn.com/3.png)

### 第一个NDKDemo
* studio中，新建一个project，	name： NDKDemo
* 在本项目中，配置 **ndk** 路径，如图：

	![](http://7xr1vo.com1.z0.glb.clouddn.com/4.png)
	![](http://7xr1vo.com1.z0.glb.clouddn.com/5.png)

	点击**OK**， 此时，等待gradle构建，构建完成后，观察 **local.properties** 文件，多出来了 **ndk** 的路径，如图所示：
	![](http://7xr1vo.com1.z0.glb.clouddn.com/6.png)
* 打开 **grade.properties** 文件，在末尾添加：

	> **android.useDeprecatedNdk="true"**


	![](http://7xr1vo.com1.z0.glb.clouddn.com/7.png)
		
		* 此步骤，为了防止下一步修改 app/build.gradle 文件后报错
* 打开 **app/build.gradle** 文件，在 **defaultConfig** 下加入**ndk**相关配置参数：

		ndk {
            moduleName "HelloNDK"
        }
![](http://7xr1vo.com1.z0.glb.clouddn.com/8.png)

		* 此处的 modleName 是 加载库文件的标识，必须和后面代码中的 System.loadLibrary("HelloNDK") 保持一致，否则会报错
	
* 书写 **native** 方法 和 加载类库，代码如下：

	    static {
			// 加载类库
	        System.loadLibrary("HelloNDK");
	    }
	
	    // native方法 调用 C代码
	    public native String javaCallC();
如图，我这是后续的图片，过程中的未截：
	
	![](http://7xr1vo.com1.z0.glb.clouddn.com/10.png)
* 打开左下角的 **Terminal**，**cd** 至 **java** 路径下，并执行命令：

		javah -d jni [native方法所在类的全路径]

	如图：

	![](http://7xr1vo.com1.z0.glb.clouddn.com/11.png)

		* 在studio中：类的全路径可 通过 右击该类 -> Copy Reference获得
	执行成功后，刷新下目录，可看到目录中多了一个 **咖啡色的jni目录**，和一个 **.h文件**，如图：

	![](http://7xr1vo.com1.z0.glb.clouddn.com/12.png)
* 右击 **Main目录的图标 -> New -> Floder -> JNI Floder**，之后会出现一个 **蓝色的JNI** 文件夹，如图：

	 ![](http://7xr1vo.com1.z0.glb.clouddn.com/13.png)
	
	此时，继续解决上面 **native** 方法名报错（红色）的问题，**Alt + Enter** 选择创建 **.c** 中的方法，这时，会在蓝色的JNI文件夹下自动生成一个 **.c** 文件，并且 含有 **c代码**，里面有复杂的方法头

	![](http://7xr1vo.com1.z0.glb.clouddn.com/9.png)

	![](http://7xr1vo.com1.z0.glb.clouddn.com/14.png)

		* 可能 native 方法名仍然是红色的，这可能是 Studio 的一个bug，但其实是可以运行的。可能过一段时间红色会消失变为正常，我的就是这种情况
* 下面就可以书写真正的 **C代码**了，此处 只简单写下，关于更牛逼的用法，将更新在后面blog中，继续总结。

	此处代码改为：
		
		return (*env)->NewStringUTF(env, "你好，NDK");

*  OK，第一个NDK Demo可以运行了，上图：

	![](http://7xr1vo.com1.z0.glb.clouddn.com/16.png)

### 总结
* 相比以前在 **eclipse** 中进行 **NDK** 开发，**studio**中显得方便多了，并不用安装 **CDT** 和 **cygwin**，而且 **.c** 文件及其内容可以自动生成
* 利用命令 **javah -d jni [native方法所在类的全路径]**  生成的  **.h头文件** 可以在 有了 **.c** 文件后删除，此 **.h** 文件主要是用来用于生成 **.c** 文件中相应比较长的方法头。如果之前没有 **.h** 文件，**Alt + Enter** 是无法自动生成 **.c**文件的
* 必须在 有 **/src/main/jni** 的文件夹（蓝色的jni文件夹）下，才能自动生成**.c** 文件
* **.c** 文件中要出现了中文，必须是 **UTF-8** 下，否则，运行时会崩溃报错
* 注意观察 **/app/build/intermediates/ndk** 目录下，有相关的 所有平台的 .so 文件（默认情况下，生成所有平台的；若只想生成某几种平台的，可在 **app/build.gradle（abi-Filters "armeabi","armeabi-v7a","x86"）** 中  或者 **Android.mk**中 进行配置）

	 ![](http://7xr1vo.com1.z0.glb.clouddn.com/17.png)
### 补充
* so文件和native类与混淆无关，无需keep语句
* 实现类一定要方法名正确否则会报java.lang.UnsatisfiedLinkError: dlopen failed异常
* studio工具的提示并不是很智能，所以有时要rebuild project