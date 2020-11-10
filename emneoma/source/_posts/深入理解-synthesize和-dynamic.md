---
title: 深入理解@synthesize和@dynamic
date: 2020-11-10 12:29:19
tags: [iOS,sythesize,dynamic]
Categories: iOS

---

{% cq %}

@synthesize和@dynamic这对小伙伴也需要好好剖析呢！

{% endcq %}

<!-- more -->

## 核心规则

* @property有两个对应的词，一个是@synthesize，一个是@dynamic。
* 如果@synthesize和@dynamic都没写，那么默认的就是`@syntheszie var = _var;`。



### synthesize

* **@synthesize的语义是合成，如果你没有手动实现setter方法和getter方法，那么编译器会自动为你加上这两个方法。** 
* 在Xcode4.0之后编译器会默认为你的属性加上@synthesize，因此我们可以认为，`synthesize`是`property`属性的一部分，它是系统为`property`生成变量的重要步骤。
* 简而言之，它的作用就是自动合成 *_ivar+getter+setter*。



**一个过时的完整setter&getter写法**

```Objc
@interface Sample()

@property(nonatomic, strong, readwrite)NSObject *sampleObject;
@end
@implementation Sample
@synthesize sampleObject = _sampleObject;

//getter
- (NSObject *)sampleObject
{
    return _sampleObject;
}

//setter
- (void)setSampleObject:(NSObject *)sampleObject
{
    _sampleObject = sampleObject;
}
@end
```

上述代码就是以前的写法，可以更直观的理解到synthesize默认合成的成员变量会自动在ivar前加上下划线，在重写getter&setter的时候，需要操作的是该带下划线的成员变量，property只是定义了该成员变量的属性名，原子操作和内存管理等特性。



### dynamic

* **@dynamic与@synthesize相反，它告诉编译器,属性的setter与getter方法由用户自己实现，不自动生成。**（当然对于readonly的属性只需提供getter即可）。

* 假如一个属性被声明为@dynamic var，然后你没有提供@setter方法和@getter方法，编译的时候没问题，但是当程序运行到instance.var =someVar，或者当运行到 someVar = var时，对应参数的getter和setter方法就会在其他地方去寻找，比如**父类**。最后当找不到对应方法，由于缺getter/setter方法同样会导致崩溃。

* 用到`@dynamic`的地方很少：

  0. 用于声明我要手动实现setter/getter方法，提高代码可读性。

  1. 子类中的重载参数可以设为@dynamic，实现和父类共用一个参数；

  2. 动态生成类和变量的情景（如动态model生成）。



## 重点补充

### @synthesize 合成实例变量的规则

- 如果指定了成员变量的名称，会生成一个指定的名称的成员变量 `@synthesize foo = _foo;`。如果这个成员已经存在了就不再生成了；
- 如果手动写成` @synthesize foo;` 会生成一个名称为 foo 的成员变量，也就是说：如果没有指定成员变量的名称会自动生成一个**属性同名的成员变量**；
- 假如 property 名为 foo，同时还存在一个名为 _foo 的实例变量，则不会自动合成新变量。并且Xcode会报警告。



### synthesize什么情况下会用？

正常情况下，你不需要使用的`synthesize`，因为大多数情况下，系统都会为你自动合成。

但是，你必须能清楚，系统自动合成有时候也是会失效的。这时候就需要你手动添加 `synthesize`。

这些情况大约有3种：

- 手动修改生成的成员变量名字（不建议）。
- 手动实现了 `setter/getter` 俩方法。

> 如果你的属性是只读属性，但是你重写了`getter`方法，系统不会为你自动生成成员变量。你需要添加`@synthesize`。

> 如果你的属性可读可写，但是你同时重写了`setter/getter`方法，系统不会为你自动生成成员变量。你需要添加`@synthesize`。这种情况下，你如果只重写了`setter/getter`其中一个，系统仍然会执行自动合成。

- 实现了带有 `property` 属性的 `protocol`。



### 什么情况下不会 autosynthesis（自动合成）？

- 属性同时重写了 setter 和 getter 时；
- 重写了只读属性的 getter 时；
- 使用了 @dynamic 时
- 在 @protocol 中定义的所有属性
- 重载的属性



