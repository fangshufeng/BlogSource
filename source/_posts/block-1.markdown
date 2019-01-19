---
layout: post
title: "OC--Block--(一)"
date: 2018-08-19 18:45:14 +0800
comments: true
categories: 
tag:
    Objective-C
---


##基本知识点回顾

我们知道按照变量的作用域划分的话，变量可划分为`局部变量`和`全局变量`，而`局部变量`又分为`自动变量`和`静态变量`，`全局变量`分为`静态全局变量`和`非静态全局变量`

<!-- more -->

```
#import <Foundation/Foundation.h>


static int a = 10;
int b = 20;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
      
        auto int  c = 30; // auto 可省略
        static int d = 40;
        
        NSLog(@"\n全局静态变量a:%d\n全局非静态变量b:%d\n自动变量c:%d\n静态局部变量d:%d\n",
              a,
              b,
              c,
              d);
        
        
    }
    return 0;
}


```


![](https://upload-images.jianshu.io/upload_images/3279997-162bcc43e900e88b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##Block的内存结构

新建一个命令行项目 `mian.m`

```
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        void (^myBlock) (void) = ^ {
             NSLog(@"0000");
        };
        
    }
    return 0;
}
```

通过`clang`命令

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m
```
得到c++的代码,找到`main`函数的实现为以下代码，


```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*myBlock) (void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}

====去除掉强制类型转换的代码实际就是下面的形式====
int main(int argc, const char * argv[]) {
{

        void (*myBlock) (void) = &__main_block_impl_0(__main_block_func_0,    
                                                    &__main_block_desc_0_DATA));

    }
    return 0;
}

```

`__main_block_impl_0` 相关的结构以及相关说明如下


```
//__main_block_impl_0 

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)
   { //这个是C++的结构体 该函数是C++结构体的初始化函数 
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

//__block_impl
struct __block_impl { // 这是block函数相关的结构体
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

//__main_block_desc_0  
static struct __main_block_desc_0 {//这是对__main_block_impl_0结构体的描述信息
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};


```

对于`__main_block_impl_0`结构体组成，类似于我们平时开发过程中把一些相关的东西封装成一个对象是一个道理的。

再回到`main.m`函数的实现上:

`myBlock`也就是保存了`__main_block_impl_0`地址，`__main_block_impl_0`初始化函数的第一个参数这里赋值了`__main_block_func_0`函数其实就是那句` NSLog(@"0000");`打印，
`__main_block_func_0`实现如下


```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

    //为了方便理解我直接写成了 打印读者请根据自己的生成的代码对比着看
    NSLog(@"0000");
 }
```

而第二个参数就是`__main_block_desc_0_DATA`就是对`block`的描述，由于构造函数接受的指针类型，所以这里提供的是`__main_block_desc_0_DATA`的地址

接下来改造下`main.m`的代码

```
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        void (^block) (void) = ^ {
            NSLog(@"0000");
        };
        
        block();
        
        NSLog(@"%@",[block class]);
        NSLog(@"%@",[[block class] superclass]);
        NSLog(@"%@",[[[block class] superclass] superclass]);
        NSLog(@"%@",[[[[block class] superclass] superclass] superclass]);
    }
    return 0;
}
```

输出

```
__NSGlobalBlock__
__NSGlobalBlock
NSBlock
NSObject
```


发现`block`的类型的链是`__NSGlobalBlock__` -> `__NSGlobalBlock` -> `NSBlock` -> `NSObject`

所以我们得出以下结论，`block`的内存结构是一个结构体，并且也是一个**oc对象**;
内存结构图如下

![15346658296514.jpg](https://upload-images.jianshu.io/upload_images/3279997-e0f48d079d407a06.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##Block调用

上面说的都是`block`的申明，关于`block`调用就是下面这行代码了


``` 

block();

// 转成C++就是下面的形式

 block->FuncPtr(block);
```

读者可能有疑惑既然`block`是指向`__main_block_impl_0`结构体的但是`__main_block_impl_0`又没有`FuncPtr`，那怎么可以直接调用呢，应该是`block->impl.FuncPtr(block)`才对吧，由于`__main_block_impl_0`是直接拥有了`__block_impl`的并且处于第一个位置的，所以`block`的地址也就是`impl`的地址，所以可以这样写。



## Block的类型

`block`有3种类型

`__NSGlobalBlock__` 对应 `_NSConcreteGloablBlock`；
`__NSStackBlock__` 对应 `_NSConcreteStackBlock`；
`__NSMallocBlock__`对应 `_NSConcreteMallocBlock`;

在内存中存放的位置如图

![15346678342585.jpg](https://upload-images.jianshu.io/upload_images/3279997-8c7046e860ddf2e7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



接下来验证以下

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)
   { //这个是C++的结构体 该函数是C++结构体的初始化函数 
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

由上面的源码中可以看到一个`impl.isa`成员，看过我前面的文章的应该知道`isa`是用来标识该对象是谁的实例的,为了更好的理解`block`的类型，我们先把项目变成`MRC`，将`targets` -> `Build Setting ` -> `Object-C Automatic Reference Counting` 改为`NO`

**MRC环境**

再讲`main`代码改为

```
int a  = 10;
static int b  = 10;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        void (^myBlock1) (void) = ^{
            
        };
        
        void (^myBlock2) (void) = ^{
            NSLog(@"----%d",a);
        };
        
        
        void (^myBlock3) (void) = ^{
            NSLog(@"----%d",b);
        };
        
        
        static int c  = 20;
        void (^myBlock4) (void) = ^{
            NSLog(@"----%d",c);
        };
        
        int d  = 20;
        void (^myBlock5) (void) = ^{
              NSLog(@"----%d",d);
        };
        
        
        void (^myBlock6) (void) = ^{
            NSLog(@"----%d--%d",d,a);
        };
        
        
        void (^myBlock7) (void) = [myBlock5 copy];
        
        void (^myBlock8) (void) = [myBlock1 copy];
        
        void (^myBlock9) (void);
        {
            MyPerson *person = [[MyPerson alloc] init];
            myBlock9 = [^ {
                NSLog(@"----%p",person);
            } copy];
            
            [person release];
        }
        
        
        NSLog(@"myBlock1:%@\nmyBlock2:%@\nmyBlock3:%@\nmyBlock4:%@\nmyBlock5:%@\nmyBlock6:%@\nmyBlock7:%@\nmyBlock8:%@\nmyBlock9:%@",
              [myBlock1 class],
              [myBlock2 class],
              [myBlock3 class],
              [myBlock4 class],
              [myBlock5 class],
              [myBlock6 class],
              [myBlock7 class],
              [myBlock8 class],
              [myBlock9 class]
              );
        
    }
    return 0;
}
```

输出


```
myBlock1:__NSGlobalBlock__
myBlock2:__NSGlobalBlock__
myBlock3:__NSGlobalBlock__
myBlock4:__NSGlobalBlock__
myBlock5:__NSStackBlock__
myBlock6:__NSStackBlock__
myBlock7:__NSMallocBlock__
myBlock8:__NSGlobalBlock__
myBlock9:__NSMallocBlock__

```

对于`block9`:

不使用`copy`则打印`__NSStackBlock__`，并且`MyPerson`正常释放；
使用`copy`则打印`__NSMallocBlock__`，`MyPerson`无法正常释放；


得出以下规律:

1. `block`内部没有访问`auto`变量则为`__NSGlobalBlock__`
2. 访问了`auto`变量则为`__NSStackBlock__`，
3. `__NSStackBlock__` copy以后就会成为`__NSMallocBlock__`
4. `__NSMallocBlock__` copy以后引用计数增加；
5. `__NSGlobalBlock__` copy以后什么也不做


![15346693124195.jpg](https://upload-images.jianshu.io/upload_images/3279997-0ba6987abbcd0a0e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于`block`进行copy操作

![15346693288345.jpg](https://upload-images.jianshu.io/upload_images/3279997-a03c8e9bf61dc8aa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




**ARC环境**

再把项目调回到`arc`环境看看输出

```
myBlock1:__NSGlobalBlock__
myBlock2:__NSGlobalBlock__
myBlock3:__NSGlobalBlock__
myBlock4:__NSGlobalBlock__
myBlock5:__NSMallocBlock__
myBlock6:__NSMallocBlock__
myBlock7:__NSMallocBlock__
myBlock8:__NSGlobalBlock__
myBlock9:__NSMallocBlock__
```


发现有变化的是`myBlock5`和`myBlock6`,这是因为在`ARC`环境下，会根据一些特定的场景会自动调用`copy`方法，规律如下

1. 赋值给有`__strong`修饰的指针变量；
2. COcoa API 中有usingBlock的方法参数时；
3. block作为GCD API方法入参时
4. block作为函数返回值时

`block`作为函数返回值


```
typedef   void (^myBlock) (void);

myBlock  func() {
    
    int a = 10;
    
    myBlock block =  ^ {
        NSLog(@"---%d",a);
    };
    
    return block;
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
  
     
        myBlock temp  = func();
        
        NSLog(@"%@",[temp class]);
        
        temp();
        

    }
    return 0;
}


```

读者可以切换是否为`arc`来观看不同，其他情况无需做说明了 读者自行写demo测试吧



下篇将带来`block`的值捕获，以及如何修改捕获的值





