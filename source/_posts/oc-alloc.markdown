---
layout: post
title: "OC-对象的内存分配"
date: 2018-07-29 23:10:23 +0800
comments: true
categories: 
tag:
    Objective-C
---




### 问题1：一个`NSObject`对象占用了多少内存？

可以通过方法`malloc_size`来查看某个对象分配了多少内存


```
#import <Foundation/Foundation.h>

#import <malloc/malloc.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
       NSObject *obj = [[NSObject alloc] init];
     
        NSLog(@"%zd",malloc_size( (__bridge const void *)(obj)));
    }
    
    return 0;
}

```

输出

```
16
```

为什么会是16呢？？

<!--more-->

我们都知道oc的代码经过编译以后都会生成C或者是C++代码，再到汇编，最后到机器码，通过以下代码查看下编译成c++以后的代码的样子,为了准确指定下架构和设备


```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m
```

成功后在同层目录下出现一个main.cpp文件

```
➜  OCObjeDemo tree
.
├── main.cpp
└── main.m

```

在`main.cpp`中搜索`NSObject`会找到以下代码

```
struct NSObject_IMPL {
	Class isa;
};

```

`IMPL`就是`@implementation`简写，同时，在`NSObject`的头文件中会看到

```
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}

 // 去除忽略警告的代码也即是
 
 @interface NSObject <NSObject> {
    Class isa 
}

```

可以看出`NSObject`就是一个结构体类型，并且只有一个`isa`变量，到这里可能会以为之前输出的16应该就是这里`isa`所占用的字节数，然而事实并不是这样的，通过runtime的一个方法`class_getInstanceSize`可以证明这个结论，我们修改`main.m`代码如下


```
#import <Foundation/Foundation.h>

#import <malloc/malloc.h>
#import <objc/runtime.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
       NSObject *obj = [[NSObject alloc] init];
     
        NSLog(@"%zd",class_getInstanceSize([NSObject class]));
        NSLog(@"%zd",malloc_size( (__bridge const void *)(obj)));
    }
    
    return 0;
}
```
输出

```
8
16
```

`class_getInstanceSize`这个方法需要解释一下打开[苹果开源代码](https://opensource.apple.com/tarballs/objc4/)中的`	objc4-723.tar.gz`，搜索`class_getInstanceSize`可得

```
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}


// Class's ivar size rounded up to a pointer-size boundary.
uint32_t alignedInstanceSize() {
   return word_align(unalignedInstanceSize());
}
    
```
就是获取某个类的所有成员变量的[内存对齐](https://baike.baidu.com/item/%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90/9537460?fr=aladdin)后的地址大小


回到正题，既然只有一个元素并且这个元素的所占的大小是8个字节，为什么`malloc_size`会输出16呢，这就要看下`NSObject`的`alloc`方法的实现了

```
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

一路点下去会看到以下代码

```
    size_t instanceSize(size_t extraBytes) {
        size_t size = alignedInstanceSize() + extraBytes;
        // CF requires all objects be at least 16 bytes.
        if (size < 16) size = 16;
        return size;
    }
```

看到这里也就解释了为什么是输出16而不是8了，也就是`NSObject`最少都应该是16个字节大小的，想想应该是框架需要吧

### 问题2：`NSObject`子类的内存如何分配的


将`main.m`改造如下，新增`Animal`类


```
#import <Foundation/Foundation.h>

#import <malloc/malloc.h>
#import <objc/runtime.h>


@interface Animal : NSObject {
    @public
    int _age;
    int _number;
}

@end


@implementation Animal

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
     
        Animal *animal = [[Animal alloc] init];
        animal->_age = 3;
        animal->_number = 5;
        
        NSLog(@"%zd",class_getInstanceSize([Animal class]));
        NSLog(@"%zd",malloc_size((__bridge const void *)(animal)));
    }
    
    return 0;
}

```


输出

```
16
16
```

同样使用命令

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

```
搜索`Animal`

```
struct Animal_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	int _age;
};

```
由上面可知`NSObject_IMPL`为`NSObject`的内存结构

```
struct NSObject_IMPL {
	Class isa;
};

```

从而`Animal`可以改为下面的形式

```
struct Animal_IMPL {
	Class isa;
	int _age;
};
```
这也就解释了为什么2个都输出16了，还可以通过xcode的断点调试下

通过 `p` + `需要打印的对象`，获取对象的地址

![](https://upload-images.jianshu.io/upload_images/3279997-c2477bcfafc03656.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


再 `Debug` -> `Debug Workflow` -> `View Memory` 

![](https://upload-images.jianshu.io/upload_images/3279997-5f07ce4b00ff02af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![](https://upload-images.jianshu.io/upload_images/3279997-dc14852528dfca57.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


还可以通过`memory write` + `地址` + `值` 修改值，下面的是吧_age的值由原来的5改成了97


```
memory write  0x100636308 61

```

1 . `0x100636308` 是变量`_age`所在的地址，它占4个字节，1个字节8位，1个十六进制表示4位，2个正好一个字节，`0x100636300`为结构体的起始地址，往右数8位正好是`0x100636308`；

2. `61`是十进制`97`的`ASCII`值

![15328696961029.jpg](https://upload-images.jianshu.io/upload_images/3279997-7d3f5ee9ecc2b55a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 问题3：多重继承又该如何分配内存

再次修改`main.m`代码

```
#import <Foundation/Foundation.h>

#import <malloc/malloc.h>
#import <objc/runtime.h>


@interface Animal : NSObject
{
    @public
    int _age;
}

@end

@implementation Animal

@end

@interface Dog : Animal {
    @public
   int _number;
}

@end

@implementation Dog

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
     
        Dog *dog = [[Dog alloc] init];
        dog->_number = 9;
        
        NSLog(@"%zd",class_getInstanceSize([Dog class]));
        NSLog(@"%zd",malloc_size((__bridge const void *)(dog)));
    }
    
    return 0;
}

```

增加`Dog`继承`Animal`

输出

```
16
16
```

同样查看main.cpp

```
struct Dog_IMPL {
	struct Animal_IMPL Animal_IVARS;
	int _number;
};

```

相当于

```
struct Dog_IMPL {
	Class isa;
	int _age;
	int _number;
};
```

 `isa`占8个字节 ， `_age`和 `_number`各占4个字节，对象一共分配`malloc_size`16个字节，成员变量`class_getInstanceSize`分配16个字节



---
**补充说明**

通过一个简单的例子证明下结构体的地址和首元素的地址相同

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
     
        
        struct Test {
            int a;
        };
        
        struct Test test;
        test.a = 33;
        
        printf("%p\n", &test);
        printf("%p\n", &test.a);
    }
    
    return 0;
}

```

都输出

```
0x7ffeefbff518
0x7ffeefbff518

```








