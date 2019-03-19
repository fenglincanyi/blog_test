---
title: instant run 相关分析
date: 2016-12-17 13:00
categories: Android
tags: instant run
---
Android Studio 2.0 引入了即时编译功能：instant run，一定程度上进行了增量编译、增量更新代码，节省了开发耗时（喝咖啡的时间）。
下面具体分析下instant run相关工作流程和相关的源码

### instant run 使用
#### 版本要求
Gradle 2.0 以上
build.gradle：minSdkVersion 15 以上（设置21以上可获得最佳性能）
Android 5.0以上的手机或模拟器

#### 使用
首先确认开启instant run，在settings中搜索instant run，可看到相关设置，默认instant run功能是开启的
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/Image.png)

当第一次点击 run 按钮 ![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/Image1.png)，进行第一次编译打包。
apk成功安装之后，再观察工具栏，run按钮发生了变化：![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/Image2.png)

然后我们随意修改一部分代码，点击运行，可看到手机屏幕上弹出toast，提示代码已经改变了，可看到最新的运行效果。使用起来也比较方便、快捷。

#### 更新方式
 * 热交换 hot swap
更改现有方法的实现代码；不会重新初始化正在运行的app，不要做重启app，activity的操作，即可看到最新代码运行结果
 * 温交换 warm swap
更改或移除当前的资源；activity会自动重启（小闪烁），即可看到最新运行结果
 * 冷交换 cold swap
 对代码有结构性的更改（字段更改、类继承关系、清单文件更改）；此时会重启app

![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/1-Y-gIucAGyqQscMdXuHaSiA.png)

参考：
https://developer.android.com/studio/run/index.html?hl=zh-cn
https://medium.com/google-developers/instant-run-how-does-it-work-294a1633367f#.go1u6yq2o

### 过程分析
#### 第一次打包
instant run 第一次编译打包流程，会执行下面的工作
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/1-U2tXGUWaeDU7L3u9Z_U0fw.png)

先来看看生成的apk：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20apk.png)
 
 多出了 instant-run.zip文件，那它里面是什么内容呢？
 ![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20zip%20class.png)
 instant-run.zip里的dex文件，是我们真正的业务代码

 那instant run 相关的类呢，反而跑到了外层的classes.dex和classes2.dex中。
 
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/%E6%97%A0%E6%A0%87%E9%A2%98.png)
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20%E7%9B%B8%E5%85%B3jar.png)

实际上这2个dex中的内容是instant-run.jar和instant-run-bootstrap.jar 的内容（自己可反编译出来看看）：

**就是说，第一次运行打包时候，是将 instant-run.jar 和 instant-run-bootstrap.jar 2个jar 变成 2个dex文件，真正的业务代码编译后整合到别的dex中，然后放在了instant-run.zip中**
> classes.dex  ->   instant-run.jar &nbsp;&nbsp;&nbsp;&nbsp; instant run 相关api类
> classes2.dex  ->  instant-run-bootstrap.jar   &nbsp;&nbsp;  AppInfo.class

再来看看清单文件，application 被替换成 BootstrapApplication：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20application.png)

#### instant run 代码分析
##### attachBaseContext() 中执行的三个步骤
首先来观察下该类下的 attachBaseContext()方法，其中做了3个比较重要的事情：
createResources() 、setupClassLoaders()、createRealApplication()

* createResources() 
主要是判断资源resource.ap_是否改变，然后保存resource.ap_的路径到externalResourcePath中
* setupClassLoaders()
设置instant run 相关的classLoader，及其继承关系（PathClassLoader -> BootClassLoader   变为  PathClassLoader -> IncrementalClassLoader -> BootClassLoader）
* createRealApplication()
进行application 的相关替换，当前app的application变为realApplication；
反射的方式拿到 真实的 Application，通过AppInfo相关字段进行获取


下面我们分析一下setClassLoader详细过程：
主要经历了以下的方法：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20%E8%AE%BE%E7%BD%AE%E7%88%B6classloader%E8%BF%87%E7%A8%8B.png)
这几个ClassLoader类定义的逻辑关系如下：
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant-run%E7%9B%B8%E5%85%B3classLoader%E5%AE%9A%E4%B9%89.png)
findClass过程依次委托给 父ClassLoader，最后是让PathClassLoader去加载类

##### onCreate() 过程
* 通过MonkeyPatcher 替换当前的 application 为 realApplication
包含ActivityThread中相应的Application 都替换成 realApplication
* 替换相应的资源resource
替换当前app的assetManager，资源相关的变量等等（期间都是用反射的方式）
* Server 创建，建立Socket连接，开启连接

##### Server 部署工作
在Server 建立起连接后，三种部署工作（hot swap、warm swap、cold swap），都是通过Server进行操作。具体在那种情形下进行哪种交换，源码中有具体实现：

``` java
private int handlePatches(List<ApplicationPatch> paramList, boolean paramBoolean, int paramInt) {
    if (paramBoolean) {
        FileManager.startUpdate();
    }
    Iterator localIterator = paramList.iterator();
    while (localIterator.hasNext()) {
        Object localObject = (ApplicationPatch) localIterator.next();
        String str = ((ApplicationPatch) localObject).getPath();
        if (str.endsWith(".dex")) {// 冷交换
            handleColdSwapPatch((ApplicationPatch) localObject);
            int j = 0;
            localObject = paramList.iterator();
            do {
                i = j;
                if (!((Iterator) localObject).hasNext()) {
                    break;
                }
            }
            while (!((ApplicationPatch) ((Iterator) localObject).next()).getPath().equals("classes.dex.3"));
            int i = 1;
            if (i == 0) {
                paramInt = 3;
            }
        } else if (str.equals("classes.dex.3")) {// 热交换
            paramInt = handleHotSwapPatch(paramInt, (ApplicationPatch) localObject);
        } else if (isResourcePath(str)) {// 资源：温交换
            paramInt = handleResourcePatch(paramInt, (ApplicationPatch) localObject, str);
        }
    }
    if (paramBoolean) {
        FileManager.finishUpdate(true);
    }
    return paramInt;
}
```
### 代码热更新流程
在我们增加一行代码后，点击运行，我们来观察生成的类的变化
在 build 目录下，transforms 中有生成相关的代码
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant-run%E6%9C%89%E4%BB%A3%E7%A0%81%E5%8F%98%E6%9B%B4%E6%97%B6%E6%96%B0%E7%94%9F%E6%88%90.png)

#### 几个重要类
我们来具体看看demo 中代码更改：MainActivity$override类内容的确是最新的代码内容
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20%E5%A2%9E%E5%8A%A0%E7%9A%84%E4%BB%A3%E7%A0%81override%E7%B1%BB.png)

AppPatchesLoaderImpl类记录了更改的类，存储在一个数组中，供类加载时候，替换成最新的类的代码内容
![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%20%E5%8F%98%E5%8C%96%E7%9A%84%E7%B1%BB%E6%95%B0%E7%BB%84%E5%AD%98%E5%82%A8.png)

![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant-run%20patch%E7%9B%B8%E5%85%B3.png)

在此处，我反编译了slice_0-classes.dex：
 ![](https://canyifenglin-1258849639.cos.ap-beijing.myqcloud.com/blog/files/instant%20run%E7%AC%AC%E4%B8%80%E6%AC%A1%E6%89%93%E5%8C%85%E7%BB%93%E6%9E%9C.png)

第一次运行打包生成的 “业务代码” 中，生成的类中的方法里都增加了 IncrementalChange 相关的判断，如果 $change 不为空，说明我们有更改的代码，有更改的代码，则执行最新更改的代码

``` java
public Object access$dispatch(String paramString, Object... paramVarArgs) {
        switch (paramString.hashCode()) {
            case -833446436:
                initView((MainActivity) paramVarArgs[0]);
                return null;
            case -641568046:
                onCreate((MainActivity) paramVarArgs[0], (Bundle) paramVarArgs[1]);
                return null;
            case -399296056:
                return init$args((MainActivity[]) paramVarArgs[0], (Object[]) paramVarArgs[1]);
            case 781336394:
                init$body((MainActivity) paramVarArgs[0], (Object[]) paramVarArgs[1]);
                return null;
            case 2118315029:
                testClick((MainActivity) paramVarArgs[0], (View) paramVarArgs[1]);
                return null;
        }
        throw new InstantReloadException(String.format("String switch could not find '%s' with hashcode %s in %s", new Object[]{paramString, Integer.valueOf(paramString.hashCode()), "com/geng/myapplication/MainActivity"}));
    }
```
最后根据不同的类型，进行相关的重启（activity 或者 app），主要由 Restarter负责，同时也提供了相关的重启方法：
* restartActivity()
* restartApp()

<br>
至此，整个 instant run 的分析告一段落，需要慢慢消化一下。。。

<br>
源码及工具资源：
https://github.com/fenglincanyi/Study/tree/master/instant%20run%E7%9B%B8%E5%85%B3
参考：
https://github.com/nuptboyzhb/AndroidInstantRun
