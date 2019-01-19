---
layout: post
title: "如何实现一个自己的for...in...操作呢"
date: 2017-06-25 18:52:07 +0800
comments: true
categories: 
tag:
    Objective-C
---

![](http://upload-images.jianshu.io/upload_images/3279997-59defae7d37729ed.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!--more-->

需求：对于一个自定义类如何也可以想和`NSArray`和`NSDictionary`一样可以直接遍历？

本篇目录：

1. 解析系统for...in...的实现原理；
2. 自己实现一个for...in...的类;
3. 简单解释一下`objc_enumerationMutation`是如何抛出异常的。



### 1. 解析系统for...in...的实现原理

我们来看看苹果在2.0推出来的Fast Enumeration。
[引用苹果官方文档的一段总结](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Articles/Enumerators.html#//apple_ref/doc/uid/20000135-SW1)

> The enumeration is more efficient than using NSEnumerator directly
> The syntax is concise.
> The enumerator raises an exception if you modify the collection while enumerating.
> You can perform multiple enumerations concurrently.

翻译过来就是
* 它比之前的`NSEnumerator`更高效
* 语法更简洁
* 如果这个集合在遍历的过程中修改了，会抛出异常
* 可以同时执行对个枚举

`Fast Enumeration`是一个协议

    ```
    @protocol NSFastEnumeration
    
    - (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state 
                                      objects:(id __unsafe_unretained _Nullable [_Nonnull])buffer 
                                        count:(NSUInteger)len;
    
    @end
    
    ```

这个方法的作用是根据具体数据个数返回一定数量的数组供调用者使用的，为什么是一定数量的数组呢，比如说数据源是有5个数据[@"1",@"2",@"3",@"4",@"5"],若调用者想要2个一组，那么需要3组才能完成；前面提到的一组，其实就是C语言的数组，而这个方法就是用来确定返回一个怎样的数组，方法的返回值就是对应数组的长度。

这个协议方法传3个参数分别是：

 1.  `state`，它是个结构体；

    ```
     typedef struct {
            unsigned long state;
            id __unsafe_unretained _Nullable * _Nullable itemsPtr;
            unsigned long * _Nullable mutationsPtr;
            unsigned long extra[5];
        } NSFastEnumerationState;
    ```
      * `state`这个参数在for...in...的方法内部是没有使用的，是留给调用者备用的，用来记录一些状态；
      * `itemsPtr`就是C数组的指针，它和方法的返回共同构成了C语言数组；
      * `mutationsPtr`这个字段是用来记录在遍历的过程中，被遍历的对象有没有被改变，从而可以抛出异常；
      * `extra`这个和`state`字段一样，在for...in...的方法内部是没有使用的也是没有使用的，留给调用者使用的。

 2. `buffer`它是一个缓冲区，其实是一个C数组，因为在内存中不是所有的对象都是内存连续的，针对那些内存不连续，方法提供一个内存区域，调用者把数组都放到这个缓冲区，他的长度由len决定；
 3. `len`上面已经提到就是定义`buffer`长度的。

 
 为何我会这样解释这些属性呢，我们来看看for...in...的C++实现，就可以一一验证上面的说法了
 
 先写一个demo
 
```
#import <Foundation/Foundation.h>

int main(int argc, char * argv[]) {
    @autoreleasepool {
        
        NSMutableArray *arry = [NSMutableArray arrayWithObjects:@"1",@"2",@"3", nil];
        for (NSString *str  in arry) {
            NSLog(@"dddd---%@",str);
        }
        return 0;
        
}
```
使用命令

```
clang -rewrite-objc main.m

```
可以得到

```

int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        NSMutableArray *arry = ((NSMutableArray *(*)(id, SEL, ObjectType, ...))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("arrayWithObjects:"), (id)(NSString *)&__NSConstantStringImpl__var_folders_g3_yzsthm0x1xs8k_ycscbf93hr0000gn_T_main_6d9606_mi_0, (NSString *)&__NSConstantStringImpl__var_folders_g3_yzsthm0x1xs8k_ycscbf93hr0000gn_T_main_6d9606_mi_1, (NSString *)&__NSConstantStringImpl__var_folders_g3_yzsthm0x1xs8k_ycscbf93hr0000gn_T_main_6d9606_mi_2, __null);
        {
	NSString * str;
	
	// 这个就是传出去的State
	struct __objcFastEnumerationState enumState = { 0 };
	
	//这个就是buffer，可以看出是初始化了一个16长度的数组
	id __rw_items[16];
	
	//这个表示调用哪个对象的"countByEnumeratingWithState:objects:count:"方法
	id l_collection = (id) arry;
	
	//limit就是countByEnumeratingWithState:objects:count:返回值
	_WIN_NSUInteger limit =
		((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
		((id)l_collection,
		sel_registerName("countByEnumeratingWithState:objects:count:"),
		&enumState, (id *)__rw_items, (_WIN_NSUInteger)16);
		
		
	if (limit) {
	
	// 这里面有两个do...while...循环
	// 外层循环用来获取一共需要的数组的个数，并获取对应数组的长度
	// 内部循环是遍历获取对应数组的元素进行下一步操作
	
	        //startMutations就是用来监控遍历的过程中遍历对象有没有改变
	       unsigned long startMutations = *enumState.mutationsPtr;
        	do {
        		      unsigned long counter = 0;
        		do {
        		
        		//如果遍历对象发生了改变就会调用`objc_enumerationMutation`来抛出异常
        			if (startMutations != *enumState.mutationsPtr)
        				objc_enumerationMutation(l_collection);
        				
        			// 取出对应的元素，这是高效快速的关键
        			str = (NSString *)enumState.itemsPtr[counter++]; 
        			{
                    NSLog((NSString *)&__NSConstantStringImpl__var_folders_g3_yzsthm0x1xs8k_ycscbf93hr0000gn_T_main_6d9606_mi_3,str);
                };
                // 结束这次循环，进行下一次
        	__continue_label_1: ;
        		} while (counter < limit);
        	} 
        	
      while ((limit = ((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
        		((id)l_collection,
        		sel_registerName("countByEnumeratingWithState:objects:count:"),
        		&enumState, (id *)__rw_items, (_WIN_NSUInteger)16)));
        	str = ((NSString *)0);
        	__break_label_1: ;
	}else
		str = ((NSString *)0);
	}

        return 0;


    }
}


```
 
 对于上面的源码进行了一些必要的注释帮助大家理解，整个方法下来，并没有看到`state`和`extra`字段，这也验证了之前的说法。
    

### 2. 自己实现一个for...in...的类

这里参照[苹果的官方demo](https://developer.apple.com/library/content/samplecode/FastEnumerationSample/Introduction/Intro.html#//apple_ref/doc/uid/DTS40009411)写了一个简单的例子
对于`countByEnumeratingWithState:objects:count:`我们有两种方法来实现
1. 对于在内存中连续的结合来说可以直接返回这段内存的首地址；
2. 对于不连续的来说，这个时候就要使用`buffer`了，接下来分别给出两种方式

.h

```
#import <Foundation/Foundation.h>

@interface MyFastIterator : NSObject<NSFastEnumeration>


@end
```
.m

```
#import "MyFastIterator.h"
#include <vector>
#import <objc/message.h>

@interface MyFastIterator ()

@property (nonatomic, strong) NSArray *myArray;

@property (nonatomic, assign) long tagSi;

@end

@implementation MyFastIterator
{
    std::vector<NSNumber *> _list;
}

- (instancetype)init {
    if (self = [super init]) {
        for (NSUInteger i = 0; i < 17; i++) {
            _list.push_back(@(i));
        }
        
        self.myArray = @[@"1",@"2",@"3",@"4",@"5",@"6",
                         @"1",@"2",@"3",@"4",@"5",@"6",
                         @"1",@"2",@"3",@"4",@"5",
                         ];
        
    
        
    }
    return self;
}


```

`countByEnumeratingWithState:objects:count:`实现

```

#define USE_BUFF 1

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id  _Nullable __unsafe_unretained [])buffer count:(NSUInteger)len {
 NSUInteger count = 0;
    
    unsigned long countOfItemsAlreadyEnumerated = state->state;
    
    if (countOfItemsAlreadyEnumerated == 0 ) {// 等于0说明是第一次调用可以初始化一些数据
        NSLog(@"countByEnumeratingWithState");
        // 上面已经说到，mutationsPtr是用来记录遍历的过程中被遍历的对象有没有被修改的
        // 由于我们这里是NSArray是不可变的，所以无需追踪他的改变
        // 从而这里取 的是 &state->extra[0];
        state->mutationsPtr = &state->extra[0];
    }
#if USE_BUFF
    if (countOfItemsAlreadyEnumerated < self.myArray.count) {
        state->itemsPtr = buffer;
        
        while (countOfItemsAlreadyEnumerated < self.myArray.count && count < len){
//            NSLog(@"--%d",count);
            buffer[count] = self.myArray[countOfItemsAlreadyEnumerated];
            countOfItemsAlreadyEnumerated++;
            count++;
        }
    }else {
        count = 0;
    }
#else
    
    if (countOfItemsAlreadyEnumerated < _list.size()) {
        
        // 直接将 state->itemsPtr 指向内部的 C 数组指针，因为它的内存地址是连续的
        __unsafe_unretained const id * const_array = _list.data();
        
        state->itemsPtr = (__typeof__(state->itemsPtr))const_array;
        
        // 因为我们一次性返回了 _list 中的所有元素
        // 所以，countOfItemsAlreadyEnumerated 和 count 的值均为 _list 中的元素个数
        
        //  这里使用的是官方demo的写法
        countOfItemsAlreadyEnumerated = _list.size();
        count = _list.size();
    }else {
        count = 0;
    }
    
#endif
    
    state->state = countOfItemsAlreadyEnumerated;
    return count;

}
```

外面调用

```
- (void)testMyFastIterator {
    MyFastIterator *fast = [[ MyFastIterator alloc] init];
    for (NSNumber *num in fast) {
        NSLog(@"testMyFastIterator---%@",num);
    }
}
```

其实还有一种简单的写法，直接返回要遍历的对象的方法，前提是遍历的对象实现了`countByEnumeratingWithState`方法

```
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id  _Nullable __unsafe_unretained [])buffer count:(NSUInteger)len {
    
   return [self.myArray countByEnumeratingWithState:state objects:buffer count:len];
}
```

### 3. 简单解释一下`objc_enumerationMutation`是如何抛出异常的。



`objc_enumerationMutation`方法是如何抛出异常的呢，打开objc4-646的源码中可以看到具体实现


```


static void (*enumerationMutationHandler)(id);


void objc_enumerationMutation(id object) {
    if (enumerationMutationHandler == nil) {
        _objc_fatal("mutation detected during 'for(... in ...)'  enumeration of object %p.", (void*)object);
    }
    (*enumerationMutationHandler)(object);
}


void objc_setEnumerationMutationHandler(void (*handler)(id)) {
    enumerationMutationHandler = handler;
}

```
阅读源码可以得出看出：

1. `objc_setEnumerationMutationHandler`方法接收一个函数指针，保存在内部定义的之前声明好的函数`static void (*enumerationMutationHandler)(id);`；
2. `objc_enumerationMutation`被调用的时候，如果调用者没有实现`objc_setEnumerationMutationHandler`的话，此时函数指针`enumerationMutationHandler`为nil，就会执行`_objc_fatal("mutation detected during 'for(... in ...)'  enumeration of object %p.", (void*)object);`，否则就会通过`*enumerationMutationHandler`拿到函数并把`object`传递出去。

我们来看一个demo

```
//先初始化一个函数
void voidVoidTest(id objt) {
    NSLog(@"%@挂啦",objt);
}

- (void)testMutation {
    void (*funcVoidVoid)() = &voidVoidTest;
    objc_setEnumerationMutationHandler(funcVoidVoid);
    NSString *str = @"test";
    objc_enumerationMutation(str);
   } 
   
```

输出

```
test挂啦
```

