---
title: iOS - 修改导航栏透明度
date: 2019-04-20 12:50:33
tags: iOS_Tricks
categories: iOS
---

{% blockquote %}
根据iOS版本的不同，拿到某个子对象修改透明度。
{% endblockquote %}

{% codeblock lang:objc %}
[[[self.navigationController.navigationBar subviews] objectAtIndex:0] setAlpha:0.5]; 
{% endcodeblock %}

<!-- more -->
---

## 在以前 ##

我们可以通过当前控制器的_NavigationController_再获取到*navigatioBar*导航条，修改导航条的*barTintColor*属性去修改它的透明度：

``` objc
self.navigationController.navigationBar.barTintColor = [UIColor colorWithWhite:1.0 alpha:0.5];
```

但如果是沉浸式背景，那么状态栏statusBar的白色/黑色背景又会和导航栏出现颜色不同，因此我们还得同时修改statusBar的背景颜色：

``` objc
UIView *statusBar = [[[UIApplication sharedApplication] valueForKey:@"statusBarWindow"] valueForKey:@"statusBar"];
if ([statusBar respondsToSelector:@selector(setBackgroundColor:)]) {
    statusBar.backgroundColor = [UIColor colorWithWhite:1.0 alpha:0.5];
}
```

---

## iOS10之后 ##

*barTintColor*设置半透明颜色不起作用了。

可以用另外一种设置方式：

``` objc
//找到navigationBar的背景view，然后设置alpha值，这样会连statusBar背景一并修改了
[[[self.navigationController.navigationBar subviews] objectAtIndex:0] setAlpha:0.5];        
```

如果不起作用 或者 别的控制器将导航栏上面的东西改了回去… 
我们可能需要设置下面两个参数
在viewWillAppear方法中加上这两行代码，再结合上面的方法修改透明度。

``` objc
- (void)viewWillAppear{
  [super viewWillAppear];

  // 设置导航栏为有点透明的效果，注意如果设置了这个参数，控制器中有scrollView的可能会向上偏移64px，也就是回到默认会被导航栏遮住的位置
  self.navigationController.navigationBar.translucent = YES;  
  // 将导航条背景改为透明
  self.navigationController.navigationBar.backgroundColor = [UIColor clearColor];
}       
```

最终效果：

<!-- {% qnimg 201901/yhdm.png 'class:' extend:?imageMogr2/thumbnail/450x1000/interlace/1/blur/1x0/quality/100|watermark/2/text/RU1ORU9NQS5YWVo=/font/YXJpYWw=/fontsize/240/fill/IzdCNjhFRQ==/dissolve/90/gravity/SouthEast/dx/10/dy/10 %} -->
{% qnimg 201901/yhdm.png 'class:' extend:-w375 %}