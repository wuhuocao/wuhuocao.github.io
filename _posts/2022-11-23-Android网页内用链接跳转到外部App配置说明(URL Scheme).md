---
title: Android网页内用链接跳转到外部App配置说明(URL Scheme)
author: 
date: 2022-11-23 11:40:38 +0800
categories: [Blogging, Learn]
tags: [andriod, webview]
pin: false
---

## 概要

1. <a href="#title_h5">H5页面配置</a>

2. <a href="#title_app">App应用配置</a>

3. <a href="#title_brower">浏览器及系统跳转解析</a>

4. [总结](#title_summary)

---

## <a id="title_h5">H5页面配置</a>


```xml
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>网页跳转外部App</title>
</head>

<body>
    <input type="text" id="url" value="zhihu://question/19581624">
    <button onclick="javascript:jump2app()">点击跳转外部App</button>
    <script>
        function jump2app() {
            var url = document.getElementById("url").value
            window.location.href = url
        }
    </script>
</body>

</html>
```


注：除了window.location.href=url外，还可以用window.open(url)或者超链接<a>跳转

---

## <a id="title_app">App应用配置</a>

自定义协议链接跳转App页面，需在文件src\main\AndroidManifest.xml 配置如下：

```xml
<activity android:name=".MainActivity" android:exported="true">
    <intent-filter >
        <action android:name="android.intent.action.VIEW" /> <!--必须-->
        <category android:name="android.intent.category.DEFAULT" /> <!--必须-->
        <category android:name="android.intent.category.BROWSABLE" /> <!--必须-->
        <data android:scheme="zhihu"/> <!--自定义协议如：zhihu-->
    </intent-filter>
</activity>
```

注：当发送Intent的data=zhihu:// 时，才能跳转到App的MainActivity页面。

发送Intent命令可用：adb shell am start -a android.intent.action.VIEW -d "zhihu://"

通过以上两个配置，用chrome等安卓浏览器访问以上H5页面文件，点击网页内的“点击跳转外部App”就会跳转到已安装好的MainActivity页面。

---

当然也可以自定义http/https跳转，如下：

```xml
<intent-filter android:autoVerify="true"> <!--必须-->
    <action android:name="android.intent.action.VIEW" /> <!--必须-->
    <category android:name="android.intent.category.DEFAULT" /> <!--必须-->
    <category android:name="android.intent.category.BROWSABLE" /> <!--必须-->
    <data android:scheme="http" android:host="www.zhihu.com"/> <!--自定义-->
    <data android:scheme="https"/>
</intent-filter>
```

自定义http跳转的应用在安装后可用以下命令查看系统配置状态

> adb shell dumpsys package domain-preferred-apps
> 
> 命令输出如下
> 
> Package: com.example.testapi
> Domains: www.zhihu.com
> Status:  ask
> 
> Package: com.zhihu.android  
> Domains: activity activity.zhihu.com [www.zhihu.com](http://www.zhihu.com) promotion.zhihu.com oia.zhihu.com zhuanlan.zhihu.com ms.zhihu.com
>  Status: ask

---

为了验证http和App之间跳转的官网归属性，系统会做官网域名配置文件与App签名的校验操作，如下所示：

- 官网域名配置文件：https://www.zhihu.com/.well-known/assetlinks.json

```json
[
    {
        "relation":[
            "delegate_permission/common.handle_all_urls"
        ],
        "target":{
            "namespace":"android_app",
            "package_name":"com.zhihu.android",
            "sha256_cert_fingerprints":[
                "BD:84:50:55:7C:B3:96:5C:05:1F:16:11:D4:28:6A:5F:02:9B:90:9C:AE:3D:E7:57:EC:15:2D:05:63:C3:F7:FA"
            ]
        }
    }
]
```

- App签名信息：

> #keytool -printcert -jarfile zhihu_8.38.0.apk
> 
> 签名者 #1:
> 
> 签名:
> 
> 所有者: CN=Zhihu, OU=zhihu.com, O=zhihu.com, L=Peking, ST=Peking, C=CN
> 发布者: CN=Zhihu, OU=zhihu.com, O=zhihu.com, L=Peking, ST=Peking, C=CN
> 序列号: 51821772
> 有效期开始日期: Thu May 02 15:36:18 CST 2013, 截止日期: Mon Apr 26 15:36:18 CST 2038
> 证书指纹:
>  MD5: 5C:4F:61:85:36:EA:F9:AE:0E:26:28:C5:AF:16:93:BC
>  SHA1: B6:F9:97:E3:82:7B:E1:1A:F2:FA:4A:15:3F:EA:3F:E6:27:68:66:02
>  SHA256: BD:84:50:55:7C:B3:96:5C:05:1F:16:11:D4:28:6A:5F:02:9B:90:9C:AE:3D:E7:57:EC:15:2D:05:63:C3:F7:FA
>  签名算法名称: SHA1withRSA
>  版本: 3

注：以上官网域名配置文件的sha256_cert_fingerprints与 App签名的SHA256值是一样的

有些定制的系统，并不遵守以上归属规则。当一个域名对应多个App时，系统会让用户做出选择，而不是直接跳转到官网指定的App页面。

---

## <a id="title_brower">浏览器及系统跳转解析</a>

系统原生浏览器内核包名一般是：com.google.android.webview或者com.android.webview，大部分是Native层代码，应该是用来渲染网页、网络加载等功能，感兴趣可访问文末参考链接。

一般启动App的操作是浏览器实现的，且在浏览器地址栏内输入http等链接是不做跳转app检测的，跳转app检测一般是在网页内跳转时处理的。以chrome安卓浏览器（com.android.chrome[version:51.0.2704.90]）为例：

Java将url传递到Native层，Native层处理(包括javascript重定向url)后再通过JNI回调给Java层，并在如下类方法进行判断并启动App

org.chromium.chrome.browser.externalnav.ExternalNavigationHandler.shouldOverrideUrlLoadingInternal()

所以，App内嵌WebView开发时，默认是不能进行特殊链接跳转，需要自己重写以下方法处理。

android.webkit.WebViewClient#shouldOverrideUrlLoading()

android.webkit.WebViewClient#shouldInterceptRequest()

## <a id="title_summary">总结</a>

调试了解chrome应用时，大概看了chromium的源码仓库，各模块根目录有相应的说明和结构文档，值得借鉴。比如//android_view目录是给上层提供Java APIS用于加载chromium内核的代码，//android_webvuew/glue提供AW前缀的Java类用于调用C++，//content/public用于C++层调用Java层。

总的来说实现该功能挺简单的，但底层还包含了很多细节。该功能也便利了跨端的应用交互。

---

参考链接：

[app-links](https://developer.android.com/training/app-links)

[Chromium](https://www.chromium.org/Home/)

[WebView docs (go/webview-docs) - WebView Build Instructions](https://chromium.googlesource.com/chromium/src/+/HEAD/android_webview/docs/build-instructions.md)

[webview_architecture](https://source.chromium.org/chromium/chromium/src/+/main:android_webview/docs/architecture.md)
