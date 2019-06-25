---
layout: post
title: "OC--Block--(二)"
date: 2018-08-25 15:15:09 +0800
comments: true
categories: 
tag:
    Objective-C
---


上篇讲到了`Block`其实就是`oc对象`



本文主要讲解`Block值捕获`以及如何修改`block`捕获的变量,说明下本文的目录结构（简书竟然没做目录功能，有点失望，给个简书生成目录的[链接](https://blog.csdn.net/Wonder233/article/details/78558307)吧）


注意把示例中的这行代码改成下面的形式，否则无效

```
// @match        https://www.jianshu.com/p/*
```


<!-- more -->
---
 
##  一、Block是如何捕获变量的

### 1.1 局部变量

####  1.1.1  auto类型变量

##### 1.1.1.1 基本数据类型

修改下`main.m`的代码

```mm
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        int a  = 10;
        void (^block) (void) = ^{
            NSLog(@"----%d-",a);
        };
        a = 20;
        block();
    }
    return 0;
}

```

发现打印的是

```
10
```

转成`C++`代码发现有些变化了


```mm
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a; //变化1
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) { // 这也是c++的语法表示 将 参数`_a` 赋值给 变量`a`
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int a = __cself->a; // 变化3

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_4fb442_mi_0,a);
        }


int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        int a = 10;
        void (*block) (void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a));// 变化2
        a = 20;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}
```

`变化1`：发现`block`多了个`int`类型的变量`a`;
`变化2`：调用block的时候除了传递block本身外，把`a`的值也就是10传递给了里面的变量`a`
`变化3`：最后方法执行的地方`__main_block_func_0`也是把`block`的变量`a`取出 

##### 1.1.1.2 对象数据类型


原来`block`的结构还不是定的，会随着拥有的变量改变内存结构

再举个例子来进一步理解下`值捕获`，改造下代码

```mm
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
       NSString *name  = @"jack";
        NSLog(@"进入block前的地址----%p",name);
        void (^block) (void) = ^{
            NSLog(@"进入block的地址----%@--%p",name,name);
        };
        name = @"rose";
        NSLog(@"修改name之后的地址----%p",name);
        block();
    }
    return 0;
}
```
输出


```mm
进入block前的地址----0x100001078
修改name之后的地址----0x1000010d8
进入block的地址----jack--0x100001078
```

`c++`代码，这次简单写下就是多了个`*name`属性


```mm
struct __main_block_impl_0 {
 ...
  NSString *name;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSString *_name, int flags=0) : name(_name) {
...
  }
};
```

解释下为什么是输出`jack`

在进入block前`name`地址为`0x100001078`，当到了`__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, name))`的时候相当于把`name`的值也就是`0x100001078`给了`__main_block_impl_0`里面的`name`变量，当再次`name = @"rose";`的时候之前的`name`的值已经不是`0x100001078`，而是`0x1000010d8`了，所以后来当调用的时候访问的是`__main_block_impl_0`的`name`变量的值`0x100001078`所存的值是`jack`

弄了一张图帮助理解下

![15341544298315.jpg](https://upload-images.jianshu.io/upload_images/3279997-903598797f522002.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

####  1.1.2  static类型变量


##### 1.1.2.1 对象类型

接着我们再次修改下代码

```mm
int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
       static NSString *name  = @"jack";
        void (^block) (void) = ^{
            NSLog(@"----%@--",name);
        };
        name = @"rose";
        block();
    }
    return 0;
}
```

就是加了一个`static`

输出

```
----rose--
```

这次的结果完全不一样了，这又是为什么呢，继续看c++的实现


```mm
struct __main_block_impl_0 {
    ...
    NSString **name;  // 1
    ...
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  NSString **name = __cself->name; // 3

NSLog((NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_9d6d5e_mi_1,(*name));//4
        }

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

       static NSString *name = (NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_9d6d5e_mi_0;

        void (*block) (void) = &__main_block_impl_0(__main_block_func_0, 
                                                &__main_block_desc_0_DATA,
                                                 &name, //2
                                                 570425344));
        name = (NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_9d6d5e_mi_2;

    block->FuncPtr(block);
    }
    return 0;
}
```

看到代码`1`处已经是二重指针，`2`处这个时候给的不是`name`指向的内存地址，而是`name`变量的地址，`3`处在调用的时候取出的是指向`name`变量地址的内存，而不是`name`所指向的内存，所以要想获取`name`所指向的内存，`4`处通过`*name`取值进行传参

也就是红色箭头的变化
![15341552229497.jpg](https://upload-images.jianshu.io/upload_images/3279997-d30999d244aa7add.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.2 全局变量

#### 1.2.1  非static修饰的全局变量

##### 1.2.1.1 对象类型

**当变量是全局的变量时**

```mm

NSString *name  = @"jack";

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        void (^block) (void) = ^{
            NSLog(@"----%@--",name);
        };
        name = @"rose";
        block();
    }
    return 0;
}
```

发现也是输出


```
---rose--
```


额，这又是怎么回事呢，看源码吧


```mm
NSString *name = (NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_3b419d_mi_0;


struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {


            NSLog((NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_3b419d_mi_2,name);
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*block) (void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        
        name = (NSString *)&__NSConstantStringImpl__var_folders_s2_zmz_wcdj2px2kh6hm1zkcd780000gp_T_main_3b419d_mi_3;

        block)->FuncPtr(block);
    }
    return 0;
}
```

我们可以发现这个时候`block`内部并没有像之前那样生成一个同名的变量，也就是对于全局变量`block`是不会捕获的,当变量是`static`全局变量时也和全局变量一样，留给读者自行测试了

##### 1.2.1.2 基本类型

略...

#### 1.2.2  static修饰的全局变量 

略... 读者自行写demo


**说了这么多我们来梳理下，我们思考一个问题，`block`为什么要捕获变量，是因为里面有个方法，方法需要使用变量，**

1. 如果是局部变量的话，如果不持有他的话是不是过了作用域就释放了，那就不能完成方法的正常调用，所以对于局部变量，一定会捕获的；
2. 对于全局变量，刚才说了`block`捕获变量的原因要使用变量，既然是全局变量，那在哪都可以访问，所以不需要捕获；
3. 那为什么局部变量有的是传地址有的是传值呢，对于非`static`修饰的局部变量其实是`auto`的，这种变量是放在栈区的，过了作用域就会被系统回收，如果`block`捕获变量的地址的话，那可能捕获的地址已经被系统回收，或者已经被其他的对象占用了，这个时候程序会出现无法预料的异常，但是如果是`static`修饰的，是放在数据区的，不会随着作用域的而销毁，从而放地址是安全的

总结就是下面这张图
![15341565010715.jpg](https://upload-images.jianshu.io/upload_images/3279997-4eeb98d534833250.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、 Block捕获对对象的引用计数的影响

### 2.1 `__NSMallocBlock__`对对象的引用计数的影响 `ARC`环境

我们知道基本数据类型是放在栈中的，回收是由系统自动回收的无需考虑，所以我们这里只考虑auto类型的对象类型的引用计数

需要新增一个`MyPerson`类



```mm
//MyPerson.h
#import <Foundation/Foundation.h>

@interface MyPerson : NSObject

@end

//MyPerson.m

#import "MyPerson.h"

@implementation MyPerson

- (void)dealloc {
    NSLog(@"-%s",__func__);
}
@end

```

`main.m`代码如下


```mm
int main(int argc, const char * argv[]) {
    @autoreleasepool {
  
        void (^myBlock) (void);
        {
            MyPerson *p = [[MyPerson alloc] init];
           myBlock = ^ {
                NSLog(@"----%@",p);
            };
            
            NSLog(@"block类型---%@",[myBlock class]);
        }
     
        NSLog(@"----");

    }
    return 0;
}


```

![15351680937915.jpg](https://upload-images.jianshu.io/upload_images/3279997-a1cd1b9de047abaf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出

```mm
__NSMallocBlock__
```

这里`p`并没有释放掉，按道理过了`120`行应该就释放的，其实此时` MyPerson *p = [[MyPerson alloc] init];`和`__strong  MyPerson *p = [[MyPerson alloc] init];`是等价的

**说明`__NSMallocBlock__`类型的`myBlock`会对`__strong`修饰的`p`对象的引用计数产生影响**

再修改`main.m`函数
![15351682736470.jpg](https://upload-images.jianshu.io/upload_images/3279997-cb1030a4c0ae6f3c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时输出

```mm
__NSMallocBlock__
--[MyPerson dealloc]
```

发现`p`正常释放了

**说明`__NSMallocBlock__`类型的`myBlock`不会对`__weak`修饰的 `p`对象的引用计数产生影响**

---

### 2.2 `__NSStackBlock__`对对象的引用计数的影响 `MRC`环境

把项目改成`MRC`

![15351618498306.jpg](https://upload-images.jianshu.io/upload_images/3279997-60cbb1cb9f2f89de.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`MyPerson`

```mm
#import "MyPerson.h"

@implementation MyPerson

- (void)dealloc {
    [super dealloc];
    NSLog(@"-%s",__func__);
}
@end

```

`mian.m`如图

![15351615827663.jpg](https://upload-images.jianshu.io/upload_images/3279997-ca0b619c80e47bd3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



此时打印


```
block类型---__NSStackBlock__
--[MyPerson dealloc]
```

发现`p`正常释放了，此时的`myBlock`的类型为`__NSStackBlock__`类型的,

**说明`__NSStackBlock__`类型的`myBlock`不会对`__strong`修饰的 `p`对象的引用计数产生影响**

再修改一下`main.m`的代码如图，新增的代码看红色处

![](https://upload-images.jianshu.io/upload_images/3279997-fc7ce3f0029f64d6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

会发现此时只会输出

```
block类型---__NSMallocBlock__

```
但是`p`并没有释放

**说明`__NSMallocBlock__`类型的`myBlock`会对`__strong`修饰的`p`对象的引用计数产生影响**


### 2.3 结论

**得出以下结论：**
    
1. 当block内部访问了对象类型的auto变量时
    如果block是在栈上，将不会对auto变量产生强引用

2. 如果block被拷贝到堆上
    会调用block内部的copy函数;
    copy函数内部会调用_Block_object_assign函数_Block_object_assign函数会根据auto变量的修饰符（__strong、__weak、__unsafe_unretained）做出相应的操作，形成强引用（retain）或者弱引用

3. 如果block从堆上移除
    会调用block内部的dispose函数;
    dispose函数内部会调用_Block_object_dispose函数_Block_object_dispose函数会自动释放引用的auto变量（release）


## 三、__block如何做到可以修改变量的


由前面的了解我们知道block要想可以修改变量，那么就不能值捕获，也就是不能放在栈内存中，因为栈内存是的释放无法控制，所以要买放在全局区，要么放在堆区，来看下苹果是放在哪里的。

#### 3.1 __block变量的内存结构


```mm
int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        MyPerson *p = [[MyPerson alloc] init];
        p.name = @"jack";
        
        void (^myBlock) (void) = ^ {
            p = [[MyPerson alloc] init];
            NSLog(@"---%@",p.name);
        };
        
        myBlock();
    }
    return 0;
}

```

像这样在`block`的内部直接修改变量是会报错的， 要想修改需要借助`__block`修饰符


```mm
int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
      __block  MyPerson *p = [[MyPerson alloc] init];
        p.name = @"jack";
        
        void (^myBlock) (void) = ^ {
            p = [[MyPerson alloc] init];
            NSLog(@"---%@",p.name);
        };
        
        myBlock();
    }
    return 0;
}

```

我们看看转成的`c++`的代码


![15351801025148.jpg](https://upload-images.jianshu.io/upload_images/3279997-ba255e44bc79b962.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到和之前没有`__block`修饰的不同的是这次的p变成了类型为`__Block_byref_p_0 *p`
再看初始化的地方

![15351802434170.jpg](https://upload-images.jianshu.io/upload_images/3279997-9b4d442c70e374da.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


一行`__block  MyPerson *p = [[MyPerson alloc] init];`就变成了初识化一个`__Block_byref_p_0`类型的结构体，然后把该结构体的指针给到`myBlock`，而我们初始化的那个`p`则给了`__Block_byref_p_0`内部的`p`对象

#### 3.2 __block变量的内存管理

1. 当__block变量在栈上时，不会对指向的对象产生强引用

2. 当__block变量被copy到堆时
    会调用__block变量内部的copy函数，copy函数内部会调用_Block_object_assign函数_Block_object_assign函数会根据所指向对象的修饰符（__strong、__weak、__unsafe_unretained）做出相应的操作，形成强引用（retain）或者弱引用（MRC除外，MRC时不会retain）

3. 如果__block变量从堆上移除
    会调用__block变量内部的dispose函数，dispose函数内部会调用_Block_object_dispose函数
_Block_object_dispose函数会自动释放指向的对象（release）

和没有`__block`修饰的auto对象变量差不多，只是第二条中对`MRC`不起作用
    
![15351806310781.jpg](https://upload-images.jianshu.io/upload_images/3279997-b6532aa68eae570d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
![15351808003801.jpg](https://upload-images.jianshu.io/upload_images/3279997-d1efe4686a9c66d0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里的`Block0`相当于`myBlock`，`__block`变量就是`p`,`15`行的时候变量`MyPerson`类型的变量`p`，就变成了指向`__Block_byref_p_0`类型的指针了，且处于栈中，到了`21`行结束，由于是arc环境,`myBlock`就是为右边的`block`copy后的处于`堆`上了,这是变量`p`也会被拷贝到堆上，当`23`执行的时候调用的就是堆上的`block`，访问的也是堆上的内容，对于`block`内部的`NSLog(@"---%@",p.name);`则是结构体`p`内部的`MyPerson`类型的`p`对象

#### 3.3 __block的__forwarding指针

可以看到`__Block_byref_p_0`结构如下

![15351812044975.jpg](https://upload-images.jianshu.io/upload_images/3279997-14b516f58698e719.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有一个`__forwarding`，在`main.m`中初始化的时候传的就是`__Block_byref_p_0`自身的地址，取值的时候也是通过`p->__forwarding->p`去取值岂不是多此一举？
![15351812636865.jpg](https://upload-images.jianshu.io/upload_images/3279997-c122bea4070044bf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15351813367663.jpg](https://upload-images.jianshu.io/upload_images/3279997-9f1026eddb6f22ea.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实这是不管当`__block`修饰的结构体变量，处于栈上海市被复制到堆上，都可以访问到同一个`p`变量

我们把代码改下


![15351815132066.jpg](https://upload-images.jianshu.io/upload_images/3279997-4d13133766f4d408.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



此时会输出`rose`

代码过了`21`行，`myBlock`就已经在`堆`上了，到了`23`行访问还是栈中的结构体变量，那为何还是打印`rose`呢，就是因为有了`__forwarding`指针的作用，保证了此时不管在`栈`中还是在`堆`中都可以访问到同一个`MyPerson`类型的变量`p`，当`__block`修饰的变量从栈中copy到堆中的时候发送的事情入下图

![15351817089007.jpg](https://upload-images.jianshu.io/upload_images/3279997-671ebf6ca1c4aa0b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



可以看出苹果的实现是通过把变量放在堆区的方式来实现修改`__block`捕获的变量的，也可以看出`__block`对对象的内存影响还是蛮大的。

(完)






