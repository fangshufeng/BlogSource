---
layout: post
title: "OC-KVC调用和赋值原理"
date: 2018-08-11 23:37:03 +0800
comments: true
categories: 
tag:
    Objective-C
---


本文主要介绍kvo是如何查找变量然后赋值，以及通过key取出值的

<!-- more -->

#### 准备工作

`Animal`类

```
// .h
#import <Foundation/Foundation.h>
@interface Animal : NSObject
@end


//.m
#import "Animal.h"
@implementation Animal

@end

```

`main.m`


```
#import <Foundation/Foundation.h>
#import "Animal.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];

        [animal setValue:@12 forKey:@"age"];

    }
    return 0;
}
```

#### setValue:forkey:

这个时候运行上面的代码会报下面的错误

```
 Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<Animal 0x10052f6b0> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key age.'

```

下面对`Animal`代码进行改造

`Animal.m`改造1

```
- (void)setAge:(int)age {
    NSLog(@"setAge---%d",age);

}
```

程序正常运行并打印

```
setAge---12
```

`Animal.m`改造2

```
//- (void)setAge:(int)age {
//    NSLog(@"setAge---%d",age);
//
//}

- (void)_setAge:(int)age {
    NSLog(@"_setAge---%d",age);
}

```

程序正常输出

```
 _setAge---12
```

`Animal.m`改造3


```
- (void)setAge:(int)age {
    NSLog(@"setAge---%d",age);

}

- (void)_setAge:(int)age {
    NSLog(@"_setAge---%d",age);
}
```

程序正常输出并且只输出

```
setAge---12
```

**首先得出下面的结论**

* `setValue:forkey:`会调用调用`-set<Key>:`方法；
*  当-set<Key>:不存在的时候回调用-_set<Key>:方法.

---

`Animal.m`改造4


```
#import "Animal.h"

@interface Animal ()

{
    int _age;
    int _isAge;
    int age;
    int isAge;
}

@end

@implementation Animal

//- (void)setAge:(int)age {
//    NSLog(@"setAge---%d",age);
//
//}
//
//- (void)_setAge:(int)age {
//    NSLog(@"_setAge---%d",age);
//}

@end
```

先打个断点，然后看下面控制台的值的情况
![15337987372295.jpg](https://upload-images.jianshu.io/upload_images/3279997-1b7e06e03f87e90a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候是赋值给了`_age`


`Animal.m`改造5


```
#import "Animal.h"

@interface Animal ()

{
   // int _age;
    int _isAge;
    int age;
    int isAge;
}

@end

@implementation Animal

//- (void)setAge:(int)age {
//    NSLog(@"setAge---%d",age);
//
//}
//
//- (void)_setAge:(int)age {
//    NSLog(@"_setAge---%d",age);
//}

@end
```

这次注释掉`_age`

![15337988439482.jpg](https://upload-images.jianshu.io/upload_images/3279997-534cc2d387f18131.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这次给了`_isAge`

同理分别注释下其他的2个

**得出第二个结论**

* 当`-set<Key>:` 和`-_set<Key>:`方法都没实现时`kvc`会按照`_<key>` -> `_is<Key>` -> `<key>` -> `is<Key>`进行赋值

---

`Animal.m`改造6


```
#import "Animal.h"

@interface Animal ()

{
    int _age;
    int _isAge;
    int age;
    int isAge;
}

@end

@implementation Animal

//- (void)setAge:(int)age {
//    NSLog(@"setAge---%d",age);
//
//}
//
//- (void)_setAge:(int)age {
//    NSLog(@"_setAge---%d",age);
//}

+ (BOOL)accessInstanceVariablesDirectly {
    return NO;
}
@end

```

运行程序发现奔溃了

`accessInstanceVariablesDirectly`表示当没有set方法的时候要不要继续查找成员变量，默认是需要

**得出kvc赋值最终的结论**
用一张图来表示，图上是以具体的属性名称来表示的

![15337960989497.jpg](https://upload-images.jianshu.io/upload_images/3279997-2ac1909abcc0ec84.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### valueForKey:

改造代码
`Animal`

```
// .h
#import <Foundation/Foundation.h>
@interface Animal : NSObject
@end


//.m
#import "Animal.h"
@implementation Animal

@end

```

`main.m`

```
#import <Foundation/Foundation.h>
#import "Animal.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];

        NSLog(@"%@",[animal valueForKey:@"name"]);

    }
    return 0;
}

```

运行程序发现崩溃了

下面改造`Animal.m`代码

`Animal`改造1


```
- (int)getName {
    return 12;
}
```

输出

```
12
```

`Animal`改造2

```
//- (int)getName {
//    return 12;
//}

- (int)name {
    return 13;
}
```

输出

```
13
```

`Animal`改造3


```
//- (int)getName {
//    return 12;
//}

//- (int)name {
//    return 13;
//}

- (int)isName {
    return 14;
}
```

输出

```
14
```

`Animal`改造4

```
//- (int)getName {
//    return 12;
//}

//- (int)name {
//    return 13;
//}

//- (int)isName {
//    return 14;
//}

- (int)_getName {
    return 15;
}
```

输出

```
15
```

`Animal`改造5


```
//- (int)getName {
//    return 12;
//}

//- (int)name {
//    return 13;
//}

//- (int)isName {
//    return 14;
//}
//
//- (int)_getName {
//    return 15;
//}

- (int)_name {
    return 16;
}
```

输出

```
16
```


**同样先得出下面的结论**

* valueforkey:方法调用方法的顺序是`get<Key>` -> `<key>` -> `is<Key>` -> `_get<Key>` -> `_<key>`

---


`Animal`改造6


```
//.h
@interface Animal : NSObject
{
    @public
    NSString *_name;
    NSString *_isName;
    NSString *name;
    NSString *isName;
}
@end

//.m

@implementation Animal
//- (int)getName {
//    return 12;
//}

//- (int)name {
//    return 13;
//}

//- (int)isName {
//    return 14;
//}
//
//- (int)_getName {
//    return 15;
//}

//- (int)_name {
//    return 16;
//}

@end
```

`main.m`


```
#import <Foundation/Foundation.h>
#import "Animal.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        animal->_name  = @"a";
        animal->_isName  = @"b";
        animal->name  = @"c";
        animal->isName  = @"d";
        NSLog(@"%@",[animal valueForKey:@"name"]);

    }
    return 0;
}


```

运行程序发现输出


```
a
```
分别注释以下代码


```
@interface Animal : NSObject
{
    @public
//    NSString *_name;
    NSString *_isName;
    NSString *name;
    NSString *isName;
}
@end

        Animal *animal = [[Animal alloc] init];
//        animal->_name  = @"a";
        animal->_isName  = @"b";
        animal->name  = @"c";
        animal->isName  = @"d";
        NSLog(@"%@",[animal valueForKey:@"name"]);
        
```

发现输出

```
b
```

注释后面的会发现分别输出`c`,`d`

得出以下结论
* 当通过方法找不到时，会按照`_<key>` -> `_<isKey>` -> `<key>` -> `is<Key>`的顺序查找赋值


同样当我们将


```

+ (BOOL)accessInstanceVariablesDirectly {
    return NO;
}
```
就不会往下查找成员变量了，总结后就是下面的这张图


![15337973443126.jpg](https://upload-images.jianshu.io/upload_images/3279997-3aa88e6f1e164596.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 最后

不会是取值还是赋值，`kvc`都是先通过方法操作，发现没有对应的方法的时候，会根据`accessInstanceVariablesDirectly`方法决定是否进行找到对应的成员变量进行操作







