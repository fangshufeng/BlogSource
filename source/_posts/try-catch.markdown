---
layout: post
title: "oc中为什么很少用try catch"
date: 2017-11-22 22:05:39 +0800
comments: true
categories: 
tag:
    Objective-C
---


作为一位ios开发一般都很少用到try catch，因为一般不小心可能会造成一些内存泄露，先来看个例子

<!--more-->

先创建几个测试类
`TestExceptions.h`

```
#import <Foundation/Foundation.h>

@interface TestExceptions : NSObject

@end

```

`TestExceptions.m`

```
#import "TestExceptions.h"

@implementation TestExceptions

- (void)dealloc {
    NSLog(@"TestExceptions---dealloc");
}

@end

```


先看看非ARC环境下

`ViewController.m`

```

   @try {
      TestExceptions * exception = [[TestExceptions alloc] init];
        //主动异常
        @throw  [[NSException alloc] initWithName:@"主动异常" reason:@"这是错误原因" userInfo:nil];
        [exception release];
    }
    @catch (NSException *e) {
        NSLog(@"%@",e);
    }
    
```

咋一看可能没什么问题，但仔细一看会发现`TestExceptions`对象并没有释放,因为`  @throw  [[NSException alloc] initWithName:@"主动异常" reason:@"这是错误原因" userInfo:nil];`中断了`[exception release];`执行导致不能正常释放，当然你可能会像下面这样修改代码


```
   TestExceptions * exception = nil;
    @try {
        exception  = [[TestExceptions alloc] init];
        
        @throw  [[NSException alloc] initWithName:@"主动异常" reason:@"这是错误原因" userInfo:nil];
    }
    @catch (NSException *e) {
        NSLog(@"%@",e);
    }
    
    @finally {
         [exception release];
    }
    
```

这样确实是释放了，但是由于`@finally`会用到`exception`对象所以还要放到外面，如果try里面有很多对象的时候会很繁琐

再来看看`ARC`下


```
    @try {
        TestExceptions * exceptionption  = [[TestExceptions alloc] init];
        
        @throw  [[NSException alloc] initWithName:@"主动异常" reason:@"这是错误原因" userInfo:nil];
    }
    @catch (NSException *e) {
        NSLog(@"%@",e);
    }
    
  
```

由于arc下没有`release`方法，导致对象更加没法释放了，本以为arc会自动处理这种情况，然而事实并没有，我们可以手动将`ViewController.m`文件改为`-fobjc-arc-exceptions`则会出处理该种情况

![15231751172151.jpg](https://upload-images.jianshu.io/upload_images/3279997-b0d30b3b9d2b49c2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出

```
TestExceptions---dealloc

```


还有一种当文件处于objective-c++模式的时候也就是.mm格式时，arc会默认打开`-fobjc-arc-exceptions`，所以我们也可以像下面这样修改

修改`ViewController.m`为`ViewController.mm`发现也是正常释放了

**可以发现**

使用try catch还是有不少问题的，所以苹果推荐使用NSError模式处理异常,再改造一下


```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSError *err = nil;
    [self testTemoExcep:&err];
    if (err) {
        NSLog(@"%@",err);
    }
    
}

```

```
- (void)testTemoExcep:(NSError **)error {
    TestExceptions * exceptionption  = [[TestExceptions alloc] init];
    
    *error = [NSError errorWithDomain:@"异常" code:500 userInfo:nil];
}

```

