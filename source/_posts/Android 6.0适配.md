---
title: Android 6.0适配
date: 2017-04-12 12:40
categories: Android
tags: [6.0适配, 运行时权限]
---

## Android 权限变更
在 Android 6.0 之前，APP 的授权是通过在 manifest.xml 文件中，去注册相关权限，安装时就可以获得所有声明了的权限。而在 6.0 及以后的版本，这种在 manifest.xml 声明权限的做法，已经在一些权限上不适用了。
在Android 6.0 及以后的系统中，用户可以自动打开或关闭任何应用的权限（设置 -> 应用权限）

### 权限的划分
Android 是基于 Linux 系统的，所以，对于每个文件都有个权限的概念。
这次变化，主要将 Android 系统中的权限分为：
* 正常权限：如很常见的 访问网络；
* 危险权限：如访问手机状态、位置信息；
* 特殊权限：如写设置、悬浮窗

另外，还有一个 **权限组** 的概念：

  任何权限都可属于一个权限组，包括正常权限和应用定义的权限
所以，每个危险权限都属于不同的权限组中，Google已经定义好了这些归属。

我们主要关注的危险权限和权限组如下：

| 权限组      |     权限 |
| :-------- | --------:|
| CALENDAR    |   READ_CALENDAR<br/>WRITE_CALENDAR |
| CAMERA    |   CAMERA |
| CONTACTS    |   READ_CONTACTS<br/>WRITE_CONTACTS<br/>GET_ACCOUNTS |
| LOCATION    |   ACCESS_FINE_LOCATION<br/>ACCESS_COARSE_LOCATION|
| MICROPHONE    |   RECORD_AUDIO|
| PHONE    |   READ_PHONE_STATE<br/>CALL_PHONE<br/>READ_CALL_LOG<br/>WRITE_CALL_LOG<br/>ADD_VOICEMAIL<br/>USE_SIP<br/>PROCESS_OUTGOING_CALLS|
| SENSORS    |   BODY_SENSORS|
| SMS    |   SEND_SMS<br/>RECEIVE_SMS<br/>READ_SMS<br/>RECEIVE_WAP_PUSH<br/>RECEIVE_MMS|
| STORAGE    |   READ_EXTERNAL_STORAGE<br/>WRITE_EXTERNAL_STORAGE|

> 如果已经申请到了某个组的某一个权限时，那么也同时拥有了该组的其他权限；只要拥有了同类中的一个，那么就拥有了该组的其他权限

大部分的适配工作就是对以上的权限进行预处理，就OK了。

### 不同版本运行，区别对待
* 如果你的 targetSdkVersion <= 22，那么安装在 6.0 以上的手机的话，还是以老的授权方式（安装时期）对所有声明了的权限一次性授权。
* 如果你的 targetSdkVersion > 22，那么安装在 6.0 以上的手机上，不光要在 manifest.xml 中声明，而且有一部分比较危险或者特殊的权限， 需要在 app 运行时期，来提前申请权限。一旦未申请而直接进行操作的话，会直接崩溃SecurityException。


## 权限处理原则
Google 对权限做了进一步要求之后，作为我们开发者，该如何作何权限的适配呢?
官方也给出了几个建议：
### 考虑使用 Intent
我们一般可以使用以下 2 种方法，来执行某个任务：
* 要求提供权限才能执行操作
* 使用 intent，让其他应用来执行任务

比如：
我们要使用拍照的功能，那么我们可以请求 CAMERA 权限，然后可以直接访问相机，控制相机进行拍照；也可以不用申请 CAMERA 权限，我们通过 ACTION_IMAGE_CAPTURE intent 来请求，系统会提示用户选择相机，拍照后返回 onActivityResult()。

类似的：打电话、访问用户的联系人 也是同样的做法

这两种方式各有优缺点：
使用权限：
* 应用可在您执行操作时完全控制用户体验；不过，如此广泛的控制会增加任务的复杂性，因为您需要设计适当的 UI。
* 系统会在运行或安装应用时各提示用户提供一次权限，应用即可执行操作，不再需要用户进行其他交互。不过，如果用户不授予权限（或稍后撤销权限），应用将根本无法执行操作。

使用 intent:
* 无需为操作设计 UI。处理 intent 的应用将提供 UI。不过，这意味着您无法控制用户体验。用户可能从未见过的应用交互。
* 如果用户没有适用于操作的默认应用，则系统会提示用户选择一款应用。如果用户未指定默认处理程序，则他们每次执行此操作时都必须处理一个额外对话框。

### 仅要求APP所需的权限
每次您要求权限时，实际上是在强迫用户作出决定。
您应尽量减少提出这些请求的次数。

### 不要让用户感到无所适从
某些情况下，一项或多项权限可能是应用所必需的。在这种情况下，合理的做法是，在应用启动之后立即要求提供这些权限。
在这种情况下，合理的做法是，在应用启动之后立即要求提供这些权限。例如，如果您运行摄影应用，应用需要访问设备的相机。在用户首次启动应用时，他们不会对提供相机使用权限的要求感到惊讶。

### 解释需要权限的原因
系统在您调用 requestPermissions() 时显示的权限对话框将说明应用需要的权限，但不会解释为何需要这些权限。某些情况下，用户可能会感到困惑。因此，最好在调用 requestPermissions() 之前向用户解释应用需要相应权限的原因。
例如，摄影应用可能需要使用位置服务，以便能够为照片添加地理标签。通常，用户可能不了解照片能够包含位置信息，并且对摄影应用想要了解具体位置感到不解。因此在这种情况下，应用最好在调用 requestPermissions() 之前告知用户此功能的相关信息。
<br/>
以上的准则，我们可以参考 **微信** 或者 **头条** 进行分析，他们是如何处理好与用户的交互的

## 6.0 权限适配
适配工作我们要对市面上常用的手机进行适配，大部分的手机还是遵守这些规则的。但是对部分小米或魅族系统的权限，厂商还是做了自己的定制的，我们需要花点心思来处理。

### 普通手机适配
一般流程我们分为 3步：
* checkPermission
* requestPermission
* handleResult

如果我们自己实现，就使用 ContextCompat.checkSelfPermission 检查需要的权限；
如果不是被授权，则我们需要请求这个权限 ActivityCompat.requestPermissions，可以同时请求一个或多个权限；
然后我们重写 Activity 的 onRequestPermissionsResult() 方法，根据用户授权结果，进行相应的结果处理。

不再造轮子了，我项目中了 RxPermission 开源库，并借鉴了EasyPermission，AndPermission 的思路。
代码层面不再展开，实现比较容易。。。

**处理流程：**
![](http://7xr1vo.com1.z0.glb.clouddn.com/%E9%80%82%E9%85%8D.png)

我们再来看看 微信 的流程，在第一次安装，启动后，就会要求3个比较重要的权限，一个一个来请求获取，如果一旦有未同意的，就不让进入。
这种做法也是比较合理的，这些权限都没同意，后续的操作没法正常进行。


### 小米、魅族适配

适配最头痛的就是 这种 爱搞特殊的手机了，没办法，手机碎片化的问题只能盼着慢慢消磨了。

小米、魅族手机的话，会出现 checkPermission 时，返回true，但是我们使用的时候实际是 关闭权限的；
这时候，我们就不能只 check了，还要 使用 AppOpsManager 来获取相关的操作，看这个操作是否有执行的权限。
通过这样双重的检查权限，都通过了，才可以执行操作。

``` java
public static boolean hasPermission(@NonNull Context context, @NonNull List<String> permissions) {
        if(Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
            return true;
        } else {

            for (Object permission : permissions) {
                // 获取app中 权限关联的操作
                String op = AppOpsManagerCompat.permissionToOp(permission.toString());
                if (!TextUtils.isEmpty(op)) {
                    // 查看 操作 是否被允许
                    int result = AppOpsManagerCompat.noteProxyOp(context, op, context.getPackageName());
                    if (result == MODE_IGNORED) {
                        return false;
                    }

                    // 再次检查，确保有权限
                    result = ContextCompat.checkSelfPermission(context, permission.toString());
                    if (result != PERMISSION_GRANTED) {
                        return false;
                    }
                }
            }

            return true;
        }
    }
```

在小米论坛上，已经有人提到这个问题，解决方法就是再次使用 AppOpsManager 来检查。
http://www.miui.com/thread-4498742-1-1.html


## 总结
* Android 安全架构的中心设计点是：**在默认情况下任何应用都没有权限执行对其他应用、操作系统或用户有不利影响的任何操作**
* 可以使用 intent 启动的操作，尽量还是使用intent吧，申请权限毕竟是要弹框阻碍用户操作的，而且如果没同意，还要做相应的操作来配合。

--------

参考：

https://developer.android.com/guide/topics/permissions/normal-permissions.html?hl=zh-cn
https://developer.android.com/training/permissions/best-practices.html?hl=zh-cn#perms-vs-intents
https://mp.weixin.qq.com/s?__biz=MzIxNzEyMzIzOA==&mid=2652313851&idx=1&sn=a15519b65e7bedefbb566fe6d01935cb&scene=4#wechat_redirect
