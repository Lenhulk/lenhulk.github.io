---
title: iOS中的遍历方法与技巧
date: 2020-11-21 23:03:54
tags: [iOS,遍历]
Categories: iOS
---

<div class="note primary"><p>这可能是你学过的第一种函数</p></div>

<!-- more -->

# ios中的遍历方法

## 经典for循环

又叫“C循环”，也就是（for/while） 方法。

```objc
for (NSUInteger i = 0; i < [array count]; i++) {
  id object = array[i];
  NSLog(@"%@", object)
}
```

但是用过 C 风格循环的人也都知道，这个方法容易导致 [差一错误](https://zh.wikipedia.org/wiki/差一错误)，当使用经典for做遍历删除操作的时候也会导致崩溃，同时它也是一个单线程的方法。

因此，Smalltalk 使用一种叫做 [列表生成式](https://en.wikipedia.org/wiki/List_comprehension) 的方法改善了这个问题，也就是大家今天所熟知的 `for/in` 循环。

## for/in (NSFastEnumeration)

通过使用高层抽象，表明我们想遍历一个集合当中的所有元素，这种方法不仅减少了错误的发生，同时也减少了代码量：

```objc
for (id object in array) {
    NSLog(@"%@", object);
}
```

在 Cocoa 当中，生成式方法可以用在任何实现了 `NSFastEnumeration` 协议的类上，包括 `NSArray`, `NSSet`, 和 `NSDictionary`。

对于 `NSFastEnumeration` 你需要了解的是它 *很快* 。哪怕没有很明显的超过，也至少和你自己使用 `for` 循环是一样快的。高速背后的秘密是 `-countByEnumeratingWithState:objects:count:` 这个函数，它会缓存集合成员，并按需加载。和单线程的 `for` 循环实现不同的是，对象的加载是可以并发的，以最大程度利用系统资源。且该方法做遍历删除操作不会导致崩溃。

因此苹果推荐在可能的情况下使用 `NSFastEnumeration` `for/in` 风格进行集合的遍历。

## 使用 Blocks 进行遍历

- **enumerateObjectsUsingBlock**

  ```objc
  [array enumerateObjectsUsingBlock:^(id object, NSUInteger idx, BOOL *stop) {
      NSLog(@"%@", object);
  }];
  ```

  诸如 `NSArray`, `NSSet`, `NSDictionary`，和 `NSIndexSet` 这些集合类型都包含了一系列类似的 block 遍历方法。

  这种方法的一个优势是当前对象的索引 （`idx`）会跟随对象传递进来。`BOOL` 指针可以用于提前返回，相当于传统 C 循环当中的 `break` 语句。

  除非你真的需要在遍历时使用数字索引，使用 `for/in` `NSFastEnumeration` 几乎总是更快的选择。

  重点是，这个系列方法还有带有 `options` 参数的扩展版本：

- **enumerateObjectsWithOptions(NSEnumerationConcurrent)**

  ```objc
  - (void)enumerateObjectsWithOptions:(NSEnumerationOptions)opts
                           usingBlock:(void (^)(id obj, NSUInteger idx, BOOL *stop))block
  ```

  `NSEnumerationOptions`包含：NSEnumerationConcurrent和NSEnumerationReverse，见名知意，一个是并发式遍历，另一个是反向遍历。

## dispatch_apply（快速迭代）

GCD 给我们提供了一种快速迭代方法 `dispatch_apply`，`dispatch_apply` 函数是 `dispatch_sync` 函数和 `Dispatch Group` 的关联 API，该函数按照指定的次数将指定的任务追加到指定队列中，并等待全部任务执行结束，系统会根据实际情况自动分配和管理线程。

```objc
dispatch_apply(count, dispatch_queue_create(nil, DISPATCH_QUEUE_CONCURRENT), ^(size_t index) {
    [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
    NSLog(@"%zu -- %@", index, [NSThread currentThread]);
});
```



# 遍历代码性能选择

- 对于集合中对象数很多的情况下，`for in (NSFastEnumeration)`的遍历速度非常之快

- `enumerateObjectsWithOptions(NSEnumerationConcurrent)`和`dispatch_apply(Concurrent)`的遍历执行可以利用到多核cpu的优势，执行耗时操作效率更高

  

# 遍历日常开发Tips

## 倒序遍历

`NSArray`和`NSOrderedSet`都支持使用`reverseObjectEnumerator`倒序遍历，如：

```objc
NSArray *strings = @[@"1", @"2", @"3"];
for (NSString *string in [strings reverseObjectEnumerator]) {
    NSLog(@"%@", string);
}
```

这个方法只在循环第一次被调用，所以也不必担心循环每次计算的问题。

同时，使用`enumerateObjectsWithOptions:NSEnumerationReverse`也可以实现倒序遍历：

```objc
[array enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(Sark *sark, NSUInteger idx, BOOL *stop) {
    [sark doSomething];
}];
```

## 使用block同时遍历字典key，value

block版本的字典遍历可以同时取key和value（forin只能取key再手动取value）：

```objc
NSDictionary *dict = @{@"a": @"1", @"b": @"2"};
[dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
    NSLog(@"key: %@, value: %@", key, obj);
}];
```

## 对于耗时且顺序无关的遍历，使用并发版本

```objc
[array enumerateObjectsWithOptions:NSEnumerationConcurrent usingBlock:^(Sark *sark, NSUInteger idx, BOOL *stop) {
    [sark doSomethingSlow];
}];
```

遍历执行block会分配在多核cpu上执行（底层很可能就是gcd的并发queue），对于耗时的任务来说是很值得这么做的。同时，对于遍历的外部是保持同步的（遍历都完成后才继续执行下一行），这个特性和dispatch_apply一致，如果要防止该特性可以在外部再嵌套一层`dispatch_async()`方法。