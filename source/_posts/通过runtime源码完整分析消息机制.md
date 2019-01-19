---
title: 通过runtime源码完整分析消息机制
date: 2018-12-24 21:04:24
tags:
    [Objective-C, runtime]
---


## 一、前言

1. 本文主要分析当我们调用`[p test1]`的过程中，runtime是如何调用的。
2. 本文的调试代码[地址](https://github.com/fangshufeng/demo_project/tree/master/runtime_project)
3. 由于runtime源码无法正常跑在真机上，本文是通过断点`x86`代码来类比分析`arm64`。
4. 本文的代码是`objc-750`和之前的`480`有些不一样的地方;

<!-- more -->

## 二、缓存查找

先添加如下测试代码

![15454413434857.jpg](https://upload-images.jianshu.io/upload_images/3279997-ab3ede58bbf13a31.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


`23`行添加断点

![15454419571462.jpg](https://upload-images.jianshu.io/upload_images/3279997-49bbc1d566165712.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


点击运行程序，程序将断点在`23`行


### 2.1、计算方法`test1`的索引

![15454455754798.jpg](https://upload-images.jianshu.io/upload_images/3279997-43f90e41fca74089.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


1. 计算方法`test1`的缓存索引 ，`4294971225 & 3 == 1`  ；
2. 先打印索引`1`的地方有没有占用，可以看到索引为`1`被`init`方法占用了，由于是`x86`架构索引将`4294971225`➕`1`然后再`& 3` == 2，输出为2的位置的`_key == 0`，所有方法`test1`索引是`2`（要是`arm64`架构的话就是-1了），关于如何缓存的请看[这篇](https://fangshufeng.com/2018/09/04/cache/)


```c
#if __arm__  ||  __x86_64__  ||  __i386__
    #define CACHE_END_MARKER 1
    static inline mask_t cache_next(mask_t i, mask_t mask) {
        return (i+1) & mask;
    }
    
    #elif __arm64__
    #define CACHE_END_MARKER 0
    static inline mask_t cache_next(mask_t i, mask_t mask) {
        return i ? i-1 : mask;
    }
```


###  2.2、首次调用`test`（没有缓存的汇编代码）

在`objc-msg-x86_64.s`中打下断点


![15454460158277.jpg](https://upload-images.jianshu.io/upload_images/3279997-67876c05f635f497.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


继续调试会到宏`CacheLookup`代码处,由于此时传的是`NORMAL`，由于我们的代码最终是跑在真机上的，所以我这里还是分析`arm64`的汇编吧，`x86`汇编留给你自己分析吧，读懂`arm64`的汇编代码，`x86`汇编不在话下区别就是使用的汇编写法不同（x86是AT&T汇编）

#### 2.2.1、`objc-msg-arm64.s`汇编代码如下


```arm

.macro CacheLookup
	// p1 = SEL, p16 = isa
	ldp	p10, p11, [x16, #CACHE] // p10 = buckets, p11 = occupied|mask
	
#if !__LP64__
	and	w11, w11, 0xffff	// p11 = mask
#endif
	and	w12, w1, w11		// x12 = _cmd & mask
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
	add	p12, p12, w11, UXTW #(1+PTRSHIFT)
		                        // p12 = buckets + (mask << 1+PTRSHIFT)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// double wrap
	JumpMiss $0
	
.endmacro

```

如果你觉得一头雾水的话我带你一步一步看汇编，首先初步看到这个汇编代码会发现出现很多不知道的寄存器比如`p`是什么鬼，`PTRSHIFT`又是什么鬼，原来在`objc-750`中，苹果使用宏重定义了，其实就是`x`，可以查看`arm64-asm.h`头文件可以看到，由于我们是`arm64`的所以也就是下面的代码


```arm
#if __arm64__

#if __LP64__
// true arm64

#define SUPPORT_TAGGED_POINTERS 1
#define PTR .quad
#define PTRSIZE 8
#define PTRSHIFT 3  // 1<<PTRSHIFT == PTRSIZE
// "p" registers are pointer-sized
#define UXTP UXTX
#define p0  x0
#define p1  x1
#define p2  x2
#define p3  x3
#define p4  x4
#define p5  x5
#define p6  x6
#define p7  x7
#define p8  x8
#define p9  x9
#define p10 x10
#define p11 x11
#define p12 x12
#define p13 x13
#define p14 x14
#define p15 x15
#define p16 x16
#define p17 x17

// true arm64
#else
```

这里还是不厌其烦的我还是把宏替换成习惯的形式来方便理解，替换后的结果如

```arm
.macro CacheLookup
	// x1 = SEL, x16 = isa
	ldp	x10, x11, [x16, #16]	// x10 = buckets, x11 = occupied|mask

	and	w12, w1, w11		// x12 = _cmd & mask
	add	x12, x10, x12, LSL #(1+3)
		             // x12 = buckets + ((_cmd & mask) << (1+3))

	ldp	x17, x9, [x12]		// {imp, sel} = *bucket
1:	cmp	x9, x1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	x12, x10		// wrap if bucket == buckets
	b.eq	3f
	ldp	x17, x9, [x12, #-16]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: x12 = first bucket, w11 = mask
	add	x12, x12, w11, UXTW #(1+3)
		                        // x12 = buckets + (mask << 1+3)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	x17, x9, [x12]		// {imp, sel} = *bucket
1:	cmp	x9, x1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: x12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	x12, x10		// wrap if bucket == buckets
	b.eq	3f
	ldp	x17, x9, [x12, #-16]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// double wrap
	JumpMiss $0
	
.endmacro

```

#### 2.2.2、汇编代码分析

下图表示`person`类对象的内存所占的内存大小图


![15454726370834.jpg](https://upload-images.jianshu.io/upload_images/3279997-9b5b9c51ee5e5504.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800/h/300)


##### 2.2.2.1、`ldp	x10, x11, [x16, #16`

`_objc_msgSend` 调用`CacheLookup`传的是`NORMAL`，入参是放在`x1 = SEL, x16 = isa`中的，最先执行的是`ldp	x10, x11, [x16, #16]`，这行命令的意思是从`x16`往下`16`个字节长度开始，连续读取`8` + `8`个字节的长度分别赋值给`x10`和`x11`。

1. 由于`x16 == isa` ，那么 `x16 + 16` 就是 `_buckets`的地址，赋值给`x10`；
2. 由于 `x10`所占的字节长度是`8`，那么`x11`就是接下来的`8`个字节，也就是读取到了`_mask`和`_occupied`，所以`x11 == _mask + _occupied`,由于是小端存储模式，`_mask`被存放在低`16`位，可以通过`w11`取到。

所以执行完`ldp	x10, x11, [x16, #16]`代码，`x10`和`x11`内容如下


![15454735233773.jpg](https://upload-images.jianshu.io/upload_images/3279997-10bdb79aef6b3a2e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)




##### 2.2.2.2、查找缓存列表

`2.2.2.1`已经找到了缓存`_buckets`的地址，下面就开始查找缓存了,`_buckets`的数据结构以及每个元素的地址可以通过下面的方式计算得出如下


![15454822875161.jpg](https://upload-images.jianshu.io/upload_images/3279997-c8f0644d65a76960.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)



```arm
and	w12, w1, w11		// x12 = _cmd & mask
add	x12, x10, x12, LSL #(1+3) // x12 = buckets + ((_cmd & mask) << (1+3))
ldp	x17, x9, [x12]		// {imp, sel} = *bucket

```

既然已经找到了存放缓存的地址，接下来只要找到调用的方法在缓存中的索引值就可以了，我们需要找的方法是`test1`，他的`SEL`为`4294971226`。

1. `and	w12, w1, w11` 初步计算出方法的索引值，`4294971226 & 3 == 2` ；
2. `add	x12, x10, x12, LSL #(1+3)`，步骤1计算出了`&`的结果，接下来就把指针移动到，序号为`2`的地方，左移`4`位正好是`16`个字节大小，要想跳到哪个序号直接序号左移`4`位即可，再加上初始的地址`x10`，就是序号的地址。
3. `ldp	x17, x9, [x12]	`，分别取出序号`2`的`_imp`和`_key`赋值给寄存器`x17`和`x9`。


知道每个元素的地址计算方式了，那么就是挨个判断是否是我们要找的方法了，我们初始值是序号`2`，接下来进入判断逻辑,流程图如下。

![untitled.png](https://upload-images.jianshu.io/upload_images/3279997-80f3a8556997601f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



由于我们是第一次调用是没有缓存的，所以即将进入方法`__objc_msgSend_uncached`，不幸的是这个方法还是汇编代码。

## 三、方法列表查找 & 动态方法解析

如果缓存中没有我们要找的方法，那么就会进入`__objc_msgSend_uncached`，汇编代码如下


```mm
  STATIC_ENTRY __objc_msgSend_uncached
	
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup
	TailCallFunctionPointer x17

	END_ENTRY __objc_msgSend_uncached
	
```

实际上这个就是调用的宏`MethodTableLookup`如果宏没有跳转就会调用`TailCallFunctionPointer x17`了，我们先看`MethodTableLookup`


```arm
.macro MethodTableLookup
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	// save parameter registers: x0..x8, q0..q7
	sub	sp, sp, #(10*8 + 8*16)
	stp	q0, q1, [sp, #(0*16)]
	stp	q2, q3, [sp, #(2*16)]
	stp	q4, q5, [sp, #(4*16)]
	stp	q6, q7, [sp, #(6*16)]
	stp	x0, x1, [sp, #(8*16+0*8)]
	stp	x2, x3, [sp, #(8*16+2*8)]
	stp	x4, x5, [sp, #(8*16+4*8)]
	stp	x6, x7, [sp, #(8*16+6*8)]
	str	x8,     [sp, #(8*16+8*8)]

	// receiver and selector already in x0 and x1
	mov	x2, x16
	bl	__class_lookupMethodAndLoadCache3

	// IMP in x0
	mov	x17, x0
	
	// restore registers and return
	ldp	q0, q1, [sp, #(0*16)]
	ldp	q2, q3, [sp, #(2*16)]
	ldp	q4, q5, [sp, #(4*16)]
	ldp	q6, q7, [sp, #(6*16)]
	ldp	x0, x1, [sp, #(8*16+0*8)]
	ldp	x2, x3, [sp, #(8*16+2*8)]
	ldp	x4, x5, [sp, #(8*16+4*8)]
	ldp	x6, x7, [sp, #(8*16+6*8)]
	ldr	x8,     [sp, #(8*16+8*8)]

	mov	sp, fp
	ldp	fp, lr, [sp], #16
	AuthenticateLR

.endmacro
```

如果懂函数栈帧的话，其实这个段汇编代码很简单，头和尾其实是常规操作，前面是开辟栈空间，保护寄存器的值，尾部恢复寄存器的值，以及恢复栈平衡，关键代码其实是`bl __class_lookupMethodAndLoadCache3`,这个`__class_lookupMethodAndLoadCache3`方法是一个我们熟悉的高级语言方法了，其实就是`_class_lookupMethodAndLoadCache3`

在文件`objc-runtime-new.m`中

```mm
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}


```


### `lookUpImpOrForward`的代码如下（有删减）

```mm
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();
    
 retry:    

    // 1.查找缓存 
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // 2.查找自己的方法列表并存入缓存
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }

    // 3.递归查找父类的缓存或者方法列表
    {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            
           // 3.1 查找父类的缓存
            imp = cache_getImp(curClass, sel);
             // 如果父类缓存中有 ！！！存入的是自己的缓存列表，并不是存到父类的缓存列表！！！
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                // 不在缓存的话存入自己的缓存
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
            
            // 3.2 父类缓存中没有 ， 就查找父类的方法列表
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
             // 如果父类方法列表中有 ！！！存入的是自己的缓存列表，并不是存到父类的缓存列表！！！
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }

    // 以上情况都没有 就会进入动态方法解析

    if (resolver  &&  !triedResolver) {
    // 也就是我们熟悉的 【+resolveClassMethod or +resolveInstanceMethod.】
        _class_resolveMethod(cls, sel, inst);
        triedResolver = YES;
        goto retry;
    }

    // 以上还没有找到 就会进入消息转发阶段
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlock();

    return imp;
}
```

注释都写在后面了，流程图如下：

![lookUpImpOrForward.png](https://upload-images.jianshu.io/upload_images/3279997-75e5fdb34601d905.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 四、消息转发（_objc_msgForward_impcache）

这个方法是汇编实现的，`objc-msg-arm64.s`中的汇编代码如下


```mm
STATIC_ENTRY __objc_msgForward_impcache

	// No stret specialization.
	b	__objc_msgForward

	END_ENTRY __objc_msgForward_impcache


```

其实它就是对`__objc_msgForward`的一个封装而已

```mm
	
	ENTRY __objc_msgForward

	adrp	x17, __objc_forward_handler@PAGE
	ldr	p17, [x17, __objc_forward_handler@PAGEOFF]
	TailCallFunctionPointer x17
	
	END_ENTRY __objc_msgForward
```

`__objc_forward_handler`是`runtime`的一个默认实现，代码在`objc-runtime.m`


```mm
__attribute__((noreturn)) void 
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;

```

需要借助一个`demo`来继续下面的探索,选择以下`Forwarding_demo`工程

![15456305553025.jpg](https://upload-images.jianshu.io/upload_images/3279997-9daf9ab804d00475.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15456308945541.jpg](https://upload-images.jianshu.io/upload_images/3279997-f1e42941e8076c30.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 4.1、查看调用堆栈-【X86架构】

运行以后会闪退，函数堆栈如下

```mm
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000107ca2705 libobjc.A.dylib`objc_exception_throw
    frame #1: 0x0000000108bfdf44 CoreFoundation`-[NSObject(NSObject) doesNotRecognizeSelector:] + 132
    frame #2: 0x0000000108be3ed6 CoreFoundation`___forwarding___ + 1446
    frame #3: 0x0000000108be5da8 CoreFoundation`__forwarding_prep_0___ + 120
  * frame #4: 0x000000010738572c Forwarding_demo`-[ViewController viewDidLoad]

【...】
```

从函数堆栈可以看出调用完`performSelector:`方法以后，代码就进入了`CoreFoundation`框架了，然后就到了`__forwarding_prep_0___` -> `___forwarding___`，`CoreFoundation`代码是开源的可以去[这里](https://opensource.apple.com/tarballs/CF/)下载,我下载的是`CF-1153.18 2`,遗憾的是虽然`CoreFoundation`代码是开源的，但是苹果没有给出以上方法的实现。

#### 4.1.1、断点查看方法实现

运行程序，分别增加2个断点


```mm
(lldb) breakpoint set -n '__forwarding_prep_0___'
Breakpoint 3: where = CoreFoundation`__forwarding_prep_0___, address = 0x00000001079efd30
(lldb) breakpoint set -n '___forwarding___'
Breakpoint 4: where = CoreFoundation`___forwarding___, address = 0x00000001079ed930
(lldb) 

```

运行程序，程序首先进入`__forwarding_prep_0___`


```mm
CoreFoundation`__forwarding_prep_0___:
->  0x1079efd30 <+0>:   pushq  %rbp
    [...]
    0x1079efda3 <+115>: callq  0x1079ed930               ; ___forwarding___
    [...]
    
```

过掉这个断点程序会进入`___forwarding___`


```mm
CoreFoundation`___forwarding___:
->  0x1079ed930 <+0>:    pushq  %rbp
    0x1079ed931 <+1>:    movq   %rsp, %rbp
    0x1079ed934 <+4>:    pushq  %r15
    0x1079ed936 <+6>:    pushq  %r14
    0x1079ed938 <+8>:    pushq  %r13
    0x1079ed93a <+10>:   pushq  %r12
    0x1079ed93c <+12>:   pushq  %rbx
    0x1079ed93d <+13>:   subq   $0x28, %rsp
    0x1079ed941 <+17>:   movq   0x26c798(%rip), %rax      ; (void *)0x000000010907b070: __stack_chk_guard
    0x1079ed948 <+24>:   movq   (%rax), %rax
    0x1079ed94b <+27>:   movq   %rax, -0x30(%rbp)
[...]
```

代码很长，`4.1.3`会专门分析这个方法

#### 4.1.2、逆向CoreFoundation.framework查看方法实现

除了直接添加断点的方式，还可以通过逆向`CoreFoundation.framework`来窥探实现，这样更直观，还能知道函数调用关系，`CoreFoundation.framework`放在`/System/Library/Frameworks/CoreFoundation.framework`

![15456324177613.jpg](https://upload-images.jianshu.io/upload_images/3279997-7e7403e3e0e24a43.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


找到了`___forwarding___`实现，并且知道是谁调用的，分别是框出来的部分`__forwarding_prep_0___`和`__forwarding_prep_1___`，点击进入`__forwarding_prep_0___`，同样的方式最后定位到了`___CFInitialize`

![15456325212259.jpg](https://upload-images.jianshu.io/upload_images/3279997-7baef2a17a03749d.jpg)

通过汇编确实看出`___CFInitialize`调用了`___forwarding___`方法，但是开源的代码根本没有这个相关的影子。


#### 4.1.3、分析__forwarding__实现

我们已经逆向出来了`__forwarding__`的汇编代码，但是可读性还是太差了，网上[有人](http://www.arigrant.com/blog/2013/12/13/a-selector-left-unhandled)已经将汇编代码转成熟悉的样子，这个方法就是消息转发的全部了。


```mm
void __forwarding__(BOOL isStret, void *frameStackPointer, ...) {
  id receiver = *(id *)frameStackPointer;
  SEL sel = *(SEL *)(frameStackPointer + 4);

  Class receiverClass = object_getClass(receiver);

  if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
    id forwardingTarget = [receiver forwardingTargetForSelector:sel];
    if (forwardingTarget) {
      return objc_msgSend(forwardingTarget, sel, ...);
    }
  }

  const char *className = class_getName(object_getClass(receiver));
  const char *zombiePrefix = "_NSZombie_";
  size_t prefixLen = strlen(zombiePrefix);
  if (strncmp(className, zombiePrefix, prefixLen) == 0) {
    CFLog(kCFLogLevelError,
          @"-[%s %s]: message sent to deallocated instance %p",
          className + prefixLen,
          sel_getName(sel),
          receiver);
    <breakpoint-interrupt>
  }

  if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
    NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
    if (methodSignature) {
      BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
      if (signatureIsStret != isStret) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
              sel_getName(sel),
              signatureIsStret ? "" : not,
              isStret ? "" : not);
      }
      if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {
        NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature
                                                                          frame:frameStackPointer];
        [receiver forwardInvocation:invocation];

        void *returnValue = NULL;
        [invocation getReturnValue:&value];
        return returnValue;
      } else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
              receiver,
              className);
        return 0;
      }
    }
  }

  const char *selName = sel_getName(sel);
  SEL *registeredSel = sel_getUid(selName);

  if (sel != registeredSel) {
    CFLog(kCFLogLevelWarning ,
          @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
          sel,
          selName,
          registeredSel);
  } else if (class_respondsToSelector(receiverClass, @selector(doesNotRecognizeSelector:))) {
    [receiver doesNotRecognizeSelector:sel];
  } else {
    CFLog(kCFLogLevelWarning ,
          @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
          receiver,
          className);
  }

  // The point of no return.
  kill(getpid(), 9);
}

```


涉及到的顺序：

1. `forwardingTargetForSelector:`；
2. `methodSignatureForSelector:`；
3. `forwardInvocation:`;
4. `doesNotRecognizeSelector:`。


## 五、objc_msgSend完整流程


![msg_send.png](https://upload-images.jianshu.io/upload_images/3279997-46e84b0dd25ec536.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
























