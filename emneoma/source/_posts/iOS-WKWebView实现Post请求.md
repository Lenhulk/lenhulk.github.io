---
title: iOS WKWebView实现Post请求
date: 2019-09-04 01:32:04
tags: [iOS,WKWebView,开发之术]
Categories: iOS
---

{% cq %}

（敲锣打鼓）ZF渠道的朋友又来搞事情了！

{% endcq %}

<!-- more -->


（听好了听好了）现在的需求是这样的！

> 通过我们向服务器发起一个A请求，服务器返回一个URL以及所有需要的参数，我们再通过WebView发起Post请求的方式（其中有重定向跳转）拿到最后的HTML信息。

因为UIWebView有内存泄漏和严重的性能问题，本着喜新厌旧的本性，项目中的WebView已经全面转了WKWebView。


# WKWebView fail to load post request

第一次尝试，自定义了SURLRequest，将方法改为post，将参数转码，甚至还自定义了content-type，然鹅请求却失败。

```objc
NSMutableURLRequest * requestShare = [[NSMutableURLRequest alloc]initWithURL:[NSURL URLWithString:self.urlStr]] 
[request setHTTPMethod:@"POST"];
[requset setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
[request setHTTPBody:[@"id=12&name=Jhon&token=skjasjlfjalsl" dataUsingEncoding:NSUTF8StringEncoding]]; //示例
[self.wkwebview loadRequest:request];
```


后来谷歌了..发现由于 WKWebView 在独立进程里执行网络请求。一旦注册 http(s) scheme 后，网络请求将从 Network Process 发送到 App Process，这样 NSURLProtocol 才能拦截网络请求。在 webkit2 的设计里使用 MessageQueue 进行进程之间的通信，Network Process 会将请求 encode 成一个 Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode 的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了！
[官方源码](https://github.com/WebKit/webkit/blob/fe39539b83d28751e86077b173abd5b7872ce3f9/Source/WebKit2/Shared/mac/WebCoreArgumentCodersMac.mm#L61-L88)

苹果工程师对此也作出了[回复](https://forums.developer.apple.com/thread/18952)，
回复文章中，苹果工程师还非常热心的告诉开发者，哈哈你的UIWebView能够发送Post请求也只是一个意外，我们根本就没想让你们这样玩！WebView只能通过GET请求建立，要不你们尝试用JavaScript发post请求耍耍呗。


# JS大法好

OC的不足就让JS来补足吧（怪押韵的？）

{% note info %}
我们可以利用JS来动态的生成HTML里的form标签，然后用form表单来实现POST请求。
而且WKWebView即使不加载任何html和request，也可以直接执行js方法（吊炸天了）。
{% endnote %}

1. **我们将JS代码定义成一个即插即用的宏方法。**

```objc
#define POST_JS @"function my_post(path, params) {\
var method = \"POST\";\
var form = document.createElement(\"form\");\
form.setAttribute(\"method\", method);\
form.setAttribute(\"action\", path);\
for(var key in params){\
    if (params.hasOwnProperty(key)) {\
        var hiddenFild = document.createElement(\"input\");\
        hiddenFild.setAttribute(\"type\", \"hidden\");\
        hiddenFild.setAttribute(\"name\", key);\
        hiddenFild.setAttribute(\"value\", params[key]);\
    }\
    form.appendChild(hiddenFild);\
}\
document.body.appendChild(form);\
form.submit();\
}"
```

2. **发送请求！**

```objc
// url - 请求发送的地址
// paramStr - 要传递的参数,格式需要转成合法的json字符串,如：@"{\"token\":\"cac6af340960485aa334416c8a34ddbf\"}";
// js - 需要wkwebView执行的JS代码，post一份form表单

NSString * js = [NSString stringWithFormat:@"%@my_post(\"%@\", %@)",POST_JS,url,dataStr];
[self.wkwebview evaluateJavaScript:js completionHandler:nil];
```

嗯嗯，这样就太爽了。