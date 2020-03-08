---
title: assign VS weak? __weak VS __block?
date: 2019-06-11 01:10:44
tags: 
- iOS
- 修饰符
- 基础知识
categories: iOS
---



<div class="note info"><p>小知识点。</p></div>



<!-- more -->

# assign #

**assign** 用于基础类型的赋值，不改变属性的引用计数。如：*NSInteger, CGFloat, int float double*

{% blockquote %}
assign其实也可以用来修饰对象，但是你可不要轻易尝试。因为被assign修饰的对象在释放之后，指针的地址还是存在的，也就是说指针并没有被置为nil。如果在后续的内存分配中，刚好分到了这块地址，程序就会崩溃掉。
{% endblockquote %}

# weak #
**weak** 用于对象类型，不改变属性的引用计数，当该对象被释放的时候，该弱引用的属性自动失效并且被赋值为***nil***，该属性可以避免循环引用问题。

# __weak #
**__weak**是所有权修饰符，被修饰的变量在使用结束后会被释放 置nil，其也不会在block代码块中被retain。
所有权修饰符包括: \_\_strong,  \_\_weak,  unsafe_unretained,  \_\_autorealease。

{% blockquote %}
使用__unsafe_unretained 和 __weak都可以避免循环引用的问题，但由于前者是在iOS4.0以前__weak的替代品，不会被自动置nil，会造成野指针问题，所以我们果断抛弃吧
{% endblockquote %}

# __block #
**__block**用于指明当前变量可以在block内部进行修改（ARC下会被retain）。

{% blockquote %}
因为在block申明的同时会捕获该block所使用的全部自动变量的值，仅有使用权没有修改权利，使用了__block关键字修饰后的变量可以在block内部进行修改。
{% endblockquote %}

在block内，要避免循环引用要使用：
``` objc
__weak__typeof(self)weakSelf =self;
```

并且，在使用到self之后的对象或者属性防止在使用之前被析构引发不可预测的问题，所以要使用strong再把它持有一下。
``` objc
 __strong__typeof(weakSelf)strongSelf = weakSelf;   
```

## 易混淆点 ##
若 object 本身沒有去 retain 这个 block (即block不是某个对象的property)，則可以直接在 block 中使用 self。
比如自定义的block块（一个匿名函数）在代码中执行；经常问到的animation动画是否需要使用weakblock的问题。
<div class="note default">事实上大多數的 iOS 原生套件，以及 GCD 的 block 是不會造成 retain cycle 的，因为他们并沒有去 retain block。</div>

特别要注意的是，讲一个变量直接定义为实例变量而非属性的时候，在block中使用时还是会retain到self导致循环引用，因为ivar也是self的一部分：
``` objc
@interface XXXViewController () {
    NSString *str;
}
self.completionHandler = ^{
    // 直接引用就相当于 self->str
    NSLog(@"%@", str);
}
```
但是！ivar是无法用weakSelf去取值的，因此
这里正确的做法还是要用到weakSelf和strongSelf的帮助：
``` objc
__weak __typeof(self) weakSelf = self;
self.completionHandler = ^{
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    NSLog(@"%@", strongSelf->str);
}
```

