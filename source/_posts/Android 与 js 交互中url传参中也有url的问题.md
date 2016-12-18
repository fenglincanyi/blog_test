---
title: Android 与 js 交互中url传参中也有url的问题
date: 2016-12-15 13:29
categories: Hybird
tags: JS
---
### 问题场景
get请求链接中： 传参中含有url

"chinahr://customer/h5page?" + LEOGAO_PAGE+"="+finishpage+"&"+LEOGAO_ACTION+"="+finishaction + "&"+INTENT_URL+"=" + <b><font color="red">url</font></b>

其中：
<b><font color="red">url：</font></b>  file:///android_asset/hybird/cp/index.html?uid=" + cuid +"&gender="+gender +"&photo="+photoPath;

合并之后：
"chinahr://customer/h5page?" + LEOGAO_PAGE+"="+finishpage+"&"+LEOGAO_ACTION+"="+finishaction + "&"+INTENT_URL+"="+“file:///android_asset/hybird/cp/index.html?uid=" + cuid +"&gender="+gender +"&photo="+photoPath；

所以这个 url 中的 第二个及以后的参数第一次就会被router解析，并不会将 url 整体作为一个参数传到下一个页面中去再次解析识别。
### 解决方法
#### Android端
```
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
#### 前端js
```
var me = this;
var href = window.location.url || window.location.href;
var params = me.getParams(href, 'params');
params = Base64.decode(decodeURIComponent(params));
info = JSON.parse(params);
var uid = info.uid;
var photo = info.photo;
var gender = info.gender;
```
### 踩坑
第一次使用UrlEncoder编码，k可能会造成浏览器与js解析数据会存在不一致的问题
解决：先使用base64进行编码，解决数据不一致的问题，再进行UrlEncoder编码（为了防止base64编码后的数据解析时："+" 会当做空格的问题 ） 
### 补充
在线加解密、编码解析工具：
http://tool.oschina.net/encrypt?type=3