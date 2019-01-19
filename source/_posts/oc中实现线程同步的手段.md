---
title: oc中实现线程同步的手段
date: 2019-01-05 18:30:18
tags:
    [Objective-C,OC-LOCK,线程同步]
---



当多线程访问共享资源的时候，会出现资源竞争的情况，导致数据错乱的问题，OC中要想实现线程间的同步的话，有以下手段：

<!-- more -->

## 一、锁

锁就是实现多线程同步的其中一种方案，查看[示例程序](https://github.com/fangshufeng/demo_project/commit/33e7413b77677283d073076e9f7ae655dd63863a)，

总数是`15`程序执行完成应该是`0`,然而并不是,现在要做的就是让多线程对共享资源这里是


```mm
    int preCount = self.totalCount;
    sleep(0.2);
    self.totalCount = --preCount ;
    
    NSLog(@"---now value : %d---%@",self.totalCount,[NSThread currentThread]);
    
```

能够顺序执行。



### 1.1、自旋锁

所谓的自旋锁，形如以下形式


```mm
while( 抢锁 == 没抢到 ) {

}
```

只要没有抢到锁，程序就会一直重试，所以会一直占用CPU资源。

#### 1.1.1 OSSpinLockLock

跟踪下`OSSpinLockLock`的汇编代码内部确实是一个`while`循环，循环部分的代码如下

```x86asm
 0x104ba98b1 <+12>: cmpl   $-0x1, %eax
 0x104ba98b4 <+15>: jne    0x104ba98d1               ; <+44>
 0x104ba98b6 <+17>: testl  %ecx, %ecx
 0x104ba98b8 <+19>: je     0x104bab0f9               ; _OSSpinLockLockYield
 0x104ba98be <+25>: pause
 0x104ba98c0 <+27>: incl   %ecx
 0x104ba98c2 <+29>: movl   (%rdi), %eax
 0x104ba98c4 <+31>: testl  %eax, %eax
 0x104ba98c6 <+33>: jne    0x104ba98b1               ; <+12>
 0x104ba98c8 <+35>: xorl   %eax, %eax
 0x104ba98ca <+37>: lock
 0x104ba98cb <+38>: cmpxchgl %edx, (%rdi)
 0x104ba98ce <+41>: jne    0x104ba98b1               ; <+12>
 
```

`OSSpinLockLock`特点：

1. 效率高，因为一直占用着cpu，其实这个是优点也是缺点
2. 会出现优先级反转（对于优先级不同的线程，使用同一把锁访问共同资源，优先级低的线程先拿到锁，这时优先级高的线程过来后，由于锁被占用，但是优先级高，一直占有CPU资源，导致持有锁的低优先级的线程无法执行释放锁的操作，从而优先级高的线程一直处于忙等的状态）的问题；
3. 无法处理递归。


OSSpinLockLock使用见[demo](https://github.com/fangshufeng/demo_project/commit/f67150c4e51a0a8de485373f98a500477783fd50)


### 1.2、互斥锁

由于自旋锁一直占用着cpu资源

```mm
while( 抢锁 == 没抢到 ) {

}
```

其实不需要一直循环等待，只要当检测到锁被占用了，那么就去睡觉，当锁的状态改变了，通知它就行了，这就是互斥锁。

```mm
while (抢锁 == 没抢到) {
    当前线程先去睡觉，当锁的状态改变了，再唤醒;
}

```

#### 1.2.1 os_unfair_lock

由于`OSSpinLockLock`存在优先级反转的问题，`os_unfair_lock`是苹果用来替代`OSSpinLockLock`的。跟踪了汇编代码，等待的汇编代码如下

跟踪路径：`os_unfair_lock_lock` -> `_os_unfair_lock_lock_slow` -> `__ulock_wait`

```x86asm

 libsystem_kernel.dylib`__ulock_wait:
 0x10701f360 <+0>:  movl   $0x2000203, %eax          ; imm = 0x2000203
 0x10701f365 <+5>:  movq   %rcx, %r10
 
 0x10701f368 <+8>:  syscall
 
 0x10701f36a <+10>: jae    0x10701f374               ; <+20>
 0x10701f36c <+12>: movq   %rax, %rdi
 0x10701f36f <+15>: jmp    0x10701ce67               ; cerror_nocancel
 0x10701f374 <+20>: retq
 0x10701f375 <+21>: nop
 0x10701f376 <+22>: nop
 0x10701f377 <+23>: nop
 
```

一旦执行到`syscall`时，该条线程就去睡觉了。


`os_unfair_lock`特点：

1. 没有优先级反转的问题；
2. 由于线程睡觉，当然唤醒也是要时间的，效率没有自旋锁高；
3. `iOS10.0`以上才可用;
4. 也不能处理递归的场景。


os_unfair_lock使用见[demo](https://github.com/fangshufeng/demo_project/commit/a04220d6d78b997e3e09a26e44f77ea99523cab3)


#### 1.2.2 pthread_mutex_t


这个是一个比较底层的api，在`C`和`C++`都可以使用，这个对于没有抢到的资源也是睡觉，所以是互斥锁，通过断点可以查看休眠的代码


```x86asm
 libsystem_kernel.dylib`__psynch_mutexwait:
 0x10c5f1868 <+0>:  movl   $0x200012d, %eax          ; imm = 0x200012D
 0x10c5f186d <+5>:  movq   %rcx, %r10
 
 0x10c5f1870 <+8>:  syscall // 也是调用的这个系统调用，线程处于休眠状态
 
 0x10c5f1872 <+10>: jae    0x10c5f187c               ; <+20>
 0x10c5f1874 <+12>: movq   %rax, %rdi
 0x10c5f1877 <+15>: jmp    0x10c5eee67               ; cerror_nocancel
 0x10c5f187c <+20>
 
```

找到该代码的路径为`pthread_mutex_lock -> pthread_mutex_firstfit_lock_slow -> _pthread_mutex_firstfit_lock_wait -> __psynch_mutexwait`

1. 普通锁用法[demo](https://github.com/fangshufeng/demo_project/commit/5489c25186d88147c8c14a031fa9923aac705901)或者[demo](https://github.com/fangshufeng/demo_project/commit/ca3b8d5c8985017b3dac80016d300702a6bc5a64)
2. 递归锁用法:
    1. 演示没有递归锁程序的[bug](https://github.com/fangshufeng/demo_project/commit/584301bf766c254039598320795d2ae8537d0559);
    2. 使用递归锁解决bug,[demo](https://github.com/fangshufeng/demo_project/commit/e6df707ee017707be121f1c79e15a1298413d809)
3. 条件锁用法[demo](https://github.com/fangshufeng/demo_project/commit/bc6ab6cd362784acc858b45de99da3a1eeb2bb6e)

`pthread_mutex_t`特点：

1. 没有优先级反转的问题；
2. 由于线程睡觉，当然唤醒也是要时间的，效率没有自旋锁高；
3. 可以处理递归场景；
4. 可以增加条件锁。


#### 1.2.3 NS---LOCK

下面介绍`Foundation`框架下的`lock`类，其实`NSLock`就是对`pthread_mutex_t`的一个上层封装，使其更加面向对象:

1. `NSLock`是对上面提到的`pthread_mutex_t`普通锁用法的封装
2. `NSCondition`和`NSConditionLock`是对上面提到的`pthread_mutex_t`条件锁用法的封装
3. `NSRecursiveLock`是对上面提到的`pthread_mutex_t`递归锁用法的封装。

2中方案验证下观点：

1. 参考`GNUStep`的源码，[地址](http://wwwmain.gnustep.org/resources/downloads.php?site=ftp%3A%2F%2Fftp.gnustep.org%2Fpub%2Fgnustep%2F)找到`Foundation`代码；
2. 通过`hopper`工具查看`/System/Library/Frameworks/Foundation.framework`


`gnu`不是苹果源码，但是有很大的参考价值，截取部分代码如下


```mm
@implementation NSLock

+ (void) initialize
{
  static BOOL	beenHere = NO;

  if (beenHere == NO)
    {
      beenHere = YES;

      pthread_mutexattr_init(&attr_normal);
      pthread_mutexattr_settype(&attr_normal, PTHREAD_MUTEX_NORMAL);
      pthread_mutexattr_init(&attr_reporting);
      pthread_mutexattr_settype(&attr_reporting, PTHREAD_MUTEX_ERRORCHECK);
      pthread_mutexattr_init(&attr_recursive);
      pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE);

      
      pthread_mutex_init(&deadlock, &attr_normal);
      pthread_mutex_lock(&deadlock);

      baseConditionClass = [NSCondition class];
      baseConditionLockClass = [NSConditionLock class];
      baseLockClass = [NSLock class];
      baseRecursiveLockClass = [NSRecursiveLock class];

      tracedConditionClass = [GSTracedCondition class];
      tracedConditionLockClass = [GSTracedConditionLock class];
      tracedLockClass = [GSTracedLock class];
      tracedRecursiveLockClass = [GSTracedRecursiveLock class];

      untracedConditionClass = [GSUntracedCondition class];
      untracedConditionLockClass = [GSUntracedConditionLock class];
      untracedLockClass = [GSUntracedLock class];
      untracedRecursiveLockClass = [GSUntracedRecursiveLock class];
    }
}
```


再通过`hopper`工具最后确认下

分别看下`init`和`lock`和`unlock`

##### 1.2.3.1 NSLock

![15465046593957.jpg](https://upload-images.jianshu.io/upload_images/3279997-f009f37b69b31b6f.jpg)

![15465046780683.jpg](https://upload-images.jianshu.io/upload_images/3279997-03faba3e93d9481b.jpg)

![15465046919800.jpg](https://upload-images.jianshu.io/upload_images/3279997-b93dfba2632c8c0c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用的[demo地址](https://github.com/fangshufeng/demo_project/commit/6f31e1e92854a7d84e5f8fbbecd55fdc20af6201)


##### 1.2.3.2 NSCondition 

![15465047250284.jpg](https://upload-images.jianshu.io/upload_images/3279997-0df9de48fefb446d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15465047399306.jpg](https://upload-images.jianshu.io/upload_images/3279997-a50ceb26d8acef80.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15465047553345.jpg](https://upload-images.jianshu.io/upload_images/3279997-99d4321958b239f2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用的[demo地址](https://github.com/fangshufeng/demo_project/commit/95a9533fc8c2537ae9ec5ad0096410cb6c2400b7)

##### 1.2.3.3 NSConditionLock

这个是对`NSCondition`的封装，扩展了功能而已

![15465048157582.jpg](https://upload-images.jianshu.io/upload_images/3279997-2f8221ccd383b345.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


主要是`lockWhenCondition`和`unlockWithCondition`，看下`GNU`的代码


```mm
- (void) lockWhenCondition: (NSInteger)value
{
  [_condition lock];
  while (value != _condition_value)
    {
      [_condition wait];
    }
}

- (void) unlockWithCondition: (NSInteger)value
{
  _condition_value = value;
  [_condition broadcast];
  [_condition unlock];
}

```

有了源码就很好理解了。

使用的[demo地址](https://github.com/fangshufeng/demo_project/commit/7e10ea4efd9bce3c512ff1d436ce61089b8fa1c5)

##### 1.2.3.4 NSRecursiveLock


![15465048348121.jpg](https://upload-images.jianshu.io/upload_images/3279997-bbe525c3929082b7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15465048480822.jpg](https://upload-images.jianshu.io/upload_images/3279997-42acffafcb1ba128.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15465048587863.jpg](https://upload-images.jianshu.io/upload_images/3279997-d2b70770214d19f1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


使用的[demo地址](https://github.com/fangshufeng/demo_project/commit/46906eef8e2057ba3240d4750ce42c28961ce782)

通过逆向验证了我们的观点，所以不用测试论速度的话肯定不如`pthread_mutex_t`，毕竟`objc_msgSend`也是要时间的。

#### 1.2.4 Synchronized

使用方式很简单


```mm
- (void)reduceCount {
    @synchronized (self) {
        [super reduceCount];
    }
}


```

通过汇编代码看下`synchronized`多线程等待的实现，代码修改见[提交](https://github.com/fangshufeng/demo_project/commit/970b2c49ef92cfb2bf74d47c4f1ed33a630bf55b)，断点在此处，拦截第二次断点，注意是第二次。


![15465721424857.jpg](https://upload-images.jianshu.io/upload_images/3279997-c9615e3e884231ef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



`objc_sync_enter` ->  `os_unfair_recursive_lock_lock_with_options` ->  `_os_unfair_lock_lock_slow` ->   `__ulock_wait` 


```x86asm
libsystem_kernel.dylib`__ulock_wait:
->  0x10f124360 <+0>:  movl   $0x2000203, %eax          ; imm = 0x2000203 
    0x10f124365 <+5>:  movq   %rcx, %r10
    0x10f124368 <+8>:  syscall 
    0x10f12436a <+10>: jae    0x10f124374               ; <+20>
    0x10f12436c <+12>: movq   %rax, %rdi
    0x10f12436f <+15>: jmp    0x10f121e67               ; cerror_nocancel
    0x10f124374 <+20>: retq   
    0x10f124375 <+21>: nop    
    0x10f124376 <+22>: nop    
    0x10f124377 <+23>: nop 
    
```

发现到最后也是调用了`syscall`，从`_os_unfair_lock_lock_slow`开始和上面的`os_unfair_lock`一样。

`Synchronized`的源码是开源的在`objc`的`objc-sync.mm`文件中

```c++
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        assert(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}


```

本质是对`os_unfair_lock`的一个封装

```mm
typedef struct os_unfair_recursive_lock_s {
	os_unfair_lock ourl_lock;
	uint32_t ourl_count;
} os_unfair_recursive_lock, *os_unfair_recursive_lock_t;


```


特点如下：

1. 没有优先级反转的问题；
2. 由于线程睡觉，当然唤醒也是要时间的，效率没有自旋锁高；
3. 可以处理递归场景；
4. synchronized的内部维护了一个map表，性能稍微其他的差些。


## 二、串行队列

我们要实现线程同步的根本原因是当多线程访问统一共享资源的时候会出现数据错乱的问题，那么只要能够保证多线程执行共享资源任务的时候能够顺序执行就可以了，那么串行队列也是一种方案。使用也很简单


```mm

- (instancetype)init {
    if (self = [super init]) {
        self.queue = dispatch_queue_create("test", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

- (void)reduceCount {
    
    dispatch_sync(self.queue, ^{
        [super reduceCount];
    });
    
}

```

## 三、信号量

使用如下：


```mm
- (instancetype)init {
    if (self= [super init]) {
    
        self.semaphore = dispatch_semaphore_create(1);
    }
    
    return self;
}

- (void)reduceCount {
    dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);
    [super reduceCount];
    dispatch_semaphore_signal(self.semaphore);
}

```

1. 通过`dispatch_semaphore_create`定好信号量的数量；
2. `dispatch_semaphore_wait`调用则`self.semaphore`会减1，如果减1以后<=0 ，那么就会等待，知道>0；
3. `dispatch_semaphore_signal`会将`self.semaphore`加1。


## 四、性能对比


关乎各种方案的性能对比yykit的作者做了个代码比较，[地址](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

性能从高到低排序：

1. os_unfair_lock
2. OSSpinLock
3. dispatch_semaphore
4. pthread_mutex
5. dispatch_queue(DISPATCH_QUEUE_SERIAL)
6. NSLock
7. NSCondition
8. pthread_mutex(recursive)
9. NSRecursiveLock
10. NSConditionLock
11. @synchronized


其实，不通过代码测试也能猜出个大概了：

1. 自旋锁高于互斥锁
2. 越接近底层的api肯定是更快的，所以c和GCD的会快于`oc`（上层封装）的；


## 五、推荐

开发中如果对性能要求比较高的话比较常见的选择：

1. 常规锁的话，推荐使用`dispatch_semaphore`和`pthread_mutex`；
2. 递归锁的话，推荐使用`pthread_mutex(recursive)`和`NSRecursiveLock`（[rac](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/ReactiveCocoa/NSObject%2BRACPropertySubscribing.m)中用的是这个）


**说明** 本文的完整[demo地址](https://github.com/fangshufeng/demo_project/tree/master/Lock)。


















