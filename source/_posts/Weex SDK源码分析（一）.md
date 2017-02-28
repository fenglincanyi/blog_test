---
title: Weex SDK源码分析（一）
date: 2017-2-28 15:06
categories: Weex
tags: [Weex, Android]
---
> **WeexSDK 初始化源码分析**

整个Weex SDK 初始化过程，入口是WXSDKEngine.initialize()，方法主要是依靠 doInitInternal() 方法执行初始化操作。里面涉及了WXEnvironment 相关环境的设置，几个重要的Manager的初始化操作，包括WXBridgeManager、WXSDKManager、WXRenderManager、WXDomManager，前两个 manager 以单例形式呈现的。
初始化完相关管理类，后面进行 component、module的注册。

下面对这一过程，进行一一分析：

WXSDKEngine.doInitInternal() 方法：
``` java
private static void doInitInternal(final Application application, final InitConfig config) {
        WXEnvironment.sApplication = application;
        WXEnvironment.JsFrameworkInit = false;

        WXBridgeManager.getInstance().post(new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                WXSDKManager sm = WXSDKManager.getInstance();
                if (config != null) {
                    sm.setInitConfig(config);
                    if (config.getDebugAdapter() != null) {
                        config.getDebugAdapter().initDebug(application);// 关于weex debug操作，不是此篇关注的重点，略过
                    }
                }
                WXSoInstallMgrSdk.init(application);
                boolean isSoInitSuccess = WXSoInstallMgrSdk.initSo(V8_SO_NAME, 1, config != null ? config.getUtAdapter() : null);
                if (!isSoInitSuccess) {
                    return;
                }
                sm.initScriptsFramework(config != null ? config.getFramework() : null);

                WXEnvironment.sSDKInitExecuteTime = System.currentTimeMillis() - start;
                WXLogUtils.renderPerformanceLog("SDKInitExecuteTime", WXEnvironment.sSDKInitExecuteTime);
            }
        });
        register();
    }
```

## WXSDKManager的初始化
首先，是WXSDKManager的初始化，会将WXRenderManager、WXDomManager初始化，并拿到WXBridgeManager实例的引用。

* 关于 WXRenderManager
管理渲染操作，主要操作管理的是WXRenderStatement对象（WXRenderManager 也并不是一个线程安全的类，涉及到UI的更新操作。后面会对WXRenderStatement 进行分析）
WXRenderManager 主要的2个成员：mRegistries 和 mWXRenderHandler：
``` java
private ConcurrentHashMap<String, WXRenderStatement> mRegistries;
private WXRenderHandler mWXRenderHandler;
```
mRegistries存储WXRenderStatement，以 WXSDKInstance.id 为key存储WXRenderStatement，所以一个WXSDKInstance对应一个WXRenderStatement。
WXRenderManager 的 createBody、addComponent等等操作都是针对某个WXSDKInstance的 statement 调用操作的。具体的渲染，WXRenderStatement负责完成。
mWXRenderHandler 是将外界传给它的 渲染相关的task 发送消息，然后主线程收到messge后，进行相关渲染操作。

* 关于 WXDomManager
管理dom操作，作为客户端执行dom命令，会调用 WXDomStatement 创建命令执行相对应的操作。里面提供的方法通常是在 dom 线程中调用。
其中有：
``` java
 private WXThread mDomThread;
  /** package **/
  Handler mDomHandler;
  private WXRenderManager mWXRenderManager;
  private ConcurrentHashMap<String, WXDomStatement> mDomRegistries;
```
mDomRegistries 和上面类似，大家一看就懂，<WXSDKInstance.id，WXDomStatement>。
mWXRenderManager 是拿到上一步初始化好的 mWXRenderManager实例。
mDomThread是WXDomManager创建的一个thread，也是WXThread，其中的handler是WXDomHandler
mDomHandler 是这个mDomThread的handler引用。
WXDomHandler 是关于dom操作的一个类，实现了Handler.Callback接口，会将mDomHandler发送的dom消息分类处理，这一任务是交给了 WXDomManager：

``` java
    @Override
    public boolean handleMessage(Message msg) {
        if (msg == null) {
            return false;
        }
        int what = msg.what;
        Object obj = msg.obj;
        WXDomTask task = null;

        if (obj instanceof WXDomTask) {
            task = (WXDomTask) obj;
        }

        if (!mHasBatch) {
            mHasBatch = true;
            mWXDomManager.sendEmptyMessageDelayed(WXDomHandler.MsgType.WX_DOM_BATCH, DELAY_TIME);
        }
        switch (what) {
            case MsgType.WX_DOM_CREATE_BODY:
                mWXDomManager.createBody(task.instanceId, (JSONObject) task.args.get(0));
                break;
            case MsgType.WX_DOM_UPDATE_ATTRS:
                mWXDomManager.updateAttrs(task.instanceId, (String) task.args.get(0), (JSONObject) task.args.get(1));
                break;
            case MsgType.WX_DOM_UPDATE_STYLE:
                mWXDomManager.updateStyle(
                        task.instanceId,
                        (String) task.args.get(0),
                        (JSONObject) task.args.get(1),
                        task.args.size() > 2 && (boolean) task.args.get(2)
                );
                break;
            case ...
            case ...
```
具体的manager操作，会交给对应的 domstatement 操作，上面已经说过。



WXSoInstallMgrSdk 是管理 so 相关操作和cpu平台的支持情况。
WXSoInstallMgrSdk.initSo() 执行 weexv8.so 包的加载，就和普通加载so包的方式一样（System.loadLibrary(libName) ）。
> 这里注意：weex 的 so 包不支持 mips平台的

如果so包加载失败，则初始化操作会就此结束，后续工作不再执行。
so包加载完成之后，WXSDKManager会调用WXBridgeManager执行 js Framework的初始化，并发送消息给
WXBridgeManager，WXBridgeManager调用自己的 handleMessage() 处理操作，下面会有分析。

## WXBridgeManager 的初始化
其次，WXBridgeManager 的初始化。会创建名为：WeexJSBridgeThread的 WXThread，WXThread是一个weex封装的HandlerThread。为了方便该线程的消息处理，具体细节可参考源码查看。WXBridgeManager 自己也实现了 Handler.Callback 接口，用于处理消息，接受消息后对 js framework 的初始化操作进行控制。

发送消息：

``` java
    /**
     * Initialize JavaScript framework
     * @param framework String representation of the framework to be init.
     */
    public synchronized void initScriptsFramework(String framework) {
        Message msg = mJSHandler.obtainMessage();
        msg.obj = framework;
        msg.what = WXJSBridgeMsgType.INIT_FRAMEWORK;
        msg.setTarget(mJSHandler);
        msg.sendToTarget();
    }
```

接收消息：

``` java
    @Override
    public boolean handleMessage(Message msg) {
        if (msg == null) {
            return false;
        }

        int what = msg.what;
        switch (what) {
            case WXJSBridgeMsgType.INIT_FRAMEWORK:
                invokeInitFramework(msg);// 过程中会加载 main.js，调用c++ 进行Framework的初始化：mWXBridge.initFramework(framework, assembleDefaultOptions())
                break;
            case WXJSBridgeMsgType.CALL_JS_BATCH:
                invokeCallJSBatch(msg);
                break;
            case ...
```

在 WXBridge 中，是调用 底层native的方法：

``` java
  /**
   * Init JSFrameWork
   *
   * @param framework assets/main.js
   */
  public native int initFramework(String framework, WXParams params);
```
底层 C++代码中：

``` cpp
jint Java_com_taobao_weex_bridge_WXBridge_initFramework(JNIEnv *env,
                                                        jobject object, jstring script,
                                                        jobject params) 
```
js framework 初始化就交给 C++ 来处理了

## 注册操作
weex 内置的 component 和 module 都会在此过程中注册，这一过程还包含了dom的注册操作。
注册操作，会由registerModules来进行操作：registerModules()，registerComponents()，这两个都是异步执行，最终调用：

``` java
// modules the format is like {'dom':['updateAttrs','updateStyle'],'event':['openUrl']}
WXJSObject[] args = {new WXJSObject(WXJSObject.JSON,
                                        WXJsonUtils.fromObjectToJSONString(modules))};
mWXBridge.execJS("", null, METHOD_REGISTER_MODULES, args);

WXJSObject[] args = {new WXJSObject(WXJSObject.JSON,
                                        WXJsonUtils.fromObjectToJSONString(components))};
mWXBridge.execJS("", null, METHOD_REGISTER_COMPONENTS, args);
```
execJS() 最终是调用WXBridge.execJS() 的native方法，交给底层c++来执行：

``` cpp
jint Java_com_taobao_weex_bridge_WXBridge_execJS(JNIEnv *env, jobject this1, jstring jinstanceid,
                                                 jstring jnamespace, jstring jfunction,
                                                 jobjectArray jargs)
```
C++ 使用反射的方式，找到WXJSObject类，找到相关的属性和方法，

``` cpp
jclass jsObjectClazz = env->FindClass("com/taobao/weex/bridge/WXJSObject");
```
调用 js Framework的相关api 执行。

<br/>


源码：
https://github.com/apache/incubator-weex/tree/master/android/sdk
https://github.com/alibaba/weex_v8core