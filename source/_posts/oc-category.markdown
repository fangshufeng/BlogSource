---
layout: post
title: "OC--分类"
date: 2018-08-12 15:19:56 +0800
comments: true
categories: 
tag:
    Objective-C
---

本篇是解决[上一篇](http://fangshufeng.com/blog/2018/08/02/oc-instance-metaclass/)的第一个问题的；

读过我[实例对象、类对象和元类对象是如何分工的](http://fangshufeng.com/blog/2018/08/02/oc-instance-metaclass/)的朋友应该知道一个类的实例方法是存放在`类对象`中的，类方法是存在`元类对象`中，这些是在编译阶段就能确定的，但是我们并没有发现分类的方法，也就有了下面的疑问：

1. 分类的加载时机；
2. 分类中的方法存放的地方；
3. 如果分类和原来的类拥有共同的方法，调用结果如何；
4. 如果同一个类的多个分类拥有同一个方法，调用结果如何；
5. load方法是如何加载的；
6. 继承中的load方法是如何调用的；
7. initialize方法如何加载的；
8. 综合结论

<!-- more -->

本文的篇幅有些长，请读者耐心看,看本篇文章前请先读[实例对象、类对象和元类对象是如何分工的](http://fangshufeng.com/blog/2018/08/02/oc-instance-metaclass/)，建议读者按照我的顺序自己新建类

---

#### 准备工作

新建项目

```
//Animal.h

#import <Foundation/Foundation.h>

@interface Animal : NSObject

@property (nonatomic, copy) NSString *name;

- (void)eat;

@end


//Animal.m
#import "Animal.h"

@implementation Animal

- (void)eat {
    NSLog(@"Animal--eat");
}

@end

//Animal+Test.h
#import "Animal.h"
@interface Animal (Test)

@property (nonatomic, copy) NSString *testName;

- (void)eat ;
@end

//Animal+Test.m
#import "Animal+Test.h"

@implementation Animal (Test)

- (void)eat {
    NSLog(@"Animal(Test)--eat");
}

@end

```

通过`clang`命令拿到`Animal+Test`的cpp代码

```
struct _category_t {
    const char *name;
    struct _class_t *cls;
    const struct _method_list_t *instance_methods;
    const struct _method_list_t *class_methods;
    const struct _protocol_list_t *protocols;
    const struct _prop_list_t *properties;
};

```

这里只取一些关键的代码，有段下面的代码

```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_Animal_$_Test,
};

```

`L_OBJC_LABEL_CATEGORY_$ [1]`也就是`_category_t`的`cls`字段，所以上面的意思就是，这个分类是`_OBJC_$_CATEGORY_Animal_$_Test`类的，并且存储在`__DATA`的`__objc_catlist`中。


#### 一、分类的加载时机

这里需要查看`runtime`的源码，这里给出加载的分类的方法查找路径

    * void _objc_init(void)
    * map_images
    * map_images_nolock
    * _read_images
    * remethodizeClass
    * attachCategories
    
读者可以顺着这个查看下源码,这里会给出一些关键性的代码，看到`_read_images`中的

```
 category_t **catlist = _getObjc2CategoryList(hi, &count);
 
```

`_getObjc2CategoryList`的内容是


```
GETSECT(_getObjc2CategoryList,        category_t *,    "__objc_catlist");
```

之前我们的分类信息是存放在`__DATA`的`__objc_catlist`中，这里然后从这里读取

**从而说明：**

分类是在`runtime`加载的

---

#### 二、分类中的方法存放的地方

上面得出分类的信息是在`runtime`阶段加载的，下面看看加载具体方式，进入`attachCategories`中，为了方便查看，这里只留下了方法相关的代码，读者可以对比源码

```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[I];
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
  }

```


1. `cats->list`是某个类的所有分类
2. `mlist`是某个分类的方法列表
2. `while (i--) `中的`i--`说明是反向遍历，后编译的会在数组的最前面；
3. `mlists[mcount++] = mlist;`说明`mlists`是一个二维数组，

`mlists`最后相当于以下结构

```
mlists = [
        [分类1的方法列表]，
        [分类2的方法列表]，
        ...
    ]
```

接着通过

```
 rw->methods.attachLists(mlists, mcount);
```

将分类的方法附在原来类的方法列表上

`attachLists`内容如下


```
 void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
       
```

这里运行了2个c语言的函数`memmove`和`memcpy`，这2个函数做的事情如下

![15340496357538.jpg](https://upload-images.jianshu.io/upload_images/3279997-c54e087475427010.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**从而说明：**

分类的方法还是加到原来的`类对象`和`元类对象`的方法列表中的；

---

#### 三、如果分类和原来的类拥有共同的方法，调用结果如何

编译之前的工程，会发现输出

```
Animal(Test)--eat
```

会发现是输出分类的打印，由上面二的分析，很容易知道是这个答案，在加载分类的方法是反向遍历的，从而导致了分类的方法会调用，原来的方法不会调用


**所以**

如果分类和原来的类拥有共同的方法，调用结果如何？

答案是：**调用分类的方法**

---

#### 四、如果同一个类的多个分类拥有同一个方法，调用结果如何

我们新加一个`Animal`的分类`Animal+Test2`


```
//Animal.h
#import "Animal.h"
@interface Animal (Test2)

@end

//Animal.m
#import "Animal+Test2.h"

@implementation Animal (Test2)

- (void)eat {
    NSLog(@"Animal(Test2)--eat");
}

@end

```

这个时候运行程序会发现输出的还是

```
Animal(Test)--eat
```

这是什么原因呢？通过二我们知道，加载分类的方法是反向遍历的，说明`Animal(Test)`是比`Animal(Test2)`后编译的，通过查看`targets` -> `Build Phases` -> `Compile Sources`证明了这个观点

![15340505416518.jpg](https://upload-images.jianshu.io/upload_images/3279997-f84ee80fa42ca4e6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们调整下顺序

![15340505689740.jpg](https://upload-images.jianshu.io/upload_images/3279997-1989bced6c9b041a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

会发现这个时候输出

```
Animal(Test2)--eat
```

**所以**

如果同一个类的多个分类拥有同一个方法，调用结果如何？

答案是：**执行后编译的*

---

#### 五、load方法是如何加载的

我们分别在`Animal`、`Animal+Test`和`Animal+Test2`都增加个`+load`方法，运行程序却发现如下打印

```
Animal--load
Animal(Test)--load
Animal(Test2)--load
```

如果按照二的理解不应该是只会打印`Animal(Test2)--load`的吗？还是要回到`runtime`源码


    *   _objc_init
    *   load_images
    *   prepare_load_methods
    *   schedule_class_load
    *   call_load_methods

先看`prepare_load_methods`


```
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, I;

    runtimeLock.assertWriting();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[I];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```

1. `classlist`装着所有`类`的load方法；
2. `categorylist`装着所有`分类`的load方法；


`schedule_class_load`内容如下


```
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}

```

` schedule_class_load(cls->superclass);`保证了拿类的`load`方法前会先去拿父类的`load`方法

再到`call_load_methods`方法


```
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}

```


1. 先调用`call_class_loads`也就是之前存储在`classlist`的`类`的`load`方法；
2. 再调用`call_category_loads`也就是之前存储在`categorylist`的`分类`的`load`方法


**从而**

关于load方法有以下特点

1. 是直接通过指针调用的，
2. 程序一启动就会调用,并且只会调用一次；
3. 所有的load方法都会调用，
4. 先调用类的load方法，类的load方法中又会先调用父类的load方法，然后调用分类的load方法
5. 分类的load方法调用的顺序是编译的顺序


所以一开始的现象也就解释了

---


#### 六、继承中的load方法是如何调用的

我们新增一个`Animal`的子类`Dog`,并且增加`Dog+Test`和`Dog+Test2`

编译顺序如下

![15340520050677.jpg](https://upload-images.jianshu.io/upload_images/3279997-257494b5b818c88f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实知道五以后，都能猜到答案了，你猜对了吗


```
Animal--load
Dog--load
Animal(Test)--load
Animal(Test2)--load
Dog(Test)--load
Dog(Test2)--load

```

`Dog`的父类是`Animal`，所以`Dog`的load方法会最先调用，由于类的load方法优先分类的，从而接着是`Dog--load`，最后是分类的，按照编译的顺序执行

关于这些分类的组合有很多，关键还得理解本质，读者可自行组合分析

#### 七、initialize方法如何加载的

把之前的`load`方法都注释掉，换成`initialize`方法，`main.m`代码如下


```
#import <Foundation/Foundation.h>

#import "Animal.h"
#import "Dog.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [Animal alloc];
        [Dog alloc];

        
    }
    return 0;
}
```

输出

```
Animal(Test2)--initialize
Animal(Test2)--initialize
```


可以看到我们并没有显示调用`initialize`，但是这个方法却调用了，我们知道`[Animal alloc]`是调用了`objc_msgSend()`方法，说明应该是这里做了处理了，相关类如下

    * class_getInstanceMethod
    * lookUpImpOrNil
    * lookUpImpOrForward
    * _class_initialize

  `lookUpImpOrForward`如下
  
``` 
...
 if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlockRead();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.read();
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }    
...

```
`_class_initialize`如下

```
void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
....   
}
```

这里做了很多省略，从这里可以看出：

1. 当对象第一次收到消息的时候会调用`_class_initialize`，并且会记住已经初始化过了
2. 当父类的`_class_initialize`没有调用时会初始化父类的调用；



所以

```
Animal(Test2)--initialize
Animal(Test2)--initialize
```

解释如下

1. `[Animal alloc]`发现`isInitialized`是`NO`,从而调用`initialize`方法，但是分类的优先级高，原因见二；
2. `[Dog alloc]`发现父类的`isInitialized`是`YES`,但是自己没有`initialize`方法，会去调用父类的`initialize`方法，回到了1，所以父类的会调用2遍


#### 八、综合结论

1. 常规方法

    initialize方法
    
    * 先初始化父类的；
    * 再初始化子类的，如果子类没有实现，父类实现了会被多次调用，使用的时候需要注意
    
    其他方法
    
    * 原类和分类有共同方法，只会执行分类的方法；
    * 同一个类的多个分类拥有共同的方法，执行最后编译的分类方法

2. load方法
    * 先执行类的load方法；
    * 再执行分类的load方法；
    * 多个分类的load方法，先编译的先执行；
    * 调用子类的load方法前，会调用父类的load方法





