---
title: 回炉：Android进程间通信实践
date: 2017-12-28 01:38
categories: Android
tags: [IPC, Service, AIDL]
---
## 引言
Android的跨进程通信，普通的业务场景中遇到的很少，但又是Android开发中很重要的一个知识点，作为Android Developer 不能不掌握。
最近的工作中需要实现这一功能，所以完成功能之后，决定回炉巩固一下。比较涉及底层的内容，都是比较有意思的。
进程间通信，不外乎一下的几种方式：Intent、四大组件、AIDL、Socket、共享文件。
我们来一一过一遍......

## Intent
我们常用隐示意图intent，去调用一些系统功能，如：拨打电话，打开相册：
```java
Intent callIntent = new  Intent(Intent.ACTION_CALL, Uri.parse("tel:12345678" ));
startActivity(callIntent);

Intent intent = new Intent(Intent.ACTION_PICK);
intent.setDataAndType(MediaStore.Images.Media.INTERNAL_CONTENT_URI, "image/*");
startActivityForResult(intent, 999);
```
Android系统会自动识别Action和过滤相关的应用，调起相关app, 这种最常见不过，我们不多说......

当然，上面这种是系统一定已经定义好的action，那么我们可以自己定义个action，来拉起app，如：
```xml
<activity android:name=".MainActivity">
	<intent-filter>
		<action android:name="com.geng.app2.test.callbyother"/>
		<!-- <data>  也可以定义 scheme 什么的 -->
		<!-- 必须加 DEFAULT -->
		<category android:name="android.intent.category.DEFAULT" />
	</intent-filter>
</activity>
```
```java
Intent intent = new Intent("com.geng.app2.test.callbyother");
startActivity(intent);
```
这样，也就可以启动了
这种方式呢，请注意一个坑：
**当android:mimeType和android:scheme同时存在的时候**：
https://www.jianshu.com/p/26a69e4b9ccc
**需要通过setDataAndType()方法，把 scheme 和 mimeType 同时设置进去才能匹配对应的清单文件中的 intent-filter**

再详细的话，就涉及到app拉起相关的知识了，我们不在讨论下去
## 四大组件
Android四大组件都是可以实现进程间通信的，当然各种利弊也是很明显的，具体要按需选择实现功能
### Activity
activity 通过 传入 三方app的完整包名路径、及类名，就可以吊旗任意一个app的页面了
```java
Intent intent = new Intent();
intent.setComponent(new ComponentName(packName, "com.android.demo.OtherActivity"));
intent.setFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
intent.putExtra("data", "data_info");
startActivity(intent);
// 或者 startActivityForResult(intent, REQUEST_CODE);
```
### Receiver
receiver的方式也是很简单的，通过定义receiver的action，data 也能实现。
**App B中**，静态注册receiver：
```xml
<receiver android:name=".TestReceiver">
	<intent-filter>
        <action android:name="com.android.test"/>
    </intent-filter>
</receiver>
```
```java
public class TestReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        System.out.println(intent.getStringExtra("data_info")+"");
    }
}
```
**App A中**，发送广播：
```java
Intent intent = new Intent();
intent.setAction("com.android.test");
intent.putExtra("data_info", "come from test");
sendBroadcast(intent);
```
理想情况下（原生ROM），这种方式，广播是可以收到的，即使App B是未运行的（进程是死的）。在国内各种系统定制下，这个已经在APP 死掉的情况下，不好使了。
如果，业务允许情况只在目标 app 活着的情况下，实现功能，这种广播的方式也是可以使用的。（这时，动态注册receiver也是可以的）

另外我们注意一个：
* 粘性广播（先发送，后注册）
这种广播 已经被系统**摒弃**了，如果非要使用，需要在发送的时候加一个权限：
```xml
<uses-permission android:name="android.permission.BROADCAST_STICKY" />
```
```java
Intent intent = new Intent();
intent.setAction("xxxxxxx");
intent.putExtra("info", "sticky broadcast has been receiver");
sendStickyBroadcast(i);
```
receiver 中，和普通的广播注册一样，静态或动态注册都可以。
粘性广播是一种Android系统长期持有的广播，系统保存了相关的intent信息，直到有符合条件的receiver匹配，才会让其响应。这种广播副作用比较大，所以系统以及不推荐使用了。
另外，我们可以在收到粘性广播后，移除掉：
```java
context.removeStickyBroadcast(intent);
```
### Provider
ContentProvider是Android提供的app间数据共享的方案，为其他app(调用方)提供调用接口，读取/写入本app的数据。当然自身也是可以，通过调用provider更新自己的数据的。其本质，是对数据库的操作。
**在App A中**，定义provider及数据库：
`DataInfoProvider.java`
```java
public class DataInfoProvider extends ContentProvider {

	/**
	 * AUTHORITY 必须是所有app中唯一的
	 */
    private static String AUTHORITY;// 此处：包名+.provider
    private static final UriMatcher mMatcher;
    private static final int TABLE_CODE = 1;

    private SQLiteDatabase mDatabase;

    static {
        mMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        // 初始化
        //关联不同的URI和code，便于后续getType返回不同的类型
    }

    @Override
    public boolean onCreate() {
        initProvider();
        return false;
    }

    /**
     * 初始化数据库
     */
    private void initProvider() {
        if (getContext() == null) {
            return;
        }
        AUTHORITY = getContext().getPackageName() + ".provider";
        mMatcher.addURI(AUTHORITY, DbHelper.TABLE_NAME, TABLE_CODE);

        mDatabase = DbHelper.getHelper(getContext().getApplicationContext()).getWritableDatabase();
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        String tableName = getTableName(uri);
        if (tableName != null && mDatabase!= null) {
            return mDatabase.query(tableName, projection, selection, selectionArgs, null, null, sortOrder);
        }
        return null;
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        return null;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
        return 0;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
        String tableName = getTableName(uri);
        if (tableName != null && mDatabase!= null) {
            return mDatabase.update(tableName, values, selection, selectionArgs);
        }
        return -1;
    }

    private String getTableName(final Uri uri) {
        switch (mMatcher.match(uri)) {
            case TABLE_CODE:
                return DbHelper.TABLE_NAME;
            default:
                return null;
        }
    }
}
```
`DbHelper.java`
```java
public class DbHelper extends SQLiteOpenHelper {

    private static final String DB_NAME = "data_info.db";
    private static final int DB_VERSION = 1;
    public static final String TABLE_NAME = "info";

    private static DbHelper helper;
    private DbHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }

    public synchronized static DbHelper getHelper(Context appContext) {
        if (helper == null) {
            helper = new DbHelper(appContext);
        }
        return helper;
    }

    @Override
    public void onCreate(final SQLiteDatabase sqLiteDatabase) {
        new Thread(){
            @Override
            public void run() {
                final String sqlStr = "CREATE TABLE IF NOT EXISTS "+ TABLE_NAME +
                        "(_id integer primary key autoincrement not null, " +
                        "package_name varchar(50) not null, "+
                        "package_tag integer not null)";
                sqLiteDatabase.execSQL(sqlStr);
            }
        }.start();
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
    }
}
```

```xml
<provider
    android:name="com.android.geng.DataInfoProvider"
    android:authorities="com.android.test.provider"
    android:exported="true" />
    <!-- 供外界访问 -->
```
**在App B中**，调用 A 的provider：
```java
// 查询数据
String s = "content://com.android.test.provider/data_info/";// content://authority/tablename/
Cursor c = resolver.query(Uri.parse(s), null, null, null, null);
if (c != null && c.getCount() > 0 && c.moveToNext()) {
	// 读取 Cursor
	int packageTag = c.getInt(c.getColumnIndex("package_tag"));
}
if (c != null) {
	c.close();
}

// 更新数据
ContentValues values = new ContentValues();
values.put("package_tag", 100);
// result : 更新了多少行
int result = resolver.update(Uri.parse(s), values, "package_name = ?", new String[]{"com.android.test"});
```
同样的，这种通信方式，在原生系统上，极端情况下（App A进程死的），也是可以访问其provider的。经过测试，小米普遍可以；华为、oppo、vivo基本上都是不同程度的限制。都是为了防止app胡乱进行保活机制。
### Service
Service 无疑是跨进程通信中用的最多的，可以通过好几种方式进行消息传递，生命周期也是和activity有所不同，优先级比activity低。
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/service_lifecycle.png" width="320" height="440" style="margin:0 auto;display:block"/>

系统提供2种方式让我们来玩耍:
#### startService
startService 首次首次调用，会经历上图的过程；后面再次start, 只会走onStartCommand();
startService方式 必须管理自己的生命周期。也就是说，除非系统必须回收内存资源，否则系统不会停止或销毁服务，而且服务在 onStartCommand() 返回后会继续运行。
可以由自身 stopSelf() 或者 其他组件调用 stopService(), 系统就会销毁Service
注意的有：
**onStartCommand() 的返回值**

* START_NOT_STICKY
如果系统在 onStartCommand() 返回后终止服务，则除非有挂起 Intent 要传递，否则系统不会重建服务。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。
* START_STICKY
如果系统在 onStartCommand() 返回后终止服务，则会重建服务并调用 onStartCommand()，但不会重新传递最后一个 Intent。相反，除非有挂起 Intent 要启动服务（在这种情况下，将传递这些 Intent），否则系统会通过空 Intent 调用 onStartCommand()。这适用于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务）。
* START_REDELIVER_INTENT
如果系统在 onStartCommand() 返回后终止服务，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()。任何挂起 Intent 均依次传递。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

总之，就是在重建Service时的策略，见名知意

另外，三方app也是可以直接启动Service的，达到数据同步、通信的目的。
其方式 和 Activity启动其他app 的一致，还要保证，要启动的Service是 exported="true"。
#### bindService
bindService 也有自己的管理方式，同理上面的，**bind & unBind**就可以做到。
如果同一个Service被同一个组件bind多次会怎么样，会报异常/不生效，如果非要多次，则需要unbind, 再bind
但是同一个Service可以被多个组件同时bind, 只有当所有的绑定着都 unbind, Service 才会销毁
这种绑定的方式，有个缺陷，就是当时开启绑定的组件，如activity中 bindService, 如果activity销毁了，那么这个Service就会销毁，走生命周期：unbind -> onDestory。

**对比之下：bindService 在绑定者销毁后，则Service会被销毁(走unBind)
    如果是 startService 方式，则Service不会销毁
    如果既 start, 又bind, 那么，Service也不会销毁
**
#### Messenger
有了Service的基础，我们继续深入下：
Messenger，就像是一种串行的消息机制，它是一种轻量级的IPC方案，借助于bindService的方式，可以在不同进程中传递Message对象，我们在Message中放入需要传递的数据即可轻松实现进程间通讯。但是当我们需要调用服务端方法，或者存在并发请求，那么Messenger就不合适了。把需要传递的数据，用Intent封装起来传递即可。
必须以Bundle包裹基本数据类型，否则会报错，因为未序列化
```java
// java.lang.RuntimeException: Can't marshal non-Parcelable objects across processes.
```
这次我们来玩这样一个游戏：
有两个app, A,B
* A->B:startService;
* B->A:bindService, 并通过messenger发送消息;
* A->B,messenger发送响应消息。

**App A:**
`MainActivity.java`
```java
public void onClickView(View view) {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName("com.geng.bapp", "com.geng.bapp.BAppService"));
    intent.putExtra("app_source", getPackageName());
    startService(intent);
}
```
`AppService.java`
```java
// Service 注册，android:exported="true"
// 服务端
public class AppService extends Service {

    private Messenger messenger;

    private static class MessageHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 200:
                    Bundle data = msg.getData();
                    if (data == null) {
                        return;
                    }
                    System.out.println("A收到B的消息： " + data.getString("source") + ", " + data.getBoolean("user_setting") + ", " + data.getInt("priority"));

                    Messenger clientMessenger = msg.replyTo;
                    Message m = Message.obtain();
                    m.what = 100;
                    Bundle bundle = new Bundle();
                    bundle.putString("reply msg", "A received B msg");
                    m.setData(bundle);
                    try {
                        clientMessenger.send(m);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }

                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        messenger = new Messenger(new MessageHandler());
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return START_NOT_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        // 返回binder
        return messenger.getBinder();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
    }
```
**App B:**
`BAppService.java`
```java
// Service 注册，android:exported="true"
// 客户端
public class BAppService extends Service {

    private boolean isBinded;

    private static class MessengerReplyHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 100:
                    Bundle b  = msg.getData();
                    if (b != null && !TextUtils.isEmpty(b.getString("reply msg"))) {
                        System.out.println("B收到A的响应：" + b.getString("reply msg"));
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    private Messenger clientMessenger = new Messenger(new MessengerReplyHandler());


    private ServiceConnection sc = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            isBinded = true;

            Messenger serverMessenger = new Messenger(service);
            Message msg = Message.obtain();

            msg.what = 200;
            Bundle bundle = msg.getData();
            bundle.putString("source", "browser");
            bundle.putBoolean("user_setting", false);
            bundle.putBoolean("setting_state", true);
            bundle.putInt("priority", 100);

            msg.replyTo = clientMessenger;
            try {
                serverMessenger.send(msg);// 给服务端发消息
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            isBinded = false;
        }
    };

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String serverApp = intent.getStringExtra("app_source");

        System.out.println("B onStartCommand: "+ serverApp);

        Intent in = new Intent();
        in.setComponent(new ComponentName(serverApp, serverApp+".AppService"));

        if (isBinded) {
            unbindService(sc);
        }
        bindService(in, sc, BIND_AUTO_CREATE);//自动创建
    
        return START_NOT_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
运行结果：
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/bindservicedemo.png" width="500" height="60" style="margin:0 auto;display:block"/>


## AIDL
终于来到最专业干这事的地方了，AIDL：Android 接口定义语言（Android Interface Define Language）
代码实现起来，稍显复杂，我们还是按照上面那个 demo 来做：
建立 AIDL 文件（默认会生成一个示例方法）：
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/aidl1.png" width="550" height="330" style="margin:0 auto;display:block"/>
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/aidl2.png" width="320" height="200" style="margin:0 auto;display:block"/>

在这个IAppAidlInterface.aidl文件中，我们加上自己的接口：
`IAppAidlInterface.aidl`
```java
// IAppAidlInterface.aidl
package com.geng.aidldemo
// 注意：手动导入包
import com.geng.aidldemo.PeopleBean;

// Declare any non-default types here with import statements

interface IAppAidlInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
//    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
//            double aDouble, String aString);

    String setData(String packageName, int tag);

    List<PeopleBean> queryInfo(String name, int age);
}
```
`PeopleBean.aidl`
```java
// IAppAidlInterface.aidl
package com.geng.aidldemo;

// Declare any non-default types here with import statements
// 加入自己的bean,要实现parcelable
parcelable PeopleBean;
```
然后，我们将在APP A中AIDL相关目录 复制到App B中，PeopleBean 也是。**包名要保持一致**
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/aidl4.png" style="margin:0 auto;display:block"/>

* 注意：
**建立aidl文件后(或有文件更改)，需要clean/rebuild, 不然在实现aidl中接口时，会报错。**

下面开始我们的进程通信(这里，贴出主要的代码)：
**App A中：**
```java
public class AppService extends Service {

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;// 返回Stub
    }

    // 实现 aidl 定义的接口
    private final IAppAidlInterface.Stub mBinder = new IAppAidlInterface.Stub() {

        @Override
        public String setData(String packageName, int tag) throws RemoteException {
            System.out.println("AppService被调用： " + packageName + ", " + tag);
            // 。。。模拟各种操作
            return "AppService_" + packageName + "_" + tag;
        }

        @Override
        public List<PeopleBean> queryInfo(String name, int age) throws RemoteException {
            List<PeopleBean> list = new ArrayList<>();
            for (int i = 0; i < 10; i++) {
                PeopleBean peopleBean = new PeopleBean();
                peopleBean.name = "小米";
                peopleBean.sex = "女";
                peopleBean.age = i;
                list.add(peopleBean);
            }
            // ... 各种操作，忽略

            return list;
        }
    };
}
```
**App B中：**
```java
private ServiceConnection sc = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            isBinded = true;
            mIBAppService = IAppAidlInterface.Stub.asInterface(service);

            // 调用远程app 的方法
            try {
                String result = mIBAppService.setData(getPackageName(), 1000);
                System.out.println("BAppService 调用 AppService 结果： "+ result);

                List<PeopleBean> list = mIBAppService.queryInfo("小米", 3);
                System.out.println("查询people结果：");
                for (PeopleBean p : list) {
                    System.out.println(p.toString());
                }
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            isBinded = false;
            mIBAppService = null;
        }
    };
```
测试结果：
![](http://7xr1vo.com1.z0.glb.clouddn.com/aidl5.png)

我们再看看，中间的产物：
在 app A/B 的build中, 有IAppAidlInterface的接口文件产生。
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/aidl3.png" width="350" height="230" style="margin:0 auto;display:block"/>


**一个内部抽象类Stub实现了外部的接口，并继承了android.os.Binder，最内部的静态类Proxy，真正实现了2个接口方法，并进行序列化的操作**

`build/generated/source/aidl/IAppAidlInterface.java`
```java
public interface IAppAidlInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.geng.aidldemo.IAppAidlInterface {
        private static final java.lang.String DESCRIPTOR = "com.geng.aidldemo.IAppAidlInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.geng.aidldemo.IAppAidlInterface interface,
         * generating a proxy if needed.
         */
        public static com.geng.aidldemo.IAppAidlInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.geng.aidldemo.IAppAidlInterface))) {
                return ((com.geng.aidldemo.IAppAidlInterface) iin);
            }
            return new com.geng.aidldemo.IAppAidlInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_setData: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String _result = this.setData(_arg0, _arg1);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_queryInfo: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.util.List<com.geng.aidldemo.PeopleBean> _result = this.queryInfo(_arg0, _arg1);
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.geng.aidldemo.IAppAidlInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.lang.String setData(java.lang.String packageName, int tag) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(packageName);
                    _data.writeInt(tag);
                    mRemote.transact(Stub.TRANSACTION_setData, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public java.util.List<com.geng.aidldemo.PeopleBean> queryInfo(java.lang.String name, int age) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.geng.aidldemo.PeopleBean> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(name);
                    _data.writeInt(age);
                    mRemote.transact(Stub.TRANSACTION_queryInfo, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.geng.aidldemo.PeopleBean.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_setData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_queryInfo = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.lang.String setData(java.lang.String packageName, int tag) throws android.os.RemoteException;

    public java.util.List<com.geng.aidldemo.PeopleBean> queryInfo(java.lang.String name, int age) throws android.os.RemoteException;
}
```

总体来说，AIDL的方式写起来是比较麻烦，但是逻辑还是很简单的，**客户端 调用 服务端的方法实现**

## Socket
socket 通信是在传输层 TCP/UDP 之上，进行数据通信的，主要由 Socket, ServerSocket的相关api进行数据通信。
对于服务端 ServerSocket 持续响应处理，要开启多线程进行处理

```java
while(true){
    System.out.println("MultiThreadServer~~~监听~~~");
    //accept方法会阻塞，直到有客户端与之建立连接
    Socket socket = serverSocket.accept();
    ServerThread serverThread = new ServerThread(socket);
    serverThread.start();
}
```
这部分内容，可参考：
https://www.jianshu.com/p/fb4dfab4eec1

有些app, 利用socket，作为一个保活的手段，也是有点效果的

## 总结
* 多进程app可以在系统中申请多份内存，但应合理使用，建议把一些高消耗但不常用的模块放到独立的进程，不使用的进程可及时手动关闭
* messaseger / AIDL 使用场景
只有允许不同应用的客户端用 IPC 方式访问服务，并且想要在服务中处理多线程时，才有必要使用 AIDL。 如果您不需要执行跨越不同应用的并发 IPC，就应该通过实现一个 Binder 创建接口；或者，如果您想执行 IPC，但根本不需要处理多线程，则使用 Messenger 类来实现接口。

## 附录
Demo地址：
https://github.com/fenglincanyi/AndroidProject/tree/master/AIDLDemo

参考：
http://blog.csdn.net/u011459799/article/details/52012606
http://www.10tiao.com/html/227/201704/2650239403/1.html
http://blog.leanote.com/post/jay_richard/ContentProvider
http://yuanfentiank789.github.io/2017/04/28/messenger/
http://cjw-blog.net/2017/02/26/AIDL/
https://www.jianshu.com/p/fb4dfab4eec1
https://developer.android.com/guide/components/aidl.html?hl=zh-cn