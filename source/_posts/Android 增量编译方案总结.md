---
title: Android 增量编译方案总结
date: 2017-03-18 20:10
categories: Android
tags: [Freeline]
---

> 自从去年10月份，使用Freeline 感觉非常不错，开发效率提升数倍。由于工作原因，一直将这篇总结拖到现在。ok，现在好好总结下。。。

## 前言：Android 开发者之痛

普通的编译流程：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline1.png)
存在问题：
* 存在问题：
* 全量编译
* 没有缓存机制
* 单流程构建流程
* 代码和资源越来越多，项目编译越来越慢

现状：
* Windows: 3.5min
* Mac: 2.5min
    提高机器硬件已不能解决开发耗时严重的问题了...

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline2.jpeg)

如何解决？
1. 配置Gradle 参数、调整tasks间执行顺序、依赖
2. 可否实现增量编译
3. 组件化，独立module开发、维护

## 探索之路
### Buck 
—— Facebook

* 并发编译，建立多个并发子任务依赖关系，有向拓扑图，通过多线程并发把各个子节点构建出来，充分利用多核优势
* BUCK建立了一套完善的依赖规则以及细化的缓存系统来缩减编译时间

    增量构建的方式：以工程目录为单位进行增量构建，发生变更时候，变更的工程，以及该工程作为父节点或祖先节点的工程，均需要重新构建，构建完这些变更涉及的工程后，Buck需要重新走一次合并各工程DEX,对齐，签名，打包APK的过程,构建完毕后，继续走安装流程

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline3.gif)

缺点：
* 增量机制并不完善
* 引入工程量大，入侵性强
* Windows平台不支持

后传：
**OKBuck**
    OkBuck 的目标，是通过读取工程的 Gradle 配置，自动生成 BUCK 脚本，免去开发者下载依赖的 jar/aar 文件，编写、维护 BUCK 脚本、处理依赖之间的冲突等繁琐又容易出错的工作。

### LayoutCast 
——屠毅敏（AndroidDynamicLoader ）

资源文件更改：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline4.gif)

代码变动：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline5.gif)

编译速度对比：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline6.png)

**原理实现**：
* 利用反射，将修改的patch dex 插入到 dex Elements[] 最前面 
* 资源修改：通过运行时反射,拿到 R.class 字段，得出 ids.xml 和 pubilc.xml（ids: 我们定义的 view 的 id，public：包含ids的信息，及layout、drawable、color、string、dimen、style、attr）

 
``` java
public class ArtUtils {

    public static boolean overrideClassLoader(ClassLoader cl, File dex, File opt) {
        try {
            ClassLoader bootstrap = cl.getParent();
            Field fPathList = BaseDexClassLoader.class.getDeclaredField("pathList");
            fPathList.setAccessible(true);
            Object pathList = fPathList.get(cl);
            Class cDexPathList = bootstrap.loadClass("dalvik.system.DexPathList");
            Field fDexElements = cDexPathList.getDeclaredField("dexElements");
            fDexElements.setAccessible(true);
            Object dexElements = fDexElements.get(pathList);
            DexClassLoader cl2 = new DexClassLoader(dex.getAbsolutePath(), opt.getAbsolutePath(), null, bootstrap);
            Object pathList2 = fPathList.get(cl2);
            Object dexElements2 = fDexElements.get(pathList2);
            Object element2 = Array.get(dexElements2, 0);
            int n = Array.getLength(dexElements) + 1;
            Object newDexElements = Array.newInstance(fDexElements.getType().getComponentType(), n);
            Array.set(newDexElements, 0, element2); // 插入到数组最前面
            for (int i = 0; i < n - 1; i++) {
                Object element = Array.get(dexElements, i);
                Array.set(newDexElements, i + 1, element);
                // 其余 dex 元素依次后移
            }
            fDexElements.set(pathList, newDexElements);
            return true;
        } catch (Exception e) {
            Log.e("lcast", "fail to override classloader " + cl + " with " + dex, e);
            return false;
        }
    }
}
```

缺点：
* 当前的修改，会把之前的修改一起带进来，一起增量，修改次数多时，速度也会越来越慢（只是针对第一次build后的基线包做的增量修改，修改多次会带上很多的增量文件）
* 资源修改是利用反射，项目中资源越来越多时，速度提升并不明显
* 不支持Android 5.0以下的设备

### Instant-Run

原理实现
* 第一次编译时，在transform 时通过ASM对每一个方法加入 局部变量 change，更改代码后，会将 更改的类 加上 $override ，将最新类 push 到 手机上。就是通过 hack method 的方式来实现动态代码替换的
* 资源的修改更新，通过反射的方式，生成一个 AssetManager，调用 相关外界加载 资源的方法，将最新的资源包加载进来（全量的包），然后修剪删除缓存，刷新UI使之生效

缺点：
* 资源文件仍然是一个全量的过程，资源文件越大，速度并没有明显提升
* 无法debug，因为是 方法的hack，无法追到 堆栈信息
* 只支持Android5.0以上

### JRebel for Android

https://zeroturnaround.com/software/jrebel-for-android/features/

缺点
* 收费
* Crash 后需要重新全量编译，第一次编译很慢，亲测结果

## 神器Freeline——集百家之长

### 简介
* Freeline是蚂蚁金服旗下一站式理财平台蚂蚁聚宝团队在Android平台上的量身定做的一个基于动态替换的编译方案
* Freeline 借鉴了layoutCast、buck, instant run 的思想和方法，在其他增量编译方案上做了各种优化和性能的提升

### Freeline 整体工作流程

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline7.png)

<br>
* PC端与手机建立TCP长连接
* 扫描各个子工程文件变化
* 各个子工程的增量dex构建、增量资源包构建
* 合并所有工程dex
* 传输增量包
* App 更新代码或资源，刷新或重启

### 单个工程流程
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline8.png)

#### 几个重要的模块
##### Python 实现任务调度（调度中心、发号施令）

build_commands.py   builder.py  
各种命令, 各种构建

freeline_build.py  gradle_clean_build.py  gradle_inc_build.py
task拓扑序列构建,

task_engine.py
任务并发执行，依赖的模块在进行构建时，当前task.wait()，当其依赖执行结束，再执行此task

android_tools.py
建立连接、安装apk、一些辅助类

gradle_tools.py
Gradle执行的辅助类，扫描各个文件(GradleScanChangedFilesCommand)、资源，是否有变化、存储信息

sync_client.py
将代码、资源同步到手机


##### Gradle-Plugin 负责构建任务、代码注入等

注意：对于低版本的gradle插件，则不能使用 transform 时来进行字节码修改，要通过 preDex 这个 task 进行字节码的修改

``` groovy
if (!it.moduleVersion.startsWith("1.5")
        && !it.moduleVersion.startsWith("2")) {
    isLowerVersion = true
    return false
}
```
<br>
##### Freeline-runtime  主要处理 设备连接，增量代码、资源的更新
具体参考源码查看，此处不再贴


#### 代码增量实现
使用 Qzone 的思路进行实现：
DexUtils:
    分别对 4.0 以上和以下的做兼容，具体看代码：
   

``` java
try {
   Object newDexElements;
   int dexLength;
   if (VERSION.SDK_INT >= 14) {
       pathListField = ReflectUtil.fieldGetOrg(classLoader, Class.forName("dalvik.system.BaseDexClassLoader"), "pathList");
       fDexElements = ReflectUtil.fieldGetOrg(pathListField.get(classLoader), "dexElements");
       Object e = fDexElements.get(pathListField.get(classLoader));
       dstObject = e;
       dexFiles = new DexFile[Array.getLength(e)];
       for (int i = 0; i < Array.getLength(e); ++i) {
           newDexElements = Array.get(e, i);
           dexFiles[i] = (DexFile) ReflectUtil.fieldGet(newDexElements, "dexFile");
       }
   } else {
       pathListField = ReflectUtil.fieldGetOrg(classLoader, "mDexs");
       dstObject = pathListField.get(classLoader);
       dexFiles = new DexFile[Array.getLength(dstObject)];
       for (dexLength = 0; dexLength < Array.getLength(dstObject); ++dexLength) {
           dexFiles[dexLength] = (DexFile) Array.get(dstObject, dexLength);
       }
   }
   dexLength = Array.getLength(dstObject) + 1;
   newDexElements = Array.newInstance(fDexElements.getType().getComponentType(), dexLength);

   DexClassLoader dynamicDex = new DexClassLoader(dex.getAbsolutePath(), opt.getAbsolutePath(), null, classLoader.getParent());
   Log.i(TAG, "after opt, dex len:" + dex.length() + "; opt len:" + opt.length());
   Object pathList = pathListField.get(dynamicDex);
   Object dexElements = fDexElements.get(pathList);
   Object firstDexElement = Array.get(dexElements, 0);
   Array.set(newDexElements, 0, firstDexElement);

   for (int i = 0; i < dexLength - 1; ++i) {
       Object element = Array.get(dstObject, i);
       Array.set(newDexElements, i + 1, element);
   }

   if (VERSION.SDK_INT >= 14) {
       fDexElements.set(pathListField.get(classLoader), newDexElements);
   } else {
       pathListField.set(classLoader, newDexElements);
   }
   return true;
} catch (Exception e) {
   Log.e(TAG, "fail to override classloader " + classLoader + " with " + dex.getAbsolutePath(), e);
   return false;
}
```

**Preverify 过程：**
dex2opt过程中，若发现当前类中，存在一个直接引用类也和当前类在同一个dex中，则当前类会被打上 verified=true 的标记。下次加载时，则会判断这个类所在的dex是否是同一个dex

**如何避免 preverify 异常？**
在 dexopt 过程中，Class_isPreverfied 问题：
通过在每个类的构造方式，加入一个 另一个dex的 类，让其preverify 失效，这样就可以让增量的class被加载了
这也是Google的一个安全策略

相关知识可参考：
直接引用类的定义：
https://zhuanlan.zhihu.com/p/20308548
dex分包过程，dexopt介绍：
https://segmentfault.com/a/1190000004053072


#### 资源的更新逻辑
根据最新的 R.java 文件 拿到 各个资源id  生成 public.xml 和 ids.xml，用于解决资源id 冲突    id-gen-tool 工具   
 ![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline9.png)

对于增量的 资源进行 Freelineaapt 编译，未做过更改的资源，直接使用 backup 中的资源，再打成一个 增量包：inc.pack

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline10.png)

增量包中，只包含 增量的资源，全量的arsc 和 AndroidManifest.xml

Resources.arsc 并不一定会打进 pack 中，只有 资源的更改引起 arsc 变化时，才打入包中，arsc 的体积也是占一定比例的

手机端，资源更新生效：
    通过借鉴 instant run的方式，找到 resDir 的路径，将pack解压覆盖至该目录，然后 pruneResourceCaches,刷新UI

## Freeline 使用及相关问题 
不再浪费篇幅，直接贴出我总结的内容：

https://github.com/fenglincanyi/Study/blob/master/Freeline%E7%9B%B8%E5%85%B3/Freeline_use.md

## 实例分析
> 我们重点关注 项目目录下的 **app/build/freeline**

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline11.png)

更改代码后，再执行增量编译后，观察此目录：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline12.png)

只生成了最新的更改过的文件
反编译 dex 目录下的 classes.dex 得出, 此文件里全是更改过的类文件：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline13.png)

换个目录看看：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline14.png)

对 hackload.dex 反编译：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline15.png)

发现这个 hackload.dex 就是插桩时候，使用的单独的dex，里面是用来避免 preverify 问题的类
随后，我们看看编译后的apk 文件，解压缩，反编译看看任意一个classes.dex:
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline16.png)
发现，确实是在构造方法里插入一个其他dex中的类，来避免打上 verified 的标识，验证了上面的做法。
<br>
好，松一口气，代码更新算是说完了...

<hr/>


下面，我们看看资源是怎么更新的...

前面说过：
是通过反向对 R.java 文件摘出 资源 id信息，放在 ids.xml 和 public.xml 中，那我们来看看这两个文件：
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline17.png)

**ids.xml:**
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline18.png)

里面是压缩过的id信息，就是我们在写布局文件的那些内容

**public.xml:**
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline19.png)

此处只是部分，public.xml 文件记录了 name 和 id 之间的映射

一个普通的 R.java 文件，包含了以下资源信息
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline20.png)
所以，这里的 ids.xml 只存储了 我们自己写的 id 信息，而 public.xml 存储了 R.java 完整信息。
ids.xml 是为了处理资源id冲突问题，预先准备的文件

**再来看看 增量资源相关的：**

app.pack 文件压缩了所有的资源文件，包括assets 和 res 目录下的资源文件，和清单文件，资源索引表

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline21.png)

解压 app.pack 文件后：

![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline22.png)

扩展一下，看看 resource.arsc 文件：
此文件的并不是APP一下子解析加载的，是按需加载，
它是一种二进制索引表,对应了app里所有资源name,及id
![](http://7xr1vo.com1.z0.glb.clouddn.com/freeline23.png)


## 总结
* 整个 Freeline 项目的源码很值得研究，里面有好多的实现思路和解决方法都是精益求精的。把增量和优化做到极致。
* 充分的借鉴了 layoucast 的代码更新思路，Buck 的并发构建、有向拓扑的规则、instant run的字节码修改、monkeyPatcher的实现方法，另外借助 gradle transform 插件及 preDex task时机进行字节码修改，真的是集百家之长。
**工作启示：**
* 解决方法永远会有更优的，只是你暂时没找到
* 复杂的工程都是一点一点做出来的


## 部分源码注释
阅读 freeline 的 python模块代码的注释：
https://github.com/fenglincanyi/Study/tree/master/Freeline%E7%9B%B8%E5%85%B3/freeline%E7%9B%B8%E5%85%B3%E6%BA%90%E7%A0%81%E6%B3%A8%E9%87%8A/freeline/freeline/freeline


## 补充点
想说的太多了，自己写个配注。。。

* Gradle plugin 模块：
负责构建时，做的一些逻辑
如：项目描述文件（FreelineInitializer.groovy 执行初始化时，生成项目描述文件）

* reelineInjector.groovy 里的  hackClass  -> new FreelineClassVisitor -> 进行字节码的注入
这个思路借鉴了instant run的做法

* Freeline-runtime/DexUtils
处理dex增量包的逻辑

* Instant-run-serve 项目下的 monkeyPatcher 被Freeline直接复用了

* Gradle transform plugin
transform 是在java文件编译成class之后，合成dex之前，此期间执行的，来修改class内容


transform 解释：
http://blog.csdn.net/sbsujjbcy/article/details/50839263
源码位置：
https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/


instant run 中涉及到的类：用到了ASM
IncrementalChangeVisitor.java
IncrementalSupportVisitor.java
IncrementalVisitor.java


使用到，调用 task 命令，传入参数：
``` python
command += ' -P freelineBuild=true' # 使用 gradle 方式，加入了freelineBuild 属性 ，在 FreelinePlugin.groovy 中有体现
```
