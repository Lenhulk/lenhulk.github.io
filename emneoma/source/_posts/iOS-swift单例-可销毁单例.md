---
title: iOS-swift单例&可销毁单例
date: 2020-04-01 23:54:45
tags: [iOS,swift,Singleton]

---

<div class="note primary"><p>这可能是你学过的第一种设计模式</p></div>

<!-- more -->

# 单例相关

> 单例模式（Singleton Patten）是最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。

一般情况下,使用"shared"  "sharedInstance"  "default"  "standard"等字样的,就是单例.

## 基本要求

- 只能有一个实例
- 必须内部自己创建自己的唯一实例
- 必须对所有其它对象提供该唯一实例

## iOS 中的单例

- `UIApplication.shard` ：每一个应用程序启动后创建的第一个单例对象就是UIApplication对象，利用UIApplication对象能进行一些应用级别的操作

- `NotificationCenter.defualt`：通知中心，管理应用中的通知

- `FileManager.defualt`：获取沙盒主目录的路径

- `URLSession.shared`：管理网络连接

- `UserDefaults.standard`：存储轻量级的本地数据

- `SKPaymentQueue.default()`：管理应用内购的队列。系统会用 **StoreKit** framework 创建一个支付队列，每次使用时通过类方法 `default()` 去获取这个队列。

## 单例的优缺点

{% label success@优点： %}

- **提供了对唯一实例的受控访问**：单例类封装了它的唯一实例，防止其它对象对自己的实例化，确保所有的对象都访问一个实例。

- **节约系统资源**：由于在系统内存中只存在一个对象，因此可以节约系统资源，对于一些需要频繁创建和销毁的对象，单例模式无疑可以提高系统的性能。

- **伸缩性**：单例模式的类自己来控制实例化进程，类就在改变实例化进程上有相应的伸缩性。

- **避免对资源的多重占用**：比如写文件操作，由于只有一个实例存在内存中，避免对同一个资源文件的同时写操作

{% label danger@缺点： %}

- **状态混乱**：由于单例是共享的，可能有多个对象都会访问及改变单例的状态，所以当使用单例时，程序员无法清楚的知道单例当前的状态。

- **测试困难**：测试困难主要是由于单例状态的混乱而造成的。因为单例的状态可以被其他共享的实例所修改，所以进行需要依赖单例的测试时，很难从一个干净、清晰的状态开始每一个 test case
- **访问混乱**：由于单例是全局对象，所以无法对访问权限作出限定。程序任何位置、任何实例都可以对单例进行访问，容易引起访问混乱。



# Objective-C单例

1. 在.h文件中开放单例类方法的接口：

   ```objc
   + (instanceType)sharedInstance;
   ```

   

2. 在.m文件中声明一个静态对象，善用GCD实现单例：

   ```objc
   static MySingleton *sharedInstance = nil;
   + (instanceType)sharedInstance {
     static dispatch_once_t once;
     disptach_once(&once, ^{
       sharedInstance = [[self alloc] init];
     });
     return sharedInstance;
   }
   ```

   

# Swift单例

## 实现方式

### 第一种方式：

最简洁的方式，将实例定义为全局变量。

> 如果将上述实例变量在全局命名区（global namespace）第一次调用，由于Swift中**全局变量是懒加载（lazy initialize）**。
>
> 所以，在`application(_:didFinishLaunchingWithOptions:)`中调用的时候之后，`shardManager`会在`AppDelegate`类中被初始化，之后程序中所有调用`sharedManager`实例的地方将都使用该实例。

> 另外，Swift 全局变量初始化时默认使用`dispatch_once`，这保证了全局变量的构造器（initializer）只会被调用一次，保证了`shardManager`的**原子性**。

```swift
// 在全局命名区创建实例
let sharedSingleton = MySingleton(string: someString)
class MySingleton {
  // 属性
  let string: String
  // 初始化方法
  init(string: String) {
    self.string = string
  }
}
```

```swift
// 外部直接使用sharedInstance
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    // 初始化位置，以及使用方式
    print(sharedSingleton)
    ...
    return true
}
```



### 第二种方式：（使用最多的方式）

使用`static`、`private`关键字，限定变量的作用域。

“static”关键字使`shard`成为全局变量，“private”关键字保证了单例的原子性。

```swift
class MySingleton {
	// 全局变量
  static let shared = MySingleton(string: someString)
  // 属性
  let string: String
  // 初始化方法
  private init(string: String) {
    self.string = string
  }
}
```

```swift
// 使用
MySingleton.shared
```

**采用第二种方式实现单例，代码的可读性增加了，能够直观的分辨出这是一个单例**。



### 第三种方式：

第三种方式是第二种方式的变种，更加复杂。

让单例在闭包（Closure）中初始化，同时加入类方法来获取单例。

```swift
class MySingleton {
  // 全局变量
  private static let sharedInstance: MySingleton = {
    let shared = MySingleton(string: someString)
    // ...其它配置
    return shared
  }()
  // 属性
  let string: String
  // 初始化
  private init(string: String) {
    self.string = string
  }
  // 对外接口
  class func shared() -> MySingleton {
    return sharedInstance
  }
}
```

可以看出第三种方式虽然更加复杂，但是可以在闭包中作一些额外的配置。同时，调用单例的方式也不一样，需要调用单例中的类方法`shared()`

```swift
// 使用
MySingleton.shared()
```



## 可销毁单例

虽说单例是全局唯一，但是遇到需要限定单例的作用域在某个库，某个框架里，当pop回到主工程，离开私有框架作用域时想要让单例销毁，下次进入再初始化。可以这样做：

```swift
class MySingleton {
  // 全局变量
  private static var _sharedInstance: MySingleton?
  // 属性
  let string: String
  // 初始化
  private init(string: String) {
    self.string = string
  }
  // 使用
  class func shared() -> MySingleton {
    guard let instance = _sharedInstance else {
      _sharedInstance = MySingleton(string: someString)
      return _sharedInstance
    }
    return instance
  }
  // 销毁
  class func destroy() {
    _sharedInstance = nil
  }
}
```

```swift
//使用方式
MySingleton.shared()
MySingleton.destroy()
```



