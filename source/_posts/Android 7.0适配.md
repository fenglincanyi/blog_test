---
title: Android 7.0适配
date: 2017-06-09 20:10
categories: Android
tags: [7.0适配, FileProvider]
---

Android 7.0 行为变更，涉及：电池和内存、后台优化、权限更改、NDK 应用链接至平台库。

作为开发者，我们要关注 权限更改、NDK私有库的问题，适配工作也是围绕这2者展开。

## 分享私有文件的方式
传递软件包网域外的 file:// URI 可能给接收器留下无法访问的路径。因此，尝试传递 file:// URI 会触发 FileUriExposedException。分享私有文件内容的推荐方法是使用 FileProvider

此类问题，我们要看 关注 拍照、覆盖安装。
原先的拍照，我们一般这样写：

``` java
public static void startCamera(Activity tag, String fileName) {
        if (!FileUtil.isSDMounted()) {
            ToastUtil.showShortToast(tag, "未挂载SD卡....");
            return;
        }
        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(new File(
                getExternalStorageDirectory(), fileName)));
        tag.startActivityForResult(intent, PHOTO_REQUEST_TAKEPHOTO);
    }
```
如果 适配到 7.0以上（targetSdkVersion >= 24），上面就会出错：**FileUriExposedException**
Google 也是越来越注重安全的升级了，不让使用这种方式。现在要通过 provider 来进行访问，由 FileProvider 统一去访问相关的目录、文件。
那具体，我们就看看怎么做：

### 定义FileProvider
在清单文件中，添加 provider：

``` xml
<provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.chinahr.android.m.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <!-- name：一般是固定的，如果你不需要定义自己的 provider; 
            如果需要别的特殊逻辑，你需要自定义 provider 去继承 v4 包下的 FileProvider并添加自己的逻辑代码，
            那么这时候，name 就是你自定义的 provider的全路径了
            authorities: packageName.fileprovider -->

            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/fileprovider" /><!-- file path 配置文件的路径 -->
        </provider>
```

### 配置路径文件
在固定的目录下，res/xml 目录下，新建自己的 xml。

然后我们开始配置，如何配置呢，我们来看看官方文档的要求：

``` xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 可以配置多个 -->
    <files-path name="my_images" path="images/"/>
    ...
</paths>
```
主要有4种路径：

| path      |     dir |
| :-------- | --------:|
|files-path   |   Context.getFilesDir() |
|cache-path  |   Context.getCacheDir) |
|external-path name="name" path="path"     |   Environment.getExternalStorageDirectory() |
|external-cache-path name="name" path="path"    |  Context.getExternalCacheDir()|

那我们来个demo：
在 xml 文件中配置：

``` xml
<paths>
    <external-path name="takephoto" path="tmp" />
    <external-path name="apk" path="Download" />
    <cache-path name="images" path="/" />
</paths>
```
代表的含义就是：
在 sd 下的 tmp目录，Download目录下对应 拍照后存储目录、apk下载的目录
在 packageName/cache 目录下存放app里的图片

配置工作，就ok了，现在解决拍照、覆盖安装崩溃的问题：

``` java
public static void startCameraN(Activity tag, String fileName) {
        if (!FileUtil.isSDMounted()) {
            ToastUtil.showShortToast(tag, "未挂载SD卡....");
            return;
        }
        // 必须和 xml 配置的 path 一致
        File file = new File(getExternalStorageDirectory(), "tmp/"+fileName);
        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        Uri pictureUri;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            pictureUri = FileProvider.getUriForFile(tag, tag.getPackageName()+".fileprovider", file);
            intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);// 加上权限，否则也是要出错的
        } else {
            pictureUri = Uri.fromFile(file);
        }

        intent.putExtra(MediaStore.EXTRA_OUTPUT, pictureUri);
        tag.startActivityForResult(intent, PHOTO_REQUEST_TAKEPHOTO);
    }
```
这样，我们就适配结束了

对于覆盖安装，同样的方式：

``` java
Intent installIntent = new Intent(Intent.ACTION_VIEW);
                Uri apkUri;
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                    installIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                    apkUri = FileProvider.getUriForFile(BaseAppUpdateActivity.this, getPackageName()+".fileprovider", apkFile);
                } else {
                    apkUri = Uri.parse("file://" + apkFile.toString());
                }
                installIntent.setDataAndType(apkUri, "application/vnd.android.package-archive");
                startActivity(installIntent);
                //退出整个app
                finishAllActivity();
```

具体可以看看文档：
https://developer.android.com/reference/android/support/v4/content/FileProvider.html?hl=zh-cn


## 使用 NDK 私有链接库的问题
从 Android 7.0 开始，系统将阻止应用动态链接非公开 NDK 库，这种库可能会导致您的应用崩溃。此行为变更旨在为跨平台更新和不同设备提供统一的应用体验。即使您的代码可能不会链接私有库，但您的应用中的第三方静态库可能会这么做。因此，所有开发者都应进行相应检查，确保他们的应用不会在运行 Android 7.0 的设备上崩溃。如果应用使用原生代码，则只能使用公开 NDK API。

应用可通过以下三种方式尝试访问私有平台 API：

* 应用直接访问私有平台库。应更新应用以添加该应用的库副本，或使用公开 NDK API。
* 应用使用一个可访问私有平台库的第三方库。即使您确定应用不会直接访问私有库，您仍应针对此情景测试应用。
* 应用引用一个其 APK 中未包含的库。例如，如果尝试使用自己的 OpenSSL 副本，但忘记将它与应用的 APK 进行捆绑，则可能会出现此情况。正常情况下，此应用可在包含 libcrypto.so 的 Android 平台版本上运行。不过，此应用在不包含此库的新版 Android（例如，Android 6.0 和更高的版本）上会崩溃。为修复此问题，请确保 APK 捆绑您的所有非 NDK 库。

这个问题呢，我们就得依靠别人来解决了，当然，如果有自己的 ndk 使用到了私有库，也是要解决的。

首先，我先检查下，有哪些库中使用了私有库，我们把 targetSdkVersion 改成 23，运行在 7.0的手机上，APP启动后，我们观察日志：

```
as a workaround for http://b/26394120, note that the access will be removed in future releases of Android.
06-05 16:10:28.556 4284-4284/com.chinahr.android.m W/linker: library "libcrypto.so" ("/vendor/lib/libcrypto.so") needed or dlopened by "/data/app/com.chinahr.android.m-1/lib/arm/libgmacs.so" is not accessible for the namespace "classloader-namespace" - the access is temporarily granted as a workaround for http://b/26394120, note that the access will be removed in future releases of Android.
```
发现有两处警告，都是来源于 libgmacs.so 中。联系别的部门做适配...

还有，此问题，如果项目 targetSdkVersion < 24，则会在 7.0手机上每次启动app时，弹出警告框：
![](http://7xr1vo.com1.z0.glb.clouddn.com/%E7%A7%81%E6%9C%89%E5%BA%93.png)
但也可以继续正常运行，但是这个很不友好，需要尽快联系提供者解决


## 其他 7.0 适配的小问题
### 加密方式

```
我们在 logcat 中看到：
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  ********** PLEASE READ ************
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  *
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  * New versions of the Android SDK no longer support the Crypto provider.
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  * If your app was relying on setSeed() to derive keys from strings, you
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  * should switch to using SecretKeySpec to load raw key bytes directly OR
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  * use a real key derivation function (KDF). See advice here :
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  * http://android-developers.blogspot.com/2016/06/security-crypto-provider-deprecated-in.html
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  ***********************************
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  Returning an instance of SecureRandom from the Crypto provider
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  as a temporary measure so that the apps targeting earlier SDKs
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  keep working. Please do not rely on the presence of the Crypto
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  provider in the codebase, as our plan is to delete it
06-05 16:10:28.564 4284-4284/com.chinahr.android.m E/System:  completely in the future.
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  ********** PLEASE READ ************
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  *
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  * New versions of the Android SDK no longer support the Crypto provider.
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  * If your app was relying on setSeed() to derive keys from strings, you
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  * should switch to using SecretKeySpec to load raw key bytes directly OR
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  * use a real key derivation function (KDF). See advice here :
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  * http://android-developers.blogspot.com/2016/06/security-crypto-provider-deprecated-in.html
06-05 16:10:28.566 4284-4284/com.chinahr.android.m E/System:  ***********************************
```
发现 google 对加密方式也有了要求。
名为 Crypto 的 JCA 提供程序已弃用，因为它仅有的 SHA1PRNG 算法为弱加密。应用无法再使用 SHA1PRNG（不安全地）派生密钥，因为不再提供此提供程序。
而且在未来的版本中，如果继续使用Crypto，将会出错。我们以后需要专注。
### 通知栏
Android 各个版本中，貌似对通知栏都有一定的更改，而且现在也变得很丰富
7.0 的通知栏，改变了样式，增加了小图标，还可以显示通知数量。

自己在玩原生7.0 系统时，发现小图标变成灰色的方块了：
![](http://7xr1vo.com1.z0.glb.clouddn.com/device-2017-06-08-142841.png)

然后，发现在 华为 7.0 手机上，小图标显示又是正常的，应该是华为对通知栏显示做了处理
![](http://7xr1vo.com1.z0.glb.clouddn.com/device-2017-06-08-195515.png)

这时候就比较蹩脚了，暂且这么干吧，我们项目中，使用了小米推送（通知消息），查阅了小米推送的文档：

    * 如果app中同时存在名为mipush_notification和mipush_small_notification的drawable文件，则使用mipush_notification的drawable作为通知的大图标，mipush_small_notification的drawable作为通知的小图标。
    * 如果app中只存在其中一个drawable文件，则使用该drawable作为通知的图标。
    * 如果app中不存在这两个drawable文件，则使用app的icon作为通知的图标。在MIUI中，通知栏图标统一显示为app的icon，不可以定制。

可以添加个小图标，在推送到达时候显示，透明背景的图片，术语叫：带有alpha 通道的图片。

本来也可以通过 自己控制 NotificationManager 来创建显示，如：

``` java
NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        Notification notification = builder
                .setContentTitle("这是通知标题")
                .setContentText("这是通知内容这是通知内容这是通知内容这是通知内容这是通知内容这是通知内容这是通知内容这是通知内容")
                .setWhen(System.currentTimeMillis())
                .setSmallIcon(R.mipmap.ic_launcher)
//                .setColor(Color.parseColor("#ff0000”))// 设置小图标背景颜色
                .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
                .build();
        manager.notify(1, notification);
```

![](http://7xr1vo.com1.z0.glb.clouddn.com/device-2017-06-09-162941.png)
这是我们可以定制的
但是现在的项目比较死，暂时不适用自定义的了
暂时使用小米推送的改动小图标的方式，替换上一个透明背景的logo吧，总比灰色的方块强吧。而且其他app也是同样的显示效果，如上面的图标中的百度淘宝和头条。
有精力的话，使用透传消息，可以自己控制通知栏的样式和显示。
