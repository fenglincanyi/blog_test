---
title: Android SingleTask 探究
date: 2016-05-16 11:12
categories: Android 
tags: SingleTask
---
> Android 4种启动模式来说，用法说明此处不再提及主要介绍SingleTop，SingleTask相关的问题

### 说明

* 先分析 官方文档 中的一段话：

    > As shown in the table above, standard is the default mode and is appropriate for most types of activities. SingleTop is also a common and useful launch mode for many types of activities. The other modes — singleTask and singleInstance — <font color="red">are not appropriate for most applications</font>, since they result in an interaction model that is likely to be unfamiliar to users and is very different from most other applications.

&emsp;&emsp;对于大多数应用来说，SingleTask 和 SingleInstance 并不适用，standrd 和 SingleTop对于普通的大部分Activity启动是适用的。
之所以使用singleTask，是存在这样一类问题，想要从后面的Activity 直接跳转到前面某一个Activity时，可能会采用的，如一个应用的MainActivity，LoginActivity等。

&emsp;&emsp;对于SingleTask模式，官方文档的这么一句话，坑了不少人（我还比较幸运，没被坑惨）,也带给我对之前知识的迷惑。
> The system creates a new task and instantiates the activity at the root of the new task

&emsp;&emsp;**其实事实根本不是这样的**！！！

### 场景复现

* 下面用代码事实说话：建立3个Activity，分别为Main1Activity、Main2Activity、Main3Activity，前两个都设定单击事件，跳转逻辑为：main1 -> main2 -> main3，其中 Main2Activity 为 SingleTask（AndroidManifest.xml中设置）。打印相应的生命周期方法和所在的 **taskId**

    ![](http://7xr1vo.com1.z0.glb.clouddn.com/SingleTask%20%E5%9B%BE1.png)


&emsp;&emsp;这些taskid都是一样的，所以它们都是同一个Task中的。事实胜于雄辩。然后搜了一下相关问题，老罗的博客还是给力，彻彻底底的分析了这个坑。致谢老罗的开源精神！
&emsp;&emsp;链接：[http://blog.csdn.net/luoshengyang/article/details/6714543](http://blog.csdn.net/luoshengyang/article/details/6714543)

* 同时为了解释项目中类似的一个页面跳转问题，对于上面的demo做了修改：
    
    将 Main1Activity 设置为 SingleTask，其余2个为standrd
    
    操作步骤：
    1. 点击图标启动此应用
    2. 依次点击进入 Main2，Main3，再点击Home键
    3. 点击应用图标重新进入

    操作结果：结果再次显示的是Main1Activity，而不是Main3Activity。

    先看看打印的日志：
    
    ![](http://7xr1vo.com1.z0.glb.clouddn.com/SingleTask%E5%9B%BE2.png)
    
    再次点击图标进入应用时，实际上是Main2Activity，Main3Activity 出栈了。
    分析下这个过程：
    
    Main1，Main2，Main3 依次压入栈中，然后 Home 键，则整个Task处于stop状态，是一个background Task。当再次点击应用图标时，系统检测到此时已经存在一个该应用的Task的，此时就将此Background Task 移至前台，成为Foreground Task，而且由于Main1Activity是SingleTask，且位于task底部，所以，再次启动时，将前两个Activity移除，且按照启动顺序依次移除，所以打出的日志是：Main2 desotry，Main3 destory，Main1 onResume。
    
    * **看来，SingleTask这种特殊的模式引起的Task内Activity的变化是值得注意的**

    之前我在Stackoverflow上问相关的问题，和我猜想的原因比较类似，链接如下：
        
    [http://stackoverflow.com/questions/36933755/activity-is-singletask-and-is-root-in-task-restrart-activity-of-top-is-destorye](http://stackoverflow.com/questions/36933755/activity-is-singletask-and-is-root-in-task-restrart-activity-of-top-is-destorye) （可供参考）

那么，如何才能在一个新的任务栈里创建新的Activity呢？

* 只需要在AndroidManifest.xml中配置即可
    
    在SingleTask的基础之上，再增加设置 **taskAffinify** 属性即可，默认情况下，taskAffinity属性值为 包名，所以可以自定义一个taskAffinity值，便可以实现一个新的Task,新的Activity处于这个新的Task的root。如，我的 Demo 代码如下：
    
    ![](http://7xr1vo.com1.z0.glb.clouddn.com/SingleTask%E5%9B%BE3.png)

### 总结
* SingleTask的启动模式并不会启动一个新的任务栈来承载Activity，而是在原来的Task中
* SingleTask的启动模式，在官方文档的说明中，日常开发中并不建议使用，SingleInstance更是如此。
* Activity的启动模式会带来的Task的变化和Activity的生命周期变化都会在ActivityRecord中体现的，通过源码分析可以发现。
* 通过以下命令，可以查看Task内的Activity的变化：

        adb shell dumpsys activity  获取所有应用的activity堆栈信息
        adb shell dumpsys activity | grep com.xxx.xxx.xxx   获取某个应用的activity 堆栈信息
        adb shell dumpsys activity | grep mFocusedActivity  获取处于栈顶的Activity

* [http://blog.piasy.com/2016/03/19/Android-Task-And-Back-Stack/](http://blog.piasy.com/2016/03/19/Android-Task-And-Back-Stack/) (写完此文后，发现有高人已经写得很全面很详尽了)