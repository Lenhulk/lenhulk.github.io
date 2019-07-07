---
title: Objective-C中基础锁及其性能探究
date: 2019-07-06 15:27:32
tags: [iOS,线程锁,多线程,基础知识]
categories: iOS
---

{% cq %}
懂点锁，才能写好多线程。
{% endcq %}

<!-- more -->

## 线程锁 ##
我们在多线程中大展拳脚的时候，总可能会遇到资源争夺，导致数据混乱，线程锁这个时候可以为我们的数据资源保驾护航，在OC中基础的锁有@synchronized，NSLock，pthread，OSSpinLock。

我们先构建一个类，假设当前它是我们的一个共享资源，且func1与func2是两个互斥的操作，如下：

``` objc
@implementation TokenObj
- (void)func1{
	//读写操作1
}
- (void)func2{
	//读写操作2
}
```

然后，我们来看看如何使用以上四种锁实现在不同线程切换的时候锁住资源。

## @synchronized ##
``` objc
TokenObj *obj = [[TokenObj alloc] init];
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    @synchronized(obj){
        [obj func1];
    }
});
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    sleep(1);//以保证让线程2的代码后执行
    @synchronized(obj){
        [obj func2];
    }
});
```
@synchronized指令中的token为该锁的唯一标识对象，该指令只对相同标识中的对象进行加锁。
用@synchronized实现锁的优点就是不需要在代码中显式地创建锁对象，便可以实现锁的机制，但是它会隐式地添加一个异常处理例程（该历处理例程会在异常抛出的时候自动释放互斥锁）来保护代码。因此该指令会因为该例程带来额外的内存开销，下面我们对比性能的时候就能看到。


#### NSLock ####
```objc
TokenObj *obj = [[TokenObj alloc] init];
NSLock *lock = [[NSLock alloc] init];
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [lock lock];
    [obj func1];
    [lock unlock];
});
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    sleep(1);
    [lock lock];
    [obj func2];
    [lock unlock];
});
```
NSLock是Cocoa提供给我们最基本的锁对象，除lock和unlock方法外，NSLock还提供了tryLock和lockBeforeDate:两个方法，前一个方法会尝试加锁，如果锁不可用(已经被锁住)，刚并不会阻塞线程，并返回NO。lockBeforeDate:方法会在所指定Date之前尝试加锁，如果在指定时间之前都不能加锁，则返回NO。


#### pthread ####
```objc
TokenObj *obj = [[TokenObj alloc] init];
__block pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    pthread_mutex_lock(&mutex);
    [obj func1];
    pthread_mutex_unlock(&mutex);
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	sleep(1);
    pthread_mutex_lock(&mutex);
    [obj func2];
    pthread_mutex_unlock(&mutex);
});
```
pthread_mutex_t是C语言模块，在使用时要引用"pthread.h"。


#### dispatch_semaphore 实现锁 ####
以上代码构建多线程我们就已经用到了GCD的dispatch_async方法，其实在GCD中也已经提供了一种信号机制，使用它我们也可以来构建一把”锁”(从本质意义上讲，信号量与锁是有区别，具体差异参加信号量与互斥锁之间的区别)
```objc
TokenObj *obj = [TokenObj alloc] init];
dispatch_semaphore semaphore = dispatch_semaphore_create(1);
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    [obj func1];
    dispatch_semaphore_signal(semaphore);
});
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    sleep(1);
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    [obj func2];
    dispatch_semaphore_signal(semaphore);
});
```
可以去了解一下dispatch_semaphore。


#### OSSpinLock (deprecated) ####
```objc
TokenObj *obj = [[TokenObj alloc] init];
OSSpinLock spinlock = OS_SPINLOCK_INIT;
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    OSSpinLockLock(&spinlock);
    [obj func1];
    OSSpinLockUnlock(&spinlock);
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	sleep(1);
    OSSpinLockLock(&spinlock);
    [obj func2];
    OSSpinLockUnlock(&spinlock);
});
```
OSSpinLock因为存在安全问题，在iOS10.0以后就被弃用了，可以使用os_unfair_lock_lock替代（除了 OSSpinLock 外，dispatch_semaphore 和 pthread_mutex 性能是最高的）。具体参考：[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)。

## 性能探究 ##
```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <objc/message.h>
#import <libkern/OSAtomic.h>
#import <pthread.h>

#define ITERATIONS (1024*1024*32)

static unsigned long long disp=0, land=0;

int main()
{
	double then, now;
	unsigned int i, count;
	pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
	OSSpinLock spinlock = OS_SPINLOCK_INIT;

	NSAutoreleasePool *pool = [NSAutoreleasePool new];

	NSLock *lock = [NSLock new];
	then = CFAbsoluteTimeGetCurrent();
	for(i=0;i<ITERATIONS;++i)
	{
		[lock lock];
		[lock unlock];
	}
	now = CFAbsoluteTimeGetCurrent();
	printf("NSLock: %f sec\n", now-then);    

	then = CFAbsoluteTimeGetCurrent();
	IMP lockLock = [lock methodForSelector:@selector(lock)];
	IMP unlockLock = [lock methodForSelector:@selector(unlock)];
	for(i=0;i<ITERATIONS;++i)
	{
		lockLock(lock,@selector(lock));
		unlockLock(lock,@selector(unlock));
	}
	now = CFAbsoluteTimeGetCurrent();
	printf("NSLock+IMP Cache: %f sec\n", now-then);    

	then = CFAbsoluteTimeGetCurrent();
	for(i=0;i<ITERATIONS;++i)
	{
		pthread_mutex_lock(&mutex);
		pthread_mutex_unlock(&mutex);
	}
	now = CFAbsoluteTimeGetCurrent();
	printf("pthread_mutex: %f sec\n", now-then);

	then = CFAbsoluteTimeGetCurrent();
	for(i=0;i<ITERATIONS;++i)
	{
		OSSpinLockLock(&spinlock);
		OSSpinLockUnlock(&spinlock);
	}
	now = CFAbsoluteTimeGetCurrent();
	printf("OSSpinlock: %f sec\n", now-then);

	id obj = [NSObject new];

	then = CFAbsoluteTimeGetCurrent();
	for(i=0;i<ITERATIONS;++i)
	{
		@synchronized(obj)
		{
		}
	}
	now = CFAbsoluteTimeGetCurrent();
	printf("@synchronized: %f sec\n", now-then);

	[pool release];
	return 0;
}
```
用以上代码分别对四种基础锁的性能进行探究。代码中我们简单地加锁解锁重复33554432次，然后计算前后消耗时间，排除了自动释放池的影响。运行几次并取平均值统计出下图：

![性能比较](https://s2.ax1x.com/2019/07/07/ZB8lOs.png)

@synchronized必须设置一个异常处理程序，它实际上最终会在那里采取一些内部锁。为了获得可锁定的锁而支付几个加锁解锁的费用。除了少写几行代码，带来的额外开销是其它锁的消耗3倍以上。

OSSpinLock甚至没有进入内核 - 它只是不断重新加载锁，希望它被解锁。如果锁保持超过几纳秒，这是非常低效的。一方面它节省了昂贵的系统调用和几个上下文切换，但是因为iOS系统关于线程优先级的更新，破坏了SpinLock的机制，因此它也变得不再安全了。

NSLock是pthread的一个漂亮的包装器。因此使用它而不是pthread没有多大意义。

