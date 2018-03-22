---
title: 热修复之Robust浅析
date: 2018-3-12 18:28
categories: Android
tags: [hotfix, Android]
---
## 引言
热修复在前几年来说，是个比较热的词；
按大致的时序来排的话：比如阿里的andfix，qq空间的热修复方案（多dex插桩占位，未开源），Nuwa，腾讯tinker，饿了么Amigo，美团robust，阿里Sophix ... ...
各自的技术角度也是各有不同，也可以叫做流派不同。
从dex、class加载的角度，方法hook角度，代码插桩角度 ... ....
从之前我分析的Freeline增量编译来说，也是属于这个范畴，并且对于平时开发来说，是一个很不错的增量方案。
作为一个Android 开发者来说，这么多技术方案的实现，真的是很不错的一个学习机会。无论热修复的能力好或坏，是否成熟与稳定，都可以给我们提供各种解决问题的思路和角度，十分有借鉴意义。
Robust 是借鉴了 Google instant run 的思路，将其作为热修复实现方案来改进演变而成的，从修复能力、兼容性、稳定性来讲，也算是性价比较高的。
## 基本原理
Robust 的热修复思路，是在方法执行之前，判断是否需要热修复(flag)，如果是，则执行patch类的方法，如果不是，则执行原来的内容。
——这就是热修复所要做到的效果
具体实现呢，我们再来说：
根据以上的需求，主要就需要这2点：
1. 每个类中，需要一个字段来标识是否需要热修复
2. 在每个方法前插入判断逻辑，需要热修复，则走patch的新逻辑

## 主要的流程
### 编译期插桩
要做到插入代码，自然想到通过gradle plugin 来搞事情了，毕竟Google为我们已经提供好了接口（instant run 也是这么干的）
Robust 自定义 Gradle Plugin：**gradle-plugin**
在transform时期，加入自己的处理逻辑:
`RobustTransform.groovy#transform`
```groovy
// 拿到所有的类，放入到list中
def box = ConvertUtils.toCtClasses(inputs, classPool)
if(useASM){
    // 用ASM工具来直接修改 class
    insertcodeStrategy=new AsmInsertImpl(hotfixPackageList,hotfixMethodList,exceptPackageList,exceptMethodList,isHotfixMethodLevel,isExceptMethodLevel);
} else {
    insertcodeStrategy=new JavaAssistInsertImpl(hotfixPackageList,hotfixMethodList,exceptPackageList,exceptMethodList,isHotfixMethodLevel,isExceptMethodLevel);
}
// 插入代码，输出jar
insertcodeStrategy.insertCode(box, jarFile);
// 记录类的method信息，保存
writeMap2File(insertcodeStrategy.methodMap, Constants.METHOD_MAP_OUT_PATH)
```
主要的代码路径：
AsmInsertImpl.insertCode() ->
AsmInsertImpl.transformCode() ->
InsertMethodBodyAdapter.visitMethod() ->
// 插入static field & 方法体插入代码
classWriter.visitField() & 
MethodBodyInsertor.visitCode() -> RobustAsmUtils.createInsertCode()

这些操作执行结束后，所有的class内容已经完备了，那么放入产物main.jar中，等待proguard和转成dex
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/robust1.png" width="300" height="320" style="margin:0 auto;display:block"/>
另外记录操作过的method的信息（methodMap<methodName, id>), 存储字节流到methodMap.robust中,并gzip压缩
后续编译操作，都与正常的流程一样了

![](http://7xr1vo.com1.z0.glb.clouddn.com/robust2.png)
从这里也能看出来，robust的缺点就是各处都插入了代码，对包体积是有影响的
### 打patch包
打包之后产生的 mapping.txt, methodMap.robust,需要我们保存下来，用于打patch包
生成patch包，是在上次正常打包的代码基础上，也就是相同的代码条件下，来进行操作
自然的，patch打包，也是自动化通过gradle plugin来生成
Robust 自定义 Gradle Plugin：**auto-patch-plugin**
`AutoPatchTransform..groovy#transform`
```groovy
ReadAnnotation.readAnnotation(box, logger);
if(Config.supportProGuard) {
    ReadMapping.getInstance().initMappingInfo();
}
generatPatch(box,patchPatch()

zipPatchClassesFile()
executeCommand(jar2DexCommand)
executeCommand(dex2SmaliCommand)
SmaliTool.getInstance().dealObscureInSmali();
executeCommand(smali2DexCommand)
//package patch.dex to patch.jar
packagePatchDex2Jar()
```
扫描新增的类、方法、修改的方法，并分别添加到
newlyAddedClassNameList, newlyAddedMethodSet, modifiedClassNameList, patchMethodSignatureSet中

```groovy
boolean isNewlyAddClass = scanClassForAddClassAnnotation(ctclass);
//newly add class donnot need scann for modify
if (!isNewlyAddClass) {
    patchMethodSignureSet.addAll(scanClassForModifyMethod(ctclass));
    scanClassForAddMethodAnnotation(ctclass);
}
```
mapping读取：
* initConfig()时会读取methodMap.robust的数据并将method信息存入methodMap中
* initMappingInfo()读取mapping.txt, 并存入usedInModifiedClassMappingInfo中

生成patch之前，会预先处理内部类，内联的方法(InlineClassFactory.dealInLineClass);

准备工作已经就绪，开始生成patch操作：
PatchesFactory, PatchesControlFactory, PatchesInfoFactory开始进行复杂的patch相关类的生成;
另外，还有热修复的一些辅助类：createControlClass, createPatchesInfoClass

在此过程中，有些复杂的情况，需要做一些特殊的处理：
主要是2个因素的影响：编译器对类的编译和更改、混淆引起的问题，内联优化掉的内容等等
所以在 createPatchClass()时，操作也是比较复杂的,这也是auto-patch的难点......

之后，将所有生成的class压缩至jar(meituan.jar), 执行命令去 jar -> dex, dex -> smali
之所以这样做，是为了处理混淆的问题,目前生成的dex里的class都是未混淆过的类，字段，方法，所以这时候，
通过查询mapping缓存，去替换成上一次打包所对应的各种a,b,c;
这一步骤，可以参照中间产物 xxxPatch.class, xxxPatch.smali, mapping.txt 看出来。

SmaliTool.groovy 进行一行一行的混淆问题处理，类的调用关系，混淆映射，修改好最终的smali文件；
然后再从smali还原到dex,再达成最后的patch.jar

关于混淆处理的smali操作，由于本人能力有限，就不展开讲了... ...

### patch下发及加载
patch.jar 制作好之后，按道理来说，我们是需要一个热修复后台的。
在app启动之后，请求接口，下拉patch包, 放在app packagename 下目录，对patch包有一定的校验。
由前面的步骤，我们知道，下发的其实就是个 dex, 所以app启动后，是需要加载dex里的类
#### 类加载
```java
DexClassLoader classLoader = new DexClassLoader(patch.getTempPath(), context.getCacheDir().getAbsolutePath(),
     null, PatchExecutor.class.getClassLoader());
patch.delete(patch.getTempPath());

Class patchsInfoClass;
PatchesInfo patchesInfo = null;
try {
    patchsInfoClass = classLoader.loadClass(patch.getPatchesInfoImplClassFullName());
    patchesInfo = (PatchesInfo) patchsInfoClass.newInstance();
} catch (Throwable t) {
}
```
这时候借助 DexClassLoader将PatchesInfoImpl先的加载处来，读取patchsInfoClass，拿到PatchedClassInfo列表
#### 动态替换
PatchedClassInfo记录所有了修改的类，和PatchControl的对应关系，然后做下面的事情：
1. 加载要被修复的类patchedClassName，获取里面的字段changeQuickRedirect
2. 加载PatchControl，获取PatchControl的实例patchObject
3. 赋值：changeQuickRedirect=patchObject

这样，插桩的代码会检测到，并执行新逻辑
PatchProxy.isSupport() -> 
PatchProxy.accessDispatch() ->
PatchControl.accessDispatch() ->
XXXPatch.patchMethod()

此过程中，会将方法签名，参数，原来的类，方法id进行一系列的传递和校验，具体就不展开了。。。
## 坑点
* 对于匿名对象使用，有时候patch打包支持的不是很好

```java
new Thread() {
    @Override
    public void run() {
        // ......这里的内容，打patch时可能会丢失
    }
}.start();
```
可以这样解决：
new class:
```java
public class FixThread extends Thread {
    @overide
    public void run() {
        // ....
    }
}
```
调用的地方：
```java
new FixThread().start();
```
* 打包混淆配置中，最好关掉optimizationpasses, 开启的话，很容易出现changeQuickRedirect字段被优化掉的情况，而且，后续打patch包的gradle plugin 处理不是很好
* 内部类的构造方法是private（private会生成一个匿名的构造函数）时，需要在制作补丁过程中手动修改构造方法的访问域为public
* 对kotlin的兼容性没有Java稳定，因为编译指令可能有所改变，所以要谨慎

## 使用注意点
### 接入
这里只说一点，gradle 配置plugin时，可以动态控制,在local.properties中加入配置：
    robust_insert=fasle
    build_patch=fasle

`project/build.gradle`
```gradle
Properties localProperties = new Properties()
File localFile = project.rootProject.file("local.properties")
if (localFile.exists()) {
    localProperties.load(localFile.newDataInputStream())
    ext.robusInsert = !(localProperties.getProperty("robust_insert") == 'false')
    ext.buildPatch = localProperties.getProperty("build_patch") == "true"
} else {
    ext.robusInsert = true
    ext.buildPatch = false
}
```
`app/build.gradle`
```gradle
if (!robusInsert) {
    if (buildPatch) {
        println "===========  apply plugin: 'auto-patch-plugin'  ==========="
        apply plugin: 'auto-patch-plugin'
    }
    apply plugin: 'robust'
}
```
### 配置
对`robust.xml`按需配置
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <packname name="hotfixPackage">
        <name>com.meituan</name>
        <name>com.sankuai</name>
        <name>com.dianping</name>
    </packname>

    <!--不需要Robust插入代码的包名，Robust库不需要插入代码，如下的配置项请保留，还可以根据各个APP的情况执行添加-->
    <exceptPackname name="exceptPackage">
        <name>com.meituan.robust</name>
        <name>com.meituan.sample.extension</name>
    </exceptPackname>

    <!--补丁的包名，请保持和类PatchManipulateImp中fetchPatchList方法中设置的补丁类名保持一致（ setPatchesInfoImplClassFullName("com.meituan.robust.patch.PatchesInfoImpl")），
    各个App可以独立定制，需要确保的是setPatchesInfoImplClassFullName设置的包名是如下的配置项，类名必须是：PatchesInfoImpl-->
    <patchPackname name="patchPackname">
        <name>com.meituan.robust.patch</name>
    </patchPackname>
</resources>
```
### 统计、监控
* Robust都提供了修复成功相关统计接口，需要按照我们自己的需要来进行统计上报，后台跑数据，评估真实效果
* 对于patch和release包的版本控制，兼容处理
* 修复patch时的崩溃，需要加启动保护和降级处理，对于异常情况，需要实时监控

## 总结
* Robust 坑还是有的，单从稳定性和兼容性上来说，我们是可以比较信任的
* 热修复并不仅仅是一个框架，更是一个完整的修复系统，包括：后台版本控制，patch异常处理，降级处理；毕竟热修复对于app来说，是一个毕竟敏感的事情
* 各个主流的热修复技术，都涉及到Android底层的知识，所以对Android深入了解，对开发者十分有帮助


参考：
https://tech.meituan.com/android_autopatch.html
