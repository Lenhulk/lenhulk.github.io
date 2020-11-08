---
title: “@property”和它的关键字们
date: 2020-11-08 22:33:13
tags: [iOS,@property,关键字]
categories: iOS
---

{% cq %}
记录一些碎片，又能温故知新了。
{% endcq %}

<!-- more -->

## @property

**@property = ivar + getter + setter; **

> “属性” (property)有两大概念：ivar（实例变量）、存取方法（access method ＝ getter + setter）。

“属性” (property)作为 Objective-C 的一项特性，主要的作用就在于封装对象中的数据。 

Objective-C 对象通常会把其所需要的数据保存为各种实例变量。

实例变量一般通过“存取方法”(access method)来访问。其中，“获取方法” (getter)用于读取变量值，而“设置方法” (setter)用于写入变量值。

在正规的 Objective-C 编码风格中，存取方法有着严格的命名规范。 正因为有了这种严格的命名规范，所以 Objective-C 这门语言才能根据名称自动创建出存取方法。

例如下面这个类：

```
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@end
```

上述代码写出来的类与下面这种写法等效：

```
@interface Person : NSObject
- (NSString *)firstName;
- (void)setFirstName:(NSString *)firstName;
- (NSString *)lastName;
- (void)setLastName:(NSString *)lastName;
@end
```

**ivar、getter、setter 是如何生成并添加到这个类中的?**

> “自动合成”( autosynthesis)

完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫做“自动合成”(autosynthesis)。需要强调的是，这个过程由编译 器在编译期执行，所以编辑器里看不到这些“合成方法”(synthesized method)的源代码。除了生成方法代码 getter、setter 之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。在前例中，会生成两个实例变量，其名称分别为 _firstName 与 _lastName。也可以在类的实现代码里通过 @synthesize 语法来指定实例变量的名字.

```
@implementation Person
@synthesize firstName = _myFirstName;
@synthesize lastName = _myLastName;
@end
```



## 关键字

### 关键字分类&释义

#### 原子操作(2)

* **atomic**（原子性）[默认]：这个属性是为了保证程序在多线程下，编译器会自动生成自旋锁代码，避免该变量的读写不同步问题，提供多线程安全，即多线程中只能有一个线程对它进行访问。

  注意点：

  ```css
  atomic原子性指的是一个操作不可以被CPU中途暂停，然后再调度。即不能被中断，要么就执行完，要么就不执行
  atomic是自旋锁，当上一线程没有执行完毕的时候（被锁住），下一个线程会一直等待（不会进入睡眠状态），当上一线程任务执行完毕，下一线程立即执行。
  atomic只给setter方法上锁，getter不会加锁
  atomic需要消耗大量的资源，执行效率低
  一般Mac OSX开发中选择使用
  ```

* **nonatomic**（非原子性）：非原子性，非线程安全，多个线程可以同时对其进行访问，使用该属性编译器不会成加锁代码

  ```css
  一般情况下并不要求属性必须是“原子的”，因为这并不能保证“线程安全” (thread safety)，若要实现“线程安全”的操作，还需采用更为深层的锁定机制才行。例如，一个线程在连续多次读取某属性值的过程中有别的线程在同时改写该值，那么即便将属性声明为 atomic，也还是会读到不同的属性值。
  一般iOS开发中使用
  ```



#### 读写权限(2)

* **readwrite**（可读可写）[默认]：该关键字修饰的变量Xcode会为它自动生成getter/setter方法；
* **readonly**（只读属性）：只会生成getter方法



#### 内存管理(4)

* **assign**：适用于基本数据类型：NSInteger、CGFloat和C数据类型 int、float等；

* **strong**（对应MRC中的retain）：强引用，只有OC对象才能够使用该属性，它使对象的引用计数加1；

* **weak**：弱引用，只是单纯引用某个对象，但是并未拥有该对象。

  > 一般用于delegate，IBOutlet，以及防止*对象间循环引用*；

  > 一个对象只要没有被强引用，在一个runloop循环结束后就会被系统销毁

  > weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而 assign 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。

* **copy**：为减少对上下文的依赖而引入的机制，可以理解为内容的拷贝（深），内容被拷贝后，内存中会有两个存储空间存储相同的内容。指针指向的不是同一个内存地址；

  > NSString、NSArray、NSDictionary 等等经常使用copy关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary，他们之间可能进行赋值操作，为确保对象中的字符串值不会无意间变动，应该在设置新属性值时拷贝一份。 ； 

  > block 使用 copy 是从 MRC 遗留下来的“传统”,在 MRC 中,方法内部的 block 是在栈区的,使用 copy 可以把它放到堆区.在 ARC 中写不写都行：对于 block 使用 copy 还是 strong 效果是一样的



## 一些题外话

### runtime 如何实现 weak 属性

runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak 表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。

### 各属性参数的排列顺序

@property (nonatomic, readwrite, copy) NSString *name; 属性的参数应该按照下面的顺序排列： 原子性，读写 和 内存管理。

 这样做你的属性更容易修改正确，并且更好阅读。这在《禅与Objective-C编程艺术 >》里有介绍。

而且习惯上修改某个属性的修饰符时，一般从属性名从右向左搜索需要修动的修饰符。最可能从最右边开始修改这些属性的修饰符，根据经验这些修饰符被修改的可能性从高到底应为：内存管理 > 读写权限 >原子操作。