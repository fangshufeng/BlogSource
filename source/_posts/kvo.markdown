---
layout: post
title: "OC--KVO实现原理"
date: 2018-08-08 10:35:59 +0800
comments: true
categories: 
tag:
    Objective-C
---


#### 基本用法

创建一个`iOS`项目
新建类`TestObj`


```
//TestObj.h

#import <Foundation/Foundation.h>

@interface TestObj : NSObject

@property (nonatomic, copy) NSString *name;

@end

//TestObj.m

#import "TestObj.h"

@implementation TestObj


@end


```

`ViewController`如果要需要监听`TestObj`对象的`name`值的改变，代码如下


```
#import "ViewController.h"
#import "TestObj.h"
#import <objc/runtime.h>

@interface ViewController ()

@property (nonatomic, strong) TestObj *obj1;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    TestObj *obj1 = [[TestObj alloc] init];
    obj1.name = @"jack";
    _obj1 = obj1;
    
    [self.obj1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.obj1.name = @"Tom";
}


- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"%@对象的%@值改变了%@",object,keyPath,change);
}
```

运作模拟器点击后输出以下 很简单

```
 <TestObj: 0x6000024d4400>对象的name值改变了{
    kind = 1;
    new = Tom;
    old = jack;
}
```

<!-- more -->
#### 如何实现的当属性改变的时候方法会调用

在`addObserver:forKeyPath:options:context:`方法前后加入下面的打印

```
   NSLog(@"begin--%@",object_getClass(self.obj1));
   
    [self.obj1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
    
    NSLog(@"end--%@",object_getClass(self.obj1));
    
    NSLog(@"%d",[self.obj1 isKindOfClass:[TestObj class]]);
    
```

输出`self.obj1`前后的具体是哪个对象的实例

```
begin--TestObj
end--NSKVONotifying_TestObj
1
```

由`isKindOfClass`输出的结果可以看出`addObserver:forKeyPath:options:context:`就会生成`TestObj`的子类名字为`NSKVONotifying`+`_`+`TestObj`;

#### 谁调用了`addObserver:forKeyPath:options:context:`方法
`TestObj`增加一个方法

```
//.h
- (void)updateWithName:(NSString *)name;

//.m
- (void)updateWithName:(NSString *)name {
    _name = [name copy];
}
```

`ViewController`调用修改为下面的形式

```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    self.obj1.name = @"Tom";
    [self.obj1 updateWithName:@"Tom"];
}
```

发现没有任何打印，得出新生成的子类是重写父类的`set`方法从而达到选择性调用`addObserver:forKeyPath:options:context:`方法的


#### 生成的子类`NSKVONotifying`+`_`+`XXX`的数据结构是怎样的

通过上面已经知道`addObserver:forKeyPath:options:context:`方法以后生成了一个新的对象，那么这个子类对象的元类对象和原来的类相同吗，修改代码如下看看结果


```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    TestObj *obj1 = [[TestObj alloc] init];
    obj1.name = @"jack";
    _obj1 = obj1;
    
    TestObj *obj2 = [[TestObj alloc] init];
    _obj2 = obj2;

    TestObj *obj3 = [[TestObj alloc] init];
    TestObj *obj4 = [[TestObj alloc] init];

    NSLog(@"前-%p--%p--%p--%p",
          object_getClass(object_getClass(obj1)),
          object_getClass(object_getClass(obj2)),
          object_getClass(object_getClass(obj3)),
          object_getClass(object_getClass(obj4)));
    [self.obj1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
    NSLog(@"end--%@",object_getClass(self.obj1));
    [self.obj2 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
    NSLog(@"后-%p--%p--%p--%p",
          object_getClass(object_getClass(obj1)),
          object_getClass(object_getClass(obj2)),
          object_getClass(object_getClass(obj3)),
          object_getClass(object_getClass(obj4)));
}


```

输出

```
前-0x10ee87318--0x10ee87318--0x10ee87318--0x10ee87318

后-0x600002e818c0--0x600002e818c0--0x10ee87318--0x10ee87318
```

说明生成的新类对象的元类对象也变了，`addObserver:forKeyPath:options:context:`过后就变成了下面的这张图。

![15336327004011.jpg](https://upload-images.jianshu.io/upload_images/3279997-e1add16805ced1ab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 为什么[self.obj1 isMemberOfClass:[self.obj3 class]] == true?

读过`runtime`源码应该知道`isMemberOfClass`实现如下

```
 - (BOOL)isMemberOfClass:(Class)cls {
        return [self class] == cls;
    }
    
    
- (Class)class {
    return object_getClass(self);
}
```

也就是说`isMemberOfClass`获取的是对象的`isa`指针，那按照上面的这张图`self.obj1`调用`isMemberOfClass`应该是`NSKVONotifying_TestObj`，而`self.obj3 class]`应该是`TestObj`,怎么会相等呢

再修改下代码如下


```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    TestObj *obj1 = [[TestObj alloc] init];
    obj1.name = @"jack";
    _obj1 = obj1;

    TestObj *obj3 = [[TestObj alloc] init];

    [self.obj1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];

  
    unsigned count = 0;
    Method *methodList = class_copyMethodList(object_getClass(self.obj1), &count);
    
    for (int i = 0; i < count; i++) {
        Method method = methodList[I];
        
        IMP imp = method_getImplementation(method)  ;
        NSLog(@"方法名称：%@--%p",
              NSStringFromSelector(method_getName(method)),imp);
        
    }

}
```

输出

```
方法名称：setName:--0x10f1e0e8a
方法名称：class--0x10f1df8c0
方法名称：dealloc--0x10f1df662
方法名称：_isKVOA--0x10f1df65a
```

说明`NSKVONotifying_TestObj`里面有4个方法，重写了`setName:`、`class`、`dealloc`、`_isKVOA`

断点在此处

![15336330950056.jpg](https://upload-images.jianshu.io/upload_images/3279997-05cb164687ddae05.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出

![15336331072380.jpg](https://upload-images.jianshu.io/upload_images/3279997-44254ebe41c89a0e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以分别得到具体的方法实现


```
imp = 0x000000010af1ae8a (Foundation`_NSSetObjectValueAndNotify)
方法名称：setName:--0x10af1ae8a // 最后调用了Foundation中的_NSSetObjectValueAndNotify方法


Printing description of imp:
(IMP) imp = 0x000000010af198c0 (Foundation`NSKVOClass)
方法名称：class--0x10af198c0


(IMP) imp = 0x000000010164d662 (Foundation`NSKVODeallocate)
方法名称：dealloc--0x10164d662

(IMP) imp = 0x000000010f16165a (Foundation`NSKVOIsAutonotifying)
方法名称：_isKVOA--0x10f16165a

```

对于`class`我们有理由猜测就是返回了

```
- (Class)class {
    return [TestObj class];
}
```

也就解释了为什么返回true了



#### 具体值改变的时机

我们把代码恢复到下面的样子


```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    TestObj *obj1 = [[TestObj alloc] init];
    obj1.name = @"jack";
    _obj1 = obj1;
    
    [self.obj1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
 
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.obj1.name = @"Tom";
}


- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"%@对象的%@值改变了%@",object,keyPath,change);
}

```

在`TestObj.m`中增加2个方法


```
- (void)willChangeValueForKey:(NSString *)key {
    
    NSLog(@"willChangeValueForKey--begin");
    [super willChangeValueForKey:key];
    NSLog(@"willChangeValueForKey--end");
}

- (void)didChangeValueForKey:(NSString *)key {
    NSLog(@"didChangeValueForKey--begin");
    [super didChangeValueForKey:key];
    NSLog(@"didChangeValueForKey--end");
}

```

再运行点击输出

```
willChangeValueForKey--begin
willChangeValueForKey--end
didChangeValueForKey--begin
<TestObj: 0x600003fd0640>对象的name值改变了{
    kind = 1;
    new = Tom;
    old = Tom;
}
didChangeValueForKey--end
```

得出2个结论：

1. `NSKVONotifying_TestObj`重写了set方法并且调用了`willChangeValueForKey`和`didChangeValueForKey`；
2. 真正改变的时机是`didChangeValueForKey`方法，因为`didChangeValueForKey--end`之前值已经发生了改变



#### 如何在不调用`set`方法的情况下主动触发`KVO`?

由前面我们知道直接通过以下方式是不会触发`KVO`的


```
- (void)updateWithName:(NSString *)name {
    _name = name;
}
```

要想这种方式也会触发`KVO`监听需要改造下代码


```
- (void)updateWithName:(NSString *)name {
    [self willChangeValueForKey:@"name"];
    _name = name;
    [self didChangeValueForKey:@"name"];
}
```
这样就可以了

`willChangeValueForKey`必须要调用的，读者可以自己试试看不加会不会触发到监听





