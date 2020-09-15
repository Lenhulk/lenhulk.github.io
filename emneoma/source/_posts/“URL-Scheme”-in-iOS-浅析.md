---
title: “URL Scheme” in iOS 浅析
date: 2020-09-15 23:04:10
tags: [iOS,URL Scheme]
Categories: iOS
---

{% cq %}

做一个反复横跳的魔法师🧙‍♀️

{% endcq %}

<!-- more -->



## 什么是URL Scheme

在苹果的生态中，存在沙盒机制来保障用户的隐私和app间的信息安全。但是应用之间不可能毫无数据交流和业务跳转，因此`URL Scheme`就是打破这层隔膜的工具。通过`URL Scheme`，app可以跳转或者唤起另一个app，拉起分享，支付等业务功能，当然你也可以给自己的app定制一个专属`URL Scheme`让别的app和你进行互动。



## URL Scheme 浅析

`URL`：统一资源定位符；`URL`地址格式排列为：`scheme://host:port/path` 。

类似的，一个 URL Schemes 分为 Scheme、Action、Parameter、Value 这 4 个部分，中间用冒号 `:`、斜线 `/`、问号 `?`、等号 `=` 相连接。

Scheme这个东西在 URL 里本来用来表示协议，比如 `http`、`https`、`ftp` 等。但是，在 iOS 的这些 URL 里它们就不是协议了，而是**启动一个应用的 URL**。

链接头和斜线之后，跟的是动作（Action）。动作在一些文档里也被称为「Command（命令）」。

在动作之后可以接参数（Parameter）也可以不接，「参数以问号 `?` 为起始」，所以不接参数就不用带这个问号。

（这一套机制也因为灵活高效方便而受众于各iOS组件化开发解决方案） 

**通用的规则**：

- 冒号`:`：在**链接头**和**命令**之间；

- 双斜杠 `//`：在**链接头和命令**之间，有时会是三斜杠 `///`；
- 问号 `?`：在**命令和参数**之间；
- 等号 `=`：在**参数和值**之间；
- 「和」符号 `&`：在**一组参数和另一组参数**之间。

**Tips**：

我们可以直接用safari 访问：*URL Scheme*://，就可以唤醒对应的 APP，可以用此来测试是否给自己的app配制正确。



## URL Scheme 跳转使用

以跳转支付宝进行支付为例：

1. 在infor.plist文件中添加 **LSApplicationQueriesSchemes** 数组，直接点击“+”号，在value一栏填上支付宝的Scheme——“alipay”；

> `LSApplicationQueriesSchemes`：是在iOS 9 以后苹果新增的一个参数，这个类别增加之后，可以使用`UIApplication`的`canOpenURL:`方法来判断是否安装了这个 URL scheme 对应的 APP。否则会报错提示：`-canOpenURL: failed for URL: "xxx://" - error: "This app is not allowed to query for scheme xxx"`；如果直接调用跳转代码跳转到不存在的app 也会崩溃。

2. **UIApplication - canOpenURL: **判断我们的目标app是否有安装在本机

3. **UIApplication - openURL: **来拉起对应的app

> iOS 10中，该方法被`openURL:options:completionHandler:`方法替代。



## 定义自己URL Scheme 让其它app跳转

URL Scheme必须能唯一标识一个APP，如果你设置的URL Scheme与别的APP的URL Scheme冲突时，你的APP不一定会被启动起来。因为当你的APP在安装的时候，系统里面已经注册了你的URL Scheme。

一般情况下，是会调用先安装的app。但是iOS的系统app的URL Scheme肯定是最高的。所以我们定义URL Scheme的时候，尽量避开系统app已经定义过的URL Scheme。

操作步骤：

- 在info.plist文件中添加 **CFBundleURLTypes** 数组，给每一个item的`CFBundleURLSchemes`填值，同时可以给他们一个标示在`CFBundleURLName`里。

- 如果直接操作info.plist文件不明白，可以按这样的步骤添加：

  > 点击工程名->target->Info->URL Types->点击“+”号*，
  >
  > 然后按需填写`URL Scheme`等参数，再回头看info文件就有值了。

当你完成了这些，其它app就可以根据你的scheme拉起你的app。

### 拉起app后的操作

在`AppDelegate`中实现系统方法 **application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options**

就可以解析传递过来的URL Scheme ，定制你要拉起的页面 发送的请求 传递的请求参数 及回调。



## iOS10以上系统常用URL Scheme

**设置页面**  App-Prefs:root

(之前在那个设置页面，就跳转到相应的设置页面)

**无线局域网**  App-Prefs:root=WIFI

**蓝牙**  App-Prefs:root=Bluetooth

**蜂窝移动网络**  App-Prefs:root=MOBILE_DATA_SETTINGS_ID

**个人热点**  App-Prefs:root=INTERNET_TETHERING

**运营商**  App-Prefs:root=Carrier

**通知**  App-Prefs:root=NOTIFICATIONS_ID

**通用**  App-Prefs:root=General

**通用-关于本机**  App-Prefs:root=General&path=About

**通用-键盘**  App-Prefs:root=General&path=Keyboard

**通用-辅助功能**  App-Prefs:root=General&path=ACCESSIBILITY

**通用-语言与地区**  App-Prefs:root=General&path=INTERNATIONAL

**通用-还原**  App-Prefs:root=Reset

**墙纸**  App-Prefs:root=Wallpaper

**Siri**  App-Prefs:root=SIRI

**隐私**  App-Prefs:root=Privacy

**Safari**  App-Prefs:root=SAFARI

**音乐**  App-Prefs:root=MUSIC

**音乐-均衡器**  App-Prefs:root=MUSIC&path=com.apple.Music:EQ

**照片与相机**  App-Prefs:root=Photos

**FaceTime**  App-Prefs:root=FACETIME