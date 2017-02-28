---
title: Weex SDK源码分析（二）
date: 2017-2-28 19:32
categories: Weex
tags: [Weex, Android]
---
>  **Weex 渲染页面过程**

## weex 渲染入口 render()
渲染过程从 WXSDKInstance.render() 开始追溯，render()方法是异步执行渲染工作的。render()重载的方法比较多，此处介绍最基础的一个。
参数代表的含义：
* pageName：用于查看日志，渲染的哪个页面作为标识
* template：加载的本地的或远程的 js（we文件被weex transform后的 js）
* options：配置信息，包括系统版本、app版本、设备信息等
* jsonInitData：用于渲染的初始数据
* flag：监听渲染的时机：
    * APPEND_ASYNC：渲染完第一个view后，IWXRenderListener.onViewCreated()调用
    * APPEND_ONCE：渲染完整个view tree后，onViewCreated()调用

``` java
wxSDKInstance.render(
      getPageName(),
      template,
      options,
      jsonInitData,
      ScreenUtil.getDisplayWidth(this),
      ScreenUtil.getDisplayHeight(this),
      WXRenderStrategy.APPEND_ASYNC);
```
接着，渲染工作真正开始于 renderInternal()：

``` java
private void ensureRenderArchor(){
    if(mRenderContainer == null){
      mRenderContainer = new RenderContainer(getContext());
      mRenderContainer.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
      mRenderContainer.setBackgroundColor(Color.TRANSPARENT);
      mRenderContainer.setSDKInstance(this);
      mRenderContainer.addOnLayoutChangeListener(this);
    }
  }
```
会进行 mRenderContainer 的初始化，RenderContainer 是一个 weex 扩展的 FrameLayout，后期操作，会在这个 container 中添加view元素。
我们接着看 render 过程，WXSDKManager.createInstance() 创建实例：

``` java
void createInstance(WXSDKInstance instance, String code, Map<String, Object> options, String jsonInitData) {
     mWXRenderManager.registerInstance(instance);
     mBridgeManager.createInstance(instance.getInstanceId(), code, options, jsonInitData);
  }
```
此过程中，WXRenderManager 注册实例，实际就是在 mRegistries（map<wxSDKInstanceid, wxRenderStatement>）中存储该实例的信息，并且创建 renderStatement。
mBridgeManager 创建实例时，WXModuleManager 会创建 DomModule，由WXDomManager进行创建，sDomModuleMap 存储该 wxSDKInstance 的WXDomModule。
然后，异步执行 invokeCreateInstance()，此过程还会执行 initFramework() 操作。initFramework() 后，会创建出一个 WXJSObject 将 wxsdkinstance相关的数据存储，组成一个数组的形式，传给执行 js 的方法：

``` java
WXJSObject instanceIdObj = new WXJSObject(WXJSObject.String,
                instance.getInstanceId());
        WXJSObject instanceObj = new WXJSObject(WXJSObject.String,
                                                template);
        WXJSObject optionsObj = new WXJSObject(WXJSObject.JSON,
                options == null ? "{}"
                        : WXJsonUtils.fromObjectToJSONString(options));
        WXJSObject dataObj = new WXJSObject(WXJSObject.JSON,
                data == null ? "{}" : data);
        WXJSObject[] args = {instanceIdObj, instanceObj, optionsObj,
                dataObj};
        invokeExecJS(instance.getInstanceId(), null, METHOD_CREATE_INSTANCE, args,false);
```
然后，WXBridgeManager 会调用 mWXBridge.execJS() 去调用底层的 C++方法，执行js。

## 底层C++ 调用Weex Android 功能
我们再来看看 c++ 代码实现的功能，以 callNative() 为例：

``` cpp
/**
 *  This Function is a built-in function that JS bundle can execute
 *  to call native module.
 */
v8::Handle<v8::Value> callNative(const v8::Arguments &args) {

    JNIEnv *env = getJNIEnv();
    //instacneID args[0]
    jstring jInstanceId = NULL;
    if (!args[0].IsEmpty()) {
        v8::String::Utf8Value instanceId(args[0]);
        jInstanceId = env->NewStringUTF(*instanceId);
    }
    //task args[1]
    jbyteArray jTaskString = NULL;
    if (!args[1].IsEmpty() && args[1]->IsObject()) {
        v8::Handle<v8::Value> obj[1];
        v8::Handle<v8::Object> global = V8context->Global();
        json = v8::Handle<v8::Object>::Cast(global->Get(v8::String::New("JSON")));
        json_stringify = v8::Handle<v8::Function>::Cast(json->Get(v8::String::New("stringify")));
        obj[0] = args[1];
        v8::Handle<v8::Value> ret = json_stringify->Call(json, 1, obj);
        v8::String::Utf8Value str(ret);

        int strLen = strlen(ToCString(str));
        jTaskString = env->NewByteArray(strLen);
        env->SetByteArrayRegion(jTaskString, 0, strLen,
                                reinterpret_cast<const jbyte *>(ToCString(str)));
        // jTaskString = env->NewStringUTF(ToCString(str));

    } else if (!args[1].IsEmpty() && args[1]->IsString()) {
        v8::String::Utf8Value tasks(args[1]);
        int strLen = strlen(*tasks);
        jTaskString = env->NewByteArray(strLen);
        env->SetByteArrayRegion(jTaskString, 0, strLen, reinterpret_cast<const jbyte *>(*tasks));
        // jTaskString = env->NewStringUTF(*tasks);
    }
    //callback args[2]
    jstring jCallback = NULL;
    if (!args[2].IsEmpty()) {
        v8::String::Utf8Value callback(args[2]);
        jCallback = env->NewStringUTF(*callback);
    }

    if (jCallNativeMethodId == NULL) {
        // 反射方式，拿到 weexsdk 的callNative方法的ID
        jCallNativeMethodId = env->GetMethodID(jBridgeClazz,
                                               "callNative",
                                               "(Ljava/lang/String;[BLjava/lang/String;)I");
    }

    // 调用 java 的 callNative 方法
    int flag = env->CallIntMethod(jThis, jCallNativeMethodId, jInstanceId, jTaskString, jCallback);
    if (flag == -1) {
        LOGE("instance destroy JFM must stop callNative");
    }
    env->DeleteLocalRef(jTaskString);
    env->DeleteLocalRef(jInstanceId);
    env->DeleteLocalRef(jCallback);

    return v8::Integer::New(flag);
}
```
C++ 解析 js 代码后，返回来再调用 weex sdk 的 Android 代码，通过 JNI 提供的反射方式拿到方法的 id，并调用执行。
相对应的，我们找到 Android callNative() 代码：

``` java
public int callNative(String instanceId, String tasks, String callback) {
    long start = System.currentTimeMillis();
    WXSDKInstance instance = WXSDKManager.getInstance().getSDKInstance(instanceId);
    if(instance != null) {
      instance.firstScreenCreateInstanceTime(start);
    }
    int errorCode = IWXBridge.INSTANCE_RENDERING;
    try {
      // 通过 WXBridgeManager.callNative 进行 tasks 的分发执行
      errorCode = WXBridgeManager.getInstance().callNative(instanceId, tasks, callback);
    }catch (Throwable e){
      //catch everything during call native.
      if(WXEnvironment.isApkDebugable()){
        WXLogUtils.e(TAG,"callNative throw exception:"+e.getMessage());
      }
    }

    if(instance != null) {
      instance.callNativeTime(System.currentTimeMillis() - start);
    }
    if(WXEnvironment.isApkDebugable()){
      if(errorCode == IWXBridge.DESTROY_INSTANCE){
        WXLogUtils.w("destroyInstance :"+instanceId+" JSF must stop callNative");
      }
    }
    return errorCode;
  }
```
## Weex native 响应 C++ 的调用
WXBridgeManager.callNative() 对 tasks 进行分发，我们来看看如何分发的：

``` java
JSONArray array = JSON.parseArray(tasks);
... ...
int size = array.size();
    if (size > 0) {
      try {
        JSONObject task;
        for (int i = 0; i < size; ++i) {
          task = (JSONObject) array.get(i);
          if (task != null && WXSDKManager.getInstance().getSDKInstance(instanceId) != null) {
            Object target = task.get(MODULE);
            if(target != null){
              if(WXDomModule.WXDOM.equals(target)){
                WXDomModule dom = getDomModule(instanceId);
                dom.callDomMethod(task);
              }else {
                WXModuleManager.callModuleMethod(instanceId, (String) target,
                    (String) task.get(METHOD), (JSONArray) task.get(ARGS));
              }
            }else if(task.get(COMPONENT) != null){
              //call component
              WXDomModule dom = getDomModule(instanceId);
              dom.invokeMethod((String) task.get(REF),(String) task.get(METHOD),(JSONArray) task.get(ARGS));
            }else{
              throw new IllegalArgumentException("unknown callNative");
            }
          }
        }
      } catch (Exception e) {
        WXLogUtils.e("[WXBridgeManager] callNative exception: ", e);
        commitJSBridgeAlarmMonitor(instanceId, WXErrorCode.WX_ERR_INVOKE_NATIVE,"[WXBridgeManager] callNative exception "+e.getCause());
      }
    }
    ... ...
```
分发规则，对 tasks 逐个分类处理：
* 如果是 module:
    * 如果 是 dom：执行 WXDomModule.callDomMethod()；
    * 如果 不是 dom (普通的WXModule) : 执行WXModuleManager.callModuleMethod()
* 如果是 component：
    * 调用 WXDomModule.invokeMethod()

task 数据格式（json），如图：
![](http://7xr1vo.com1.z0.glb.clouddn.com/weex%20debug%E5%9B%BE.png)

这里只分析第一种情况（其他相似）：
WXDomModule.callDomMethod()：
会根据 task 的 method 属性分类处理，此处只分析一个：CREATE_BODY：
会继续调用 createBody() 方法，将 wxSDKInstance.id 和 task的args 信息发送给WXDomHandler。
WXDomHandler 的 handleMessage() 中也有很多分类处理，对于WX_DOM_CREATE_BODY：会执行：
```java
mWXDomManager.createBody(task.instanceId, (JSONObject) task.args.get(0));
```
具体可参考 WXDomModule、WXDomHandler 类分析
这个时候，就回到我们之前说过的 WXDomStatement 的流程了，WXDomStatement会添加dom 节点（addDomInternal() ），在 addDomInternal 过程中，对dom节点又是一通猛烈的操作：根节点（root节点的准备工作），普通节点（add到父节点下）。
然后，对 dom 对象 进行遍历操作（递归）：domObject.traverseTree()，在dom 线程创建 component，生成 component 树（也是递归操作：通过WXRenderStatement.generateComponentTree() ）。
对于每个dom节点都会进行 setLayout()、setExtra()、setPadding()，可以去看看compent.setLayout()，就是去使用Android的API 对view进布局和绘制，此处不再贴代码。
![](http://7xr1vo.com1.z0.glb.clouddn.com/weex%20component%20%E6%96%B9%E6%B3%95.png)



## 总结
* WXBridge 是 Android 与 底层 C++ 的衔接处，Android <=> C++ 的交互，及方法接口都在此类中，的确起到了见名知义的效果
* Weex Android SDK 是使用 Android的API（Java层），实现了view的布局、绘制。并不是依靠 底层so包实现，与IOS的sdk实现方式不同（听同事说......）
* component 处置 weex 的view绘制、布局过程，和Android native处理view的流程十分相似，具体可以参考后面的链接（阿里大神也分析过）

<br/>

参考：
http://www.jianshu.com/p/3160a0297345
