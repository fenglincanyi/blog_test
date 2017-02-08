---
title: Android WebView 相关踩坑
date: 2016-12-15 13:29
categories: Hybird
tags: h5
---
### url 传参 url 嵌套的问题
#### 问题场景
get请求链接中： 传参中含有url

"chinahr://customer/h5page?" + LEOGAO_PAGE+"="+finishpage+"&"+LEOGAO_ACTION+"="+finishaction + "&"+INTENT_URL+"=" + <b><font color="red">url</font></b>

其中：
<b><font color="red">url：</font></b>  file:///android_asset/hybird/cp/index.html?uid=" + cuid +"&gender="+gender +"&photo="+photoPath;

合并之后：
"chinahr://customer/h5page?" + LEOGAO_PAGE+"="+finishpage+"&"+LEOGAO_ACTION+"="+finishaction + "&"+INTENT_URL+"="+“file:///android_asset/hybird/cp/index.html?uid=" + cuid +"&gender="+gender +"&photo="+photoPath；

所以这个 url 中的 第二个及以后的参数第一次就会被router解析，并不会将 url 整体作为一个参数传到下一个页面中去再次解析识别。
#### 解决方法
##### Android端
``` java
JSONObject jsonObject = new JSONObject();
try {
    jsonObject.put("uid", cUid);
    jsonObject.put("gender", gender);
    jsonObject.put("photo", potoPath);
    paramsResult =  URLEncoder.encode(Base64.encodeToString(jsonObject.toString().getBytes(), Base64.DEFAULT));
} catch (JSONException e) {
    e.printStackTrace();
}
// 此处为加载本地，也可以为网络url
BaseH5Activity.startWebViewActivity(this, "file:///android_asset/hybird/cp/index.html?params=" + paramsResult, "","","","");
```
##### 前端js
``` javascript
var me = this;
var href = window.location.url || window.location.href;
var params = me.getParams(href, 'params');
params = Base64.decode(decodeURIComponent(params));
info = JSON.parse(params);
var uid = info.uid;
var photo = info.photo;
var gender = info.gender;
```
##### 踩坑
* 第一次使用UrlEncoder编码，k可能会造成浏览器与js解析数据会存在不一致的问题
解决：先使用base64进行编码，解决数据不一致的问题，再进行UrlEncoder编码（为了防止base64编码后的数据解析时："+" 会当做空格的问题 ） 
* 加载本地 html 时：直接设置 cookie 在 本地url 上是无效的，也是没有必要的。自己遇到的问题是：加载本地 html，此html中使用的 jsonp 请求网络，此时需要的cookie，需要设置 WebView 的cookie，使之持久化，下次会自动带 cookie 访问网络的。
### Android 5.0 WebView 设置cookie问题

* 对于 Android 5.0 以上的WebView，默认关闭了接收三方cookie，但是提供了设置 cookie的接口，需要开发者去手动设置三方信任cookie。否则，加载本地 html 时，cookie 同步不过去。
代码如下：

``` java
CookieManager cookieManager = CookieManager.getInstance();
cookieManager.setAcceptCookie(true);
cookieManager.setAcceptFileSchemeCookies(true);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    cookieManager.setAcceptThirdPartyCookies(webView, true);
}
```
### Android WebView 加载本地 html

* webview 加载 sd 卡下的 html 是不能访问的，权限问题。
* 如果访问 /data/data/包名/files/index.html，需要开启文件访问权限（默认是开启的，不要动态设置关闭）
``` java
webSettings.setAllowFileAccess(false);// 关闭
```

### 补充
在线加解密、编码解析工具：
http://tool.oschina.net/encrypt?type=3