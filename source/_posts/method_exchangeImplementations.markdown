---
layout: post
title: "思考：多次使用method_exchangeImplementations同一个方法会是怎样的结果，以及分类的加载顺序又是什么呢？"
date: 2018-03-23 14:34:53 +0800
comments: true
categories: 
tag:
    Objective-C
---



### 先看第一个问题：分类的加载顺序
举例`UIViewController`3个分类分别为
`UIViewController+A`
`UIViewController+B`
`UIViewController+C`

`.A`

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        NSLog(@"----A---");
    });
}

```

`.B`

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        NSLog(@"----B---");
    });
}

```
`.C`

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        NSLog(@"----C---");
    });
}

```

运行结果

```
2018-03-23 11:06:10.975429+0800 ExchangeImp[4802:195579] ----C---
2018-03-23 11:06:10.976176+0800 ExchangeImp[4802:195579] ----A---
2018-03-23 11:06:10.976392+0800 ExchangeImp[4802:195579] ----B---

```
这个顺序是怎么来的的呢，就不卖关子了，本文的重点是第二个问题
<!--more-->
![15217744745721.jpg](https://upload-images.jianshu.io/upload_images/3279997-54008ba988f414e1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实是这里的顺序，其实压栈的顺序，上面的顺序是C,B,A

我们修改下让以B,A,C输出

![15217745766821.jpg](https://upload-images.jianshu.io/upload_images/3279997-c978c3915cde52c6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行输出

```
2018-03-23 11:09:23.697753+0800 ExchangeImp[4881:201198] ----B---
2018-03-23 11:09:23.698441+0800 ExchangeImp[4881:201198] ----A---
2018-03-23 11:09:23.698639+0800 ExchangeImp[4881:201198] ----C---
```
### 第二个问题：多次使用method_exchangeImplementations同一个方法会是怎样的结果

`method_exchangeImplementations`关于该方法的使用请自行学习，不是本文的重点

这里我们同时对系统的`presentViewController:animated:completion:`举例

代码结构

```
.
├── ExchangeImp
│   ├── AppDelegate.h
│   ├── AppDelegate.m
│   ├── TestViewController.h
│   ├── TestViewController.m
│   ├── UIViewController+A.h
│   ├── UIViewController+A.m
│   ├── UIViewController+B.h
│   ├── UIViewController+B.m
│   ├── UIViewController+C.h
│   ├── UIViewController+C.m
│   ├── ViewController.h
│   ├── ViewController.m
│   └── main.m
└── ExchangeImpUITests
    ├── ExchangeImpUITests.m
    └── Info.plist

```

`A.m`

```
//
//  UIViewController+A.m
//  ExchangeImp
//
//  Created by fangshufeng on 2018/3/23.
//  Copyright © 2018年 fangshufeng. All rights reserved.
//

#import "UIViewController+A.h"
#import <objc/runtime.h>

@implementation UIViewController (A)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSLog(@"---A-被加载了--");  
        Class class = [self class];
        
        SEL originalSelector = @selector(presentViewController:animated:completion:);
        SEL swizzledSelector = @selector(aAlertPresentViewController:animated:completion:);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
        
    });
}

- (void)aAlertPresentViewController:(UIViewController *)viewControllerToPresent
                           animated:(BOOL)flag
                         completion:(void (^)(void))completion {
    
    NSLog(@"---A-被执行了--")
    
    [self aAlertPresentViewController:viewControllerToPresent
                             animated:flag
                           completion:completion];
}

@end

```

`B.m`

```
//
//  UIViewController+B.m
//  ExchangeImp
//
//  Created by fangshufeng on 2018/3/23.
//  Copyright © 2018年 fangshufeng. All rights reserved.
//

#import "UIViewController+B.h"
#import <objc/runtime.h>

@implementation UIViewController (B)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSLog(@"---B-被加载了--");
        Class class = [self class];
        
        SEL originalSelector = @selector(presentViewController:animated:completion:);
        SEL swizzledSelector = @selector(bAlertPresentViewController:animated:completion:);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
        
    });
}

- (void)bAlertPresentViewController:(UIViewController *)viewControllerToPresent
                           animated:(BOOL)flag
                         completion:(void (^)(void))completion {
    
    NSLog(@"---B-被执行了--");
    [self bAlertPresentViewController:viewControllerToPresent
                             animated:flag
                           completion:completion];
}

@end
```


`C.m`

```
//
//  UIViewController+C.m
//  ExchangeImp
//
//  Created by fangshufeng on 2018/3/23.
//  Copyright © 2018年 fangshufeng. All rights reserved.
//

#import "UIViewController+C.h"
#import <objc/runtime.h>

@implementation UIViewController (C)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSLog(@"---C-被加载了--");
        Class class = [self class];
        
        SEL originalSelector = @selector(presentViewController:animated:completion:);
        SEL swizzledSelector = @selector(cAlertPresentViewController:animated:completion:);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
        
    });
}

- (void)cAlertPresentViewController:(UIViewController *)viewControllerToPresent
                           animated:(BOOL)flag
                         completion:(void (^)(void))completion {
    NSLog(@"---C-被执行了--");
    [self cAlertPresentViewController:viewControllerToPresent
                             animated:flag
                           completion:completion];
}


@end

```

`ViewController.m`

```
//
//  ViewController.m
//  ExchangeImp
//
//  Created by fangshufeng on 2018/3/23.
//  Copyright © 2018年 fangshufeng. All rights reserved.
//

#import "ViewController.h"
#import "TestViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    TestViewController *vc = [[TestViewController alloc] init];
    [self presentViewController:vc animated:YES completion:nil];
}

@end

```

在`build-Phase`中的`compile source`中顺序如下
![15217831021195.jpg](https://upload-images.jianshu.io/upload_images/3279997-ed5dab1ee019a2a4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行点击后

```
2018-03-23 11:25:04.816560+0800 ExchangeImp[5364:228651] ---B-被加载了--
2018-03-23 11:25:04.817206+0800 ExchangeImp[5364:228651] ---A-被加载了--
2018-03-23 11:25:04.817407+0800 ExchangeImp[5364:228651] ---C-被加载了--

2018-03-23 11:25:09.314340+0800 ExchangeImp[5364:228651] ---C-被执行了--
2018-03-23 11:25:09.314502+0800 ExchangeImp[5364:228651] ---A-被执行了--
2018-03-23 11:25:09.314619+0800 ExchangeImp[5364:228651] ---B-被执行了--

```

发现2个现象：

1. 所有的分类方法都被执行了；
2. 正好和加载的顺序相反


下面具体解释下这2个现象
在分类A、B、C执行之前2个方法是这样的指向的

![15217770799332.jpg](https://upload-images.jianshu.io/upload_images/3279997-eadd7834e6a66a41.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



在经历分类`B`以后，变成下面这样

![15217771048030.jpg](https://upload-images.jianshu.io/upload_images/3279997-ebf4bf738141fbac.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




所以在如果没有`A`和`C`的话，那么在执行

```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    TestViewController *vc = [[TestViewController alloc] init];
    [self presentViewController:vc animated:YES completion:nil];
}
```

调用顺序为：
`presentViewController:animated:completion:` -> `NSLog(@"---B-被执行了--");` -> `bAlertPresentViewController:animated:completion:)`

所以输出顺序应该是

```
---B-被执行了--
```

然后弹出vc

回到正题，那么在执行分类`A`之前的样子，A方法和`presentViewController:animated:completion:`指向就是下图
![15217773012806.jpg](https://upload-images.jianshu.io/upload_images/3279997-8174f4cc45bf21b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在执行分类A以后，指向如下

![15217773565704.jpg](https://upload-images.jianshu.io/upload_images/3279997-cfb959d903251484.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



所以如果没有分类`C`的话这时候的输出应该为

```
---A-被执行了--
---B-被执行了--
```

继续分析，在分类`c`执行之前，各个指向为下图

![15217774989252.jpg](https://upload-images.jianshu.io/upload_images/3279997-acced5a9aafbd6b9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


执行以后如下

![15217775721780.jpg](https://upload-images.jianshu.io/upload_images/3279997-bda088bf1da2b719.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



则执行顺序就很清楚了，到这里也就解释了为何一开始是的输出


```
2018-03-23 11:25:04.816560+0800 ExchangeImp[5364:228651] ---B-被加载了--
2018-03-23 11:25:04.817206+0800 ExchangeImp[5364:228651] ---A-被加载了--
2018-03-23 11:25:04.817407+0800 ExchangeImp[5364:228651] ---C-被加载了--
2018-03-23 11:25:09.314340+0800 ExchangeImp[5364:228651] ---C-被执行了--
2018-03-23 11:25:09.314502+0800 ExchangeImp[5364:228651] ---A-被执行了--
2018-03-23 11:25:09.314619+0800 ExchangeImp[5364:228651] ---B-被执行了--
```

接下来通过logo打印证实上面的观点
改造下之前的代码以`B.m`为例


```
//
//  UIViewController+B.m
//  ExchangeImp
//
//  Created by fangshufeng on 2018/3/23.
//  Copyright © 2018年 fangshufeng. All rights reserved.
//

#import "UIViewController+B.h"
#import <objc/runtime.h>

@implementation UIViewController (B)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSLog(@"---B-被加载了--");
        Class class = [self class];
        
        SEL originalSelector = @selector(presentViewController:animated:completion:);
        SEL swizzledSelector = @selector(bAlertPresentViewController:animated:completion:);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        NSLog(@"B-begin---%p-----%p",method_getImplementation(originalMethod),method_getImplementation(swizzledMethod));
        method_exchangeImplementations(originalMethod, swizzledMethod);
        
        NSLog(@"B-after---%p-----%p",method_getImplementation(originalMethod),method_getImplementation(swizzledMethod));
    });
}

- (void)bAlertPresentViewController:(UIViewController *)viewControllerToPresent
                           animated:(BOOL)flag
                         completion:(void (^)(void))completion {
    
    NSLog(@"---B-被执行了--");
    [self bAlertPresentViewController:viewControllerToPresent
                             animated:flag
                           completion:completion];
}

@end

```

执行以后


```
ExchangeImp[6709:319693] ---B-被加载了--
ExchangeImp[6709:319693] B-begin---0x10cedabc8-----0x10b6241d0
ExchangeImp[6709:319693] B-after---0x10b6241d0-----0x10cedabc8

ExchangeImp[6709:319693] ---A-被加载了--
ExchangeImp[6709:319693] A-begin---0x10b6241d0-----0x10b624450
ExchangeImp[6709:319693] A-after---0x10b624450-----0x10b6241d0

ExchangeImp[6709:319693] ---C-被加载了--
ExchangeImp[6709:319693] C-begin---0x10b624450-----0x10b6246d0
ExchangeImp[6709:319693] C-after---0x10b6246d0-----0x10b624450

```

主要看下`originalMethod`的地址

A的开始bengin的开始也即是`originalMethod`指向为`0x10b6241d0`，正是在`B`B-after中交换后的`0x10b6241d0`

可以看到每次交换的开始都是上次交换的结果

**结论**

多次对某个方法进行`method_exchangeImplementations`所有交换的都会执行到，执行顺序可以自己在`build-phase`中修改


[项目地址](https://gitee.com/he_he/iOSDemoProject)
（完）













