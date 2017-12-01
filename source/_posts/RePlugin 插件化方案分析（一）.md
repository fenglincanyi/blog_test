---
title: RePlugin插件化方案分析（一）
date: 2017-10-30 23:45
categories: 插件化
tags: 插件化
---

## 前言

> RePlugin 是今年360技术团队GMTC大会公布的插件化方案。360对插件化的技术探索及优化，在近几年是相当有技术沉淀的。从 DroidPlugin 到 RePlugin, 确实有值得学习的地方。
本文先来简单分析下 RePlugin 的基本原理。

## 一、引子 
普通的:  Application extends ContextWrapper, 其中 attachBaseContext() 是从 ContextWrapper 继承过来的，在 Application 的 attach()  中  attachBaseContext() 会被调用:

```java
/* package */ final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}
```
Application 的 attachBaseContext() 方法 在 attach时被调用。那么 这个 attach() 方法是什么时机被调用呢？在 Instrumentation 类中，找到答案：

```java
static public Application newApplication(Class<?> clazz, Context context)  {
    //实例化Application
    Application app = (Application)clazz.newInstance();

    app.attach(context); // 这个传进来的 context 是 makeApplication 时创建的 ContextImpl
    return app;
}
```
创建 Application 时，会将 Context attach 到 Application 上

另外，后面对于 类加载 知识也是必要的，再做一次复习。

可参考之前写的文章：
https://fenglincanyi.github.io/2016/11/17/Android%20%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%88%9D%E6%8E%A2/


## 二、Hook时机及Hook点
RePlugin 从 Application 就开始下手了, RePlugin 的 hook 时机：
在 application 的 attachBaseContext 方法中 初始化 Replugin 时（不管是哪种接入方式，都会调用）：
```java
RePlugin.App.attachBaseContext(this);
```
在 RePlugin.java 中：
```java
public static void attachBaseContext(Application app, RePluginConfig config) {
    if(sAttached) {
        if(LogDebug.LOG) {
            LogDebug.d("RePlugin", "attachBaseContext: Already called");
        }

    } else {
        RePluginInternal.init(app);
        RePlugin.sConfig = config;
        RePlugin.sConfig.initDefaults(app);
        IPC.init(app);
        if(LogDebug.LOG && RePlugin.sConfig.isPrintDetailLog()) {
            LogDebug.printMemoryStatus("RePlugin", "act=, init, flag=, Start, pn=, framework, func=, attachBaseContext, lib=, RePlugin");
        }

        HostConfigHelper.init();
        AppVar.sAppContext = app;
        PluginStatusController.setAppContext(app);
        PMF.init(app);// 此处是 hook classloader 入口
        PMF.callAttach();
        sAttached = true;
    }
}
```
PMF.java 中：
```java
public static final void init(Application application) {
    setApplicationContext(application);
    PluginManager.init(application);
    sPluginMgr = new PmBase(application);
    sPluginMgr.init();
    Factory.sPluginManager = getLocal();
    Factory2.sPLProxy = getInternal();
    PatchClassLoaderUtils.patch(application);// 关键点
}
```
在 PatchClassLoaderUtils.java 中，进行 hook 操作； 所以, 是在 attachBaseContext() 时 做的 classloader 的 hook。
我们具体来看看：
```java
public static boolean patch(Application application) {
    try {
        // 获取Application的BaseContext （来自ContextWrapper）
        Context oBase = application.getBaseContext();
        if (oBase == null) {
            if (LOGR) {
                LogRelease.e(PLUGIN_TAG, "pclu.p: nf mb. ap cl=" + application.getClass());
            }
            return false;
        }

        // 获取mBase.mPackageInfo
        // 1. ApplicationContext - Android 2.1
        // 2. ContextImpl - Android 2.2 and higher
        // 3. AppContextImpl - Android 2.2 and higher
        Object oPackageInfo = ReflectUtils.readField(oBase, "mPackageInfo");
        if (oPackageInfo == null) {
            if (LOGR) {
                LogRelease.e(PLUGIN_TAG, "pclu.p: nf mpi. mb cl=" + oBase.getClass());
            }
            return false;
        }

        // mPackageInfo的类型主要有两种：
        // 1. android.app.ActivityThread$PackageInfo - Android 2.1 - 2.3
        // 2. android.app.LoadedApk - Android 2.3.3 and higher
        if (LOG) {
            Log.d(TAG, "patch: mBase cl=" + oBase.getClass() + "; mPackageInfo cl=" + oPackageInfo.getClass());
        }

        // 获取mPackageInfo.mClassLoader
        ClassLoader oClassLoader = (ClassLoader) ReflectUtils.readField(oPackageInfo, "mClassLoader");
        if (oClassLoader == null) {
            if (LOGR) {
                LogRelease.e(PLUGIN_TAG, "pclu.p: nf mpi. mb cl=" + oBase.getClass() + "; mpi cl=" + oPackageInfo.getClass());
            }
            return false;
        }

        // 外界可自定义ClassLoader的实现，但一定要基于RePluginClassLoader类
        ClassLoader cl = RePlugin.getConfig().getCallbacks().createClassLoader(oClassLoader.getParent(), oClassLoader);

        // 将新的ClassLoader写入mPackageInfo.mClassLoader
        ReflectUtils.writeField(oPackageInfo, "mClassLoader", cl);

        // 设置线程上下文中的ClassLoader为RePluginClassLoader
        // 防止在个别Java库用到了Thread.currentThread().getContextClassLoader()时，“用了原来的PathClassLoader”，或为空指针
        Thread.currentThread().setContextClassLoader(cl);

        if (LOG) {
            Log.d(TAG, "patch: patch mClassLoader ok");
        }
    } catch (Throwable e) {
        e.printStackTrace();
        return false;
    }
    return true;
}
```
这里, 从 baseContext （makeApplication 时创建的 ContextImpl）获取到  mPackageInfo （是 LoadedApk 的实例）, 进而 再获取到 mPackageInfo的私有属性 mClassLoader.   拿到 classLoader 了，就可以进行 classLoader 的替换工作了.  

## 三、Hook实现过程
先不直接说怎么做的，我们先看看 这个 mClassLoader 是怎么来的？
在 LoadedApk.Java 中，构造方法里：
```java
mClassLoader = ClassLoader.getSystemClassLoader();
```
继续跟进， 
```java
public static ClassLoader getSystemClassLoader() {
    return SystemClassLoader.loader;
}
```
其中：SystemClassLoader  是  抽象类ClassLoader 的一个静态类：
```java
static private class SystemClassLoader {
    public static ClassLoader loader = ClassLoader.createSystemClassLoader();
}
```
此时，就看到了熟悉的 PatchClassLoader：
```java
private static ClassLoader createSystemClassLoader() {
    String classPath = System.getProperty("java.class.path", ".");
    String librarySearchPath = System.getProperty("java.library.path", "");

    // String[] paths = classPath.split(":");
    // URL[] urls = new URL[paths.length];
    // for (int i = 0; i < paths.length; i++) {
    // try {
    // urls[i] = new URL("file://" + paths[i]);
    // }
    // catch (Exception ex) {
    // ex.printStackTrace();
    // }
    // }
    //
    // return new java.net.URLClassLoader(urls, null);

    // TODO Make this a java.net.URLClassLoader once we have those?
    return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance());
}
```
我们再次回到 上面的 classLoader 替换步骤中， 那么 RepluginClassLoader  就  替换了  PathClassLoader.  后面的 类加载 过程 就依赖于  RePluginClassLoader 来工作了。
其中，oClassLoader 被赋值，就是 原始的 PathClassLoader

下个阶段：RePluginClassLoader 的 loadClass 过程的更改：

首先，原始的ClassLoader 的 loadClass 过程：（路径： Android/sdk/platforms/android-25/android.jar!/java/lang/ClassLoader.class）
```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
        // First, check if the class has already been loaded   检查class是否加载过, 加载过则直接 return
        Class c = findLoadedClass(name);
        if (c == null) {// 未加载过，则进行 下面的 加载过程：
            long t0 = System.nanoTime();//  获取 纳秒
            try {
                // 双亲委托，机制进行 class 的 load 工作：
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name); // 此方法内部，直接返回 null
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.    如果仍然找不到，就按顺序 查找 class
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
            }
        }
        return c;
}
```
下面是  RePluginClassLoader 的 实现：
```java
@Override
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    //
    Class<?> c = null;
    c = PMF.loadClass(className, resolve);// 先去容器中查找并加载预埋的activity 或其他类
    if (c != null) {
        return c;// 在 插件中 查找成功，则直接返回
    }
    //
    try {
        // 如果没有在 插件 中找到，则使用  mOrig  这个 classLoader 去加载
        //  mOrig 是谁呢？可以在 PatchClassLoaderUtils 中找到, 发现就是 原始的PathClassLoader，即 apk 刚被 加载的用的 classloader
        //  mOrig 就是 宿主host 的 classLoader 
        c = mOrig.loadClass(className);
        // 只有开启“详细日志”才会输出，防止“刷屏”现象
        if (LogDebug.LOG && RePlugin.getConfig().isPrintDetailLog()) {
            LogDebug.d(TAG, "loadClass: load other class, cn=" + className);
        }
        return c;
    } catch (Throwable e) {
        //
    }
    //
    return super.loadClass(className, resolve);// 极端情况，上面的方式全都加载失败，默认使用 BaseDexClassLoader 去加载
}
```
之后，PmBase 的 loadClass 过程：
```java
final Class<?> loadClass(String className, boolean resolve) {
    // 加载Service中介坑位
    if (className.startsWith(PluginPitService.class.getName())) {
        if (LOG) {
            LogDebug.i(TAG, "loadClass: Loading PitService Class... clz=" + className);
        }
        return PluginPitService.class;
    }

    //
    if (mContainerActivities.contains(className)) {
        Class<?> c = mClient.resolveActivityClass(className);
        if (c != null) {
            return c;
        }
        // 输出warn日志便于查看
        // use DummyActivity orig=
        if (LOGR) {
            LogRelease.w(PLUGIN_TAG, "p m hlc u d a o " + className);
        }
        return DummyActivity.class;
    }

    //
    if (mContainerServices.contains(className)) {
        Class<?> c = loadServiceClass(className);
        if (c != null) {
            return c;
        }
        // 输出warn日志便于查看
        // use DummyService orig=
        if (LOGR) {
            LogRelease.w(PLUGIN_TAG, "p m hlc u d s o " + className);
        }
        return DummyService.class;
    }

    //
    if (mContainerProviders.contains(className)) {
        Class<?> c = loadProviderClass(className);
        if (c != null) {
            return c;
        }
        // 输出warn日志便于查看
        // use DummyProvider orig=
        if (LOGR) {
            LogRelease.w(PLUGIN_TAG, "p m hlc u d p o " + className);
        }
        return DummyProvider.class;
    }

    // 插件定制表
    DynamicClass dc = mDynamicClasses.get(className);
    if (dc != null) {
        final Context context = RePluginInternal.getAppContext();
        PluginDesc desc = PluginDesc.get(dc.plugin);

        if (LOG) {
            LogDebug.d("loadClass", "desc=" + desc);
            if (desc != null) {
                LogDebug.d("loadClass", "desc.isLarge()=" + desc.isLarge());
            }
            LogDebug.d("loadClass", "RePlugin.isPluginDexExtracted(" + dc.plugin + ") = " + RePlugin.isPluginDexExtracted(dc.plugin));
        }

        // 加载动态类时，如果其对应的插件未下载，则转到代理类
        if (desc != null) {
            String plugin = desc.getPluginName();
            if (PluginTable.getPluginInfo(plugin) == null) {
                if (LOG) {
                    LogDebug.d("loadClass", "plugin=" + plugin + " not found, return DynamicClassProxyActivity.class");
                }
                return DynamicClassProxyActivity.class;
            }
        }

        /* 加载未安装的大插件时，启动一个过度 Activity */
        // todo fixme 仅对 activity 类型才弹窗
        boolean needStartLoadingActivity = (desc != null && desc.isLarge() && !RePlugin.isPluginDexExtracted(dc.plugin));
        if (LOG) {
            LogDebug.d("loadClass", "needStartLoadingActivity = " + needStartLoadingActivity);
        }
        if (needStartLoadingActivity) {
            Intent intent = new Intent();
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            // fixme 将 PluginLoadingActivity2 移到 replugin 中来，不写死
            intent.setComponent(new ComponentName(IPC.getPackageName(), "com.qihoo360.loader2.updater.PluginLoadingActivity2"));
            context.startActivity(intent);
        }

        Plugin p = loadAppPlugin(dc.plugin);
        if (LOG) {
            LogDebug.d("loadClass", "p=" + p);
        }
        if (p != null) {
            try {
                Class<?> cls = p.getClassLoader().loadClass(dc.className);
                if (needStartLoadingActivity) {
                    // 发广播给过度 Activity，让其关闭
                    // fixme 发送给 UI 进程
                    Tasks.postDelayed2Thread(new Runnable() {
                        @Override
                        public void run() {
                            if (LOG) {
                                LogDebug.d("loadClass", "发广播，让 PluginLoadingActivity2 消失");
                            }
                            IPC.sendLocalBroadcast2All(context, new Intent("com.qihoo360.replugin.load_large_plugin.dismiss_dlg"));
                        }
                    }, 300);
                    // IPC.sendLocalBroadcast2Process(context, IPC.getPersistentProcessName(), new Intent("com.qihoo360.replugin.load_large_plugin.dismiss_dlg"), )
                }
                return cls;
            } catch (Throwable e) {
                if (LOGR) {
                    LogRelease.w(PLUGIN_TAG, "p m hlc dc " + className, e);
                }
            }
        } else {
            if (LOG) {
                LogDebug.d("loadClass", "加载 " + dc.plugin + " 失败");
            }
            Tasks.postDelayed2Thread(new Runnable() {
                @Override
                public void run() {
                    IPC.sendLocalBroadcast2All(context, new Intent("com.qihoo360.replugin.load_large_plugin.dismiss_dlg"));
                }
            }, 300);
        }
        if (LOGR) {
            LogRelease.w(PLUGIN_TAG, "p m hlc dc failed: " + className + " t=" + dc.className + " tp=" + dc.classType + " df=" + dc.defClass);
        }
        // return dummy class
        if ("activity".equals(dc.classType)) {
            return DummyActivity.class;
        } else if ("service".equals(dc.classType)) {
            return DummyService.class;
        } else if ("provider".equals(dc.classType)) {
            return DummyProvider.class;
        }
        return dc.defClass;
    }

    //
    return loadDefaultClass(className);
}
```


## 附录
相关资料：

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1223/2205.html
http://www.jianshu.com/p/9c96b68f5ee6
https://github.com/Qihoo360/RePlugin/tree/dev/replugin-host-library
https://fenglincanyi.github.io/2016/11/17/Android%20%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%88%9D%E6%8E%A2/
