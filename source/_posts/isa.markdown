---
layout: post
title: "谈谈isa"
date: 2017-04-26 15:19:26 +0800
comments: true
categories: 
tag:
    Objective-C
---


### 先来做几道题目，做的出来的就不要往下看了。

```

@interface Animal : NSObject

@end
@implementation Animal

@end

@interface Dog  : Animal

@end

@implementation Dog

@end

@interface Demo40ViewController ()

@end

@implementation Demo40ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Animal *p     = [[Animal alloc] init];
    
    BOOL isEqual  = [p isKindOfClass:[Animal class]]; 
    BOOL isEqual1 = [Animal isKindOfClass:[Animal class]]; 
    BOOL isEqual2 = [NSObject isKindOfClass:[NSObject class]]; 
    
    BOOL isEqual3 = [p isMemberOfClass:[Animal class]]; 
    BOOL isEqual4 = [Animal isMemberOfClass:[Animal class]]; 
    BOOL isEqual5 = [NSObject isMemberOfClass:[NSObject class]]; 
    
    
    BOOL isEqual6 = [(id)[NSObject class] isKindOfClass:[NSObject class]]; 
    BOOL isEqual7 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
    BOOL isEqual8 = [(id)[Dog class] isKindOfClass:[Dog class]]; 
    BOOL isEqual9 = [(id)[Dog class] isMemberOfClass:[Dog class]];
    
NSLog(@"%d__%d_%d_%d__%d_%d_%d__%d_%d_",isEqual,isEqual1,isEqual2,isEqual3,isEqual4,isEqual5,isEqual6,isEqual7,isEqual8,isEqual9);
}

@end

```



下面公布答案，答对的就撤了吧不要在这里浪费时间了，没答出来的那就跟着我一起一探究竟吧😁😀😃

<!--more-->


```
1__0_1_1__0_0_1__0_0_

```



先打开[objc](https://opensource.apple.com/tarballs/objc4/)来看看
`+ isMemberOfClass`,
`- isMemberOfClass`,
`+ isKindOfClass`,
`- isKindOfClass`，
`+ (Class)class`，
`- (Class)class`
的实现，这些方法都是成对出现的，分别是对象方法和类方法。


分别解释一下各个方法，注意区分区别：


```

1. isMemberOfClass 方法 
     
     先看+方法
    + (BOOL)isMemberOfClass:(Class)cls {
        return object_getClass((id)self) == cls;
    }
    
    `object_getClass`方法的实现如下，其实就是取isa
    
    Class object_getClass(id obj) {
          if (obj) return obj->getIsa();
          else return Nil;
    }
    
    再看-方法
    
    - (BOOL)isMemberOfClass:(Class)cls {
        return [self class] == cls;
    }
    
    `- (Class)class`的实现如下，实际上也是取isa
    
    - (Class)class {
        return object_getClass(self);
    }

    
2. isKindOfClass 方法
   
    先看+方法,通过`object_getClass`方法取得isa，然后和cls对比，相等的话就返回YES,否则会寻找self的isa的父类，如果相等返回YES，否则返回NO.
    
    + (BOOL)isKindOfClass:(Class)cls {
            for (Class tcls = object_getClass((id)self); tcls; tcls = tcls- 
            >superclass) {
                if (tcls == cls) return YES;
            }
          return NO;
    }
    
   在看-方法，和+方法一样
    
    - (BOOL)isKindOfClass:(Class)cls {
            for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
                if (tcls == cls) return YES;
            }
            return NO;
     }

3.  class 方法
 
    先看+方法 返回自身
    + (Class)class {
        return self;
    }

    再看-方法 返回self的isa
    - (Class)class {
        return object_getClass(self);
    }

    注意这里的区别


```

上面提到了isa，那么isa到底是个什么的东西呢 ？

直接xcode搜索NSObject可以看到

```
    @interface NSObject <NSObject> {
        Class isa  OBJC_ISA_AVAILABILITY;
    }
```

再点进去看

```

/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;

struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;

```

可以得出结论

1. `Class`就是指向`objc_class`结构体的指针；
2.  `id`就是指向`objc_object`结构体的指针；


由类和对象的定义，我们可以看出，其成员变量中都含有isa指针，isa指向的是一个类，其指向该对象所属的类。我们可以把isa当作是一个对象的标志，因此类也是一个对象，我们称之为类对象，其isa指针，指向该类对象所属的类，我们称之为元类(metaClass)

isa指向的类可以通过【object class】 来获取，但是如果是类对象的话通过【class】来获取的话是本身而不是元类，这取决于上面提到的class的实现。我们可以通过object_getClass()来获取isa，适用于实例对象和类对象。


```
Animal *animal = [[Animal alloc] init];
    Dog *dog = [[Dog alloc] init];
    
    NSLog(@"对象 dog --> isa 指向的地址是 %p",[dog class]);
    NSLog(@"Dog --> isa 指向的地址是 %p",object_getClass([Dog class]));
    NSLog(@"Dog --> isa --> isa 指向的地址是 %p",object_getClass(object_getClass([Dog class])));
    NSLog(@"Dog --> isa --> superclass 的地址是 %p",[object_getClass([Dog class]) superclass]);
    NSLog(@"dog --> superclass 的地址 %p",[dog superclass]);
    
    NSLog(@"============================");
    
    
    NSLog(@"对象 animal --> isa 指向地址是 %p",[animal class]);
    NSLog(@"Animal --> isa 指向地址是 %p",object_getClass([Animal class]));
    NSLog(@"Animal --> isa --> isa 指向的地址是 %p",object_getClass(object_getClass([Animal class])));
    NSLog(@"Animal --> isa --> superclass  的地址是 %p",[object_getClass([Animal class]) superclass]);
    NSLog(@"animal --> superclass 的地址 %p",[animal superclass]);

    
    NSLog(@"============================");
    
    
    NSLog(@"NSObject 的地址 %p",[NSObject class]);
    NSLog(@"NSObject --> isa 的地址是 %p",object_getClass([NSObject class]));
    NSLog(@"NSObject --> isa --> isa 的地址是 %p",object_getClass(object_getClass([NSObject class])));
    NSLog(@"NSObject --> isa --> superclass 的地址是 %p",[object_getClass([NSObject class]) superclass]);
    
    NSLog(@"NSObject --> superclass 的地址 %p",[NSObject superclass]);

```


打印结果

```

对象 dog --> isa 指向的地址是 0x1097e37c8
Dog --> isa 指向的地址是 0x1097e37a0
Dog --> isa --> isa 指向的地址是 0x10c6eae38
Dog --> isa --> superclass 的地址是 0x1097e3750
dog --> superclass 的地址 0x1097e3778

============================

对象 animal --> isa 指向地址是 0x1097e3778
Animal --> isa 指向地址是 0x1097e3750
Animal --> isa --> isa 指向的地址是 0x10c6eae38
Animal --> isa --> superclass  的地址是 0x10c6eae38
animal --> superclass 的地址 0x10c6eae88

============================

NSObject 的地址 0x10c6eae88
NSObject --> isa 的地址是 0x10c6eae38
NSObject --> isa --> isa 的地址是 0x10c6eae38
NSObject --> isa --> superclass 的地址是 0x10c6eae88
NSObject --> superclass 的地址 0x0

```

可以得出下面一张图


![362FFD36-BE20-48E0-9019-92A371200071.png](http://upload-images.jianshu.io/upload_images/3279997-eb5374d3f3c389c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



由打印的结果以及上面的图可以得出以下结论：

1. 实例对象的isa指向对象所属的类；
2. 类对象的isa指向的是类对象所属的类，也就是元类；
3. 元类对象的isa指向元类对象所属的类，也就是根元类；
4. 根元类的isa指向本身。


好了有了上面的理解，我们再来一一解答之前的题目。


* BOOL isEqual  = [p isKindOfClass:[Animal class]]; 
  

```

1. 由于调用的是`- isKindOfClass`方法，所以【p isKindOfClass：】是取出p的isa，由于对象的isa指向的所属的类，也就是Animal类；
2. [Animal class]调用的是`+class`方法，返回是本身，也就是Animal类，所以isEqual == YES.

```
    
* BOOL isEqual1 = [Animal isKindOfClass:[Animal class]]; 


```
1. [Animal isKindOfClass],结合`+isKindOfClass`的定义，取得是Animal的isa，所以是Animl的元类；
2. [Animal class]取的是Animal类，所以isEqual1 == NO；
```

* BOOL isEqual2 = [NSObject isKindOfClass:[NSObject class]]; 


```
1. [NSObject class]取的是类NSObject ；
2. [NSObject isKindOfClass]取得是NSObject的isa也就是NSObject的元类，此时为NO；
3. 由`isKindOfClass`的定义会去寻找NSObject的元类的父类，也就是NSObject，正好等于NSObject 所以isEqual2 == YES.

```

* BOOL isEqual3 = [p isMemberOfClass:[Animal class]];  


```
1. [Animal class] - > Animal;
2. [p isMemberOfClass] -> Animal 所以 isEqual3 == YES
```

* BOOL isEqual4 = [Animal isMemberOfClass:[Animal class]]; 

```
1. [Animal class] - > Animal;
2. [Animal isMemberOfClass] -> Animal 的元类 所以isEqual4 == NO
```

* BOOL isEqual5 = [NSObject isMemberOfClass:[NSObject class]]; 

```
1. [NSObject class] - > NSObject;
2. [NSObject isMemberOfClass:] - >  NSObject的元类 所以 isEqual5 == NO
```

* BOOL isEqual6 = [(id)[NSObject class] isKindOfClass:[NSObject class]]; 

```
1. [NSObject class] - > NSObject;
2. (id)[NSObject class] - >   NSObject;
3. [(id)[NSObject class] isKindOfClass] - > NSObject的元类 不相等；
4.  NSObject的元类的父类 - >  NSObject 所以 isEqual6 == YES
```

* BOOL isEqual7 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];

```
1. [NSObject class] - > NSObject;
2. (id)[NSObject class] - >   NSObject;
3. [(id)[NSObject class] isMemberOfClass] - > NSObject的元类 不相等 所以 isEqual7 == NO；
```
* BOOL isEqual8 = [(id)[Dog class] isKindOfClass:[Dog class]]; 

```
1. [Dog class] - > Dog
2. (id)[Dog class] - > Dog
3. [(id)[Dog class] isKindOfClass] - > Dog 的元类 不相等；
4. Dog 的元类 的父类 - > Animal的元类的父类 不等于Dog；
5. Animal的元类的父类的父类 -> 根元类 还是不等于 Dog；
6. 跟元类的父类 - > NSObject 不等于Dog  
7. NSObject的父类  - > nil  所以isEqual8 == NO。

```
* BOOL isEqual9 = [(id)[Dog class] isMemberOfClass:[Dog class]];
    
```
1. [Dog class] - > Dog
2. (id)[Dog class] - > Dog
3. [(id)[Dog class] isMemberOfClass] - > Dog 的元类 不相等 所以isEqual9 == NO；
4. 
```
    
相信到这里大家应该有个很好的认识了。


参考：
http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html

