---
layout: post
title: "OC--方法缓存"
date: 2018-09-04 17:12:38 +0800
comments: true
categories: 
tag:
    [Objective-C,cache_t]
---


有过一些oc基础的开发对oc的消息机制应该都是不陌生的，调用某个对象的方法其实就是给对象发送一条消息，然后接收的对象根据接收到的方法，查找自己的方法列表，然后调用的过程。那oc是如何查找方法的？，是每一次都是去遍历自己的方法列表吗，那这样岂不是效率很低？，如何才能提高查找效率呢？我相信你也有这样的疑惑的，本篇将带来苹果的底层是如何实现的。

<!-- more -->

## 预备知识

**开始之前你需要先知道以下几点**

#### 1. 什么是SEL

先看下下面的代码

```
NSLog(@"----%p--%p---%p--%p",@selector(kk),@selector(kk),@selector(kk),@selector(kk));

NSLog(@"----%ld--%ld---%ld--%ld",(unsigned long)@selector(kk),(unsigned long)@selector(kk),(unsigned long)@selector(kk),(unsigned long)@selector(kk));
```

发现输出的结果都是一样的

```
----0x100000f5f--0x100000f5f---0x100000f5f--0x100000f5f
----4294971231--4294971231---4294971231--4294971231
```

好像这样也不能说明SEL到底是什么，这个问题先放在这里，看完整个文章你就会明白到底是什么了。到目前为止，我们可以看出的是，同一个方法的SEL是一样的，转成（unsigned long）也是一样的，目前知道这么多已经足够了

#### 2. 对象方法是放在类对象中的，类方法是放在元类对象中的。

不清楚的可以看下我的这篇[介绍](https://www.jianshu.com/p/14b096d6a233)

#### 3. 对象的内存结构

```
struct objc_class : objc_object {
    void *isa; // 由于isa不是本文所讲的范围，这里直接写成这种形式，类似oc中的id
    Class superclass;
    cache_t cache;             // 本文的重点是这个属性
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
        ...
}
```

#### 4. 最好有些c++语法的基本认识，推荐这篇[文章入门](http://www.dotcpp.com/course/cpp/200001.html)

#### 5. 关于runtime源码，可以[点击下载](https://github.com/RetVal/objc-runtime)，可以直接运行


## 演示代码


```
//Person.h
#import <Foundation/Foundation.h>
@interface Person : NSObject

- (void)test1;

- (void)test2;

//Person.m
- (void)test1 {
    NSLog(@"--%s",__func__);
}

- (void)test2 {
    NSLog(@"--%s",__func__);
}
@end
```


由于我的mac是beta版本,源码跑不起来，但是不影响查看代码，下面struct相关的代码是我把runtime代码和本文相关的部分复制了一份下来，方便查看信息


```
#import <Foundation/Foundation.h>
#import "Person.h"
#import <objc/runtime.h>

#if __LP64__
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
#else
typedef uint16_t mask_t;
#endif
typedef uintptr_t cache_key_t;

struct bucket_t {
    cache_key_t _key;
    IMP _imp;
};

struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;

};

struct class_data_bits_t {
    
    uintptr_t bits;
};

struct my_objc_object {
    void * isa;
};

struct my_objc_class: my_objc_object  {
    void * isa;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

};


int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
    }
    
    return 0;
}

```

## 正文

苹果对于方法的缓存方式是通过[散列表](https://baike.baidu.com/item/%E5%93%88%E5%B8%8C%E8%A1%A8/5981869?fromtitle=%E6%95%A3%E5%88%97%E8%A1%A8&fromid=10027933&fr=aladdin)实现的，所谓散列表就是通过一个函数，计算出某个元素应该放在哪个位置，或者从哪个位置取出的存储方式，从而提高查找速度。举个例子，现有一个长度为10的数组`arr`，


```
NSMutableArray *arr = [NSMutableArray arrayWithCapacity:10];

```
现在分别有`1`、`20`、`3`、`9`四个数字需要存放到数组中,这4个数字该放在哪个位置呢，我们定义一个函数，来计算4个数字应该存放的位置,这里我们简单的将数字的除以10的余数作为索引


```
int indexWithEle(int ele) {
    return ele % 10;
}
```

计算的余数分别是

```
1
0
3
9
```

分别放到对应的位置那么`arr`数组的元素分布如图

![15360433915836.jpg](https://upload-images.jianshu.io/upload_images/3279997-c3a859200bd23a19.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

比如我们要`9`，那也是通过`indexWithEle`函数计算出索引为`9`那么直接arr[9]直接就拿到元素了，就不用遍历数组了

苹果对于方法的缓存也是差不多的，一些小的差异后面会说到，方法的缓存信息都是放在`cache`中的

```

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

}
```

`cache_t`的结构如下

```
struct bucket_t {
    cache_key_t _key;
    IMP _imp;
}

struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
}
```

1. `_buckets`是存放的方法的信息；
2. `_mask`就相当于上面例子中的`10`，这里取得是`_buckets`的长度`-1`
3. `_occupied`表示`_buckets`拥有的元素的个数

`cache_t`结构如图

![15360441734433.jpg](https://upload-images.jianshu.io/upload_images/3279997-2e1c9fadf2836a47.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


刚才说到实现散列表缓存的话需要：

1. 一个计算索引的函数;
2. 一个集合


苹果计算索引的函数是将函数的`SEL`的数值`&`上`_mask`的结果

```
static inline mask_t cache_hash(cache_key_t key, mask_t mask) 
{
    return (mask_t)(key & mask);
}

```

`cache_key_t`其实就是`unsigned long`上面讲到一个`SEL`的数值是一样的`mask`是`_buckets`长度减1。

要知道一点就是c = a&b ，那么c应该是<= b的，所以`mask`是`_buckets`的长度减1，之前我们还有一个细节没考虑到那就是万一&出来的索引值已经有了怎么处理，苹果的`__arm64__`实现是直接将索引-1


```
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0);

    bucket_t *b = buckets();
    mask_t m = mask();
    mask_t begin = cache_hash(k, m);
    mask_t i = begin;
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[I];
        }
    } while ((i = cache_next(i, m)) != begin);

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}

```

如果索引相同则直接-1

```

#elif __arm64__
// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
```

函数已经有了那么集合呢看下面的方法


```
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{

    cache_t *cache = getCache(cls);
    cache_key_t key = getKey(sel);

    mask_t newOccupied = cache->occupied() + 1;
    mask_t capacity = cache->capacity();
    if (cache->isConstantEmptyCache()) {// 如果是空的则初始化一个`INIT_CACHE_SIZE`大小的数组也就是1<<2 == 4
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE); 
    }
    else if (newOccupied <= capacity / 4 * 3) { // 当使用的元素个数超过总数的3/4的时候则会扩容

    }
    else {
        // Cache is too full. Expand it.
        cache->expand();
    }

   
    bucket_t *bucket = cache->find(key, receiver);
    if (bucket->key() == 0) cache->incrementOccupied();
    bucket->set(key, imp);
}

```

现在数组也有了，还有一个问题，比如一开始初始化的数组大小是4，当元素大于4的时候如何处理呢，苹果的处理是当元素的个数达到总数的3/4的时候就会扩容了


```
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}

```


也就是按照`4`、`8`、`16`...扩张下去，那么现在所有的东西都齐全了，以上说的这些都是理论，下面将会通过例子来一一验证正确性

将代码修改如下

![15360504512073.jpg](https://upload-images.jianshu.io/upload_images/3279997-138ab38559db51bf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们来看看`p`的`test2`方法是放在哪里的，图中红色标出的地方，由于对象的方法是放在类对象中的，红色部分要写类对象，`[p class]` 、 `[Person class]` 、`objc_getClass(p)`都是可以的。

![15360506603796.jpg](https://upload-images.jianshu.io/upload_images/3279997-2f03e281fad6f329.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
(lldb) p (unsigned long)@selector(test2) // test2的数值
(unsigned long) $0 = 4294971225
(lldb) p 4294971225 & 3 // 计算出test2对应的索引  也就是mask的值
(long) $1 = 1  // 结果为1
(lldb) p objP->cache._buckets[1] // 我们取出1的值
(bucket_t) $2 = {
  _key = 4294971225
  _imp = 0x0000000100000d80 (Isa`-[Person test2] at Person.m:17)
  
```

上面我们通过自己计算得到了`test2`位于`_buckets`的第一个位置，也确实正确取出来了，有个地方需要注意一下，这里我们正好没有重复的索引，所以正好对了，当有重复的时候可能要-1或者--1 ... 了

再修改代码如下

![15360509867557.jpg](https://upload-images.jianshu.io/upload_images/3279997-9dce29bc54b3e560.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


具体

![15360510002579.jpg](https://upload-images.jianshu.io/upload_images/3279997-e7845566b0a2cba3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


发现此时的`_mask`为7了，`_occupied`为`1`了，当

```
p objP->cache._buckets[1]
(bucket_t) $1 = {
  _key = 140734070273825
  _imp = 0x00007fff622014b7 (libobjc.A.dylib`-[NSObject class])
}

```
说明此时的缓存只有`[p class]`了,这也正好验证了当超过3/4时会扩容,到62行的时候缓存的方法有`init`、`test2`、`test1`,此时达到总数`4`的3/4`3`所以扩容为`4`* 2 = `8`


好了到这里也就明白了方法缓存的过程了，对于什么是`SEL`，相信也有认识了，其实就是一个编号，用来计算最终方法的存放位置的他的value就是函数的地址










