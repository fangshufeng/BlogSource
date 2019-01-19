---
title: 你不知道的TaggedPointer
date: 2019-01-13 08:24:22
tags: [Objective-C,objc_debug_taggedpointer_obfuscator]
key: "5"
---

## 一、环境介绍

1. `mac版本`：`Mac Mojave 10.14`
2. `objc版本`：`objc runtime 750`

## 二、为什么要使用TaggedPointer？

以前我们初始化一个对象（64位为例），开发的代码如下


```objc
NSNumber *number2 = [NSNumber numberWithInteger:2];

```

此时的内存图如下

![15469379882496.jpg](https://upload-images.jianshu.io/upload_images/3279997-d8298957b49c3f18.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到我就想存一个`2`用掉了`24`个字节，由于我们的`NSNumber`和`NSDate`对象的值一般不需要`8`个字节，`4`个字节的长度`2^31=2147483648`可以表达的数量已经达到了`20`多亿了，为了不造成内存的浪费，想到将指针的值(`8`个字节)进行拆分，一部分表示数据，一部分用来表示是一个特殊的指针，他不执行任何对象，这就是`TaggedPointer`技术，这样`指针` = `Data` + `Tag`，那么我们的存一个数字只需要`8`个字节就够了。

<!-- more -->

## 三、一个简单的例子 

### 3.1 版本新特性

```mm
NSNumber *number1 = @1;
NSNumber *number2 = @2;
NSNumber *number3 = @3;
NSNumber *numberFFFF = @(0xFFFF);

NSLog(@"number1 pointer is %p", number1);
NSLog(@"number2 pointer is %p", number2);
NSLog(@"number3 pointer is %p", number3);
NSLog(@"numberffff pointer is %p", numberFFFF);

```

输出结果却是这个样子的

```mm
number1 pointer is 0x19ec25e574ba1459
number2 pointer is 0x19ec25e574ba1759
number3 pointer is 0x19ec25e574ba1659
numberffff pointer is 0x19ec25e57445ea59

```

这个地址有点特殊，研究了一下，发现原来是在`10_14`以后苹果对`TaggedPointer`进行了混淆，文件`objc-runtime-new.m`写到


```c++
static void
initializeTaggedPointerObfuscator(void)
{
    if (sdkIsOlderThan(10_14, 12_0, 12_0, 5_0, 3_0) ||
        DisableTaggedPointerObfuscation) {
        objc_debug_taggedpointer_obfuscator = 0;
    } else {
        arc4random_buf(&objc_debug_taggedpointer_obfuscator,
                       sizeof(objc_debug_taggedpointer_obfuscator));
        objc_debug_taggedpointer_obfuscator &= ~_OBJC_TAG_MASK;
    }
}
```

混淆的代码也很简单，类似这种加入加密前的数据是`a`,加密后的数据为`b`，
那么：

`加密 `：`b` = `a` ^ `objc_debug_taggedpointer_obfuscator`,
`解密`:  `a` = `b` ^ `objc_debug_taggedpointer_obfuscator`.

这里利用了异或的特性，源码如下：

```mm
static inline void * _Nonnull
_objc_encodeTaggedPointer(uintptr_t ptr)
{
    return (void *)(objc_debug_taggedpointer_obfuscator ^ ptr);
}

static inline uintptr_t
_objc_decodeTaggedPointer(const void * _Nullable ptr)
{
    return (uintptr_t)ptr ^ objc_debug_taggedpointer_obfuscator;
}

```

所以要想知道`0x19ec25e574ba1459`是什么意思，还是要知道`objc_debug_taggedpointer_obfuscator`值，这是个随机值，要想获取这个值：

**方法一**：通过断点来获取

![15471013678173.jpg](https://upload-images.jianshu.io/upload_images/3279997-90a321a6b8785da4.jpg)



通过lldb指令读取


```mm
(lldb) p/x objc_debug_taggedpointer_obfuscator
(uintptr_t) $0 = 0x19ec25e574ba157e
```


**方法二**： 看来`runtime`源码知道`objc_debug_taggedpointer_obfuscator`是个全局变量，只要在我们用的地方申明一下即可


```
extern uintptr_t objc_debug_taggedpointer_obfuscator;

```

通过`NSLog`打印就可以了

```
NSLog(@"%lx",objc_debug_taggedpointer_obfuscator);
```

为了方便查看，简单写了一个方法，用来解开混淆


```mm
uintptr_t _objc_decodeTaggedPointer_(id  ptr) {
    NSString *p = [NSString stringWithFormat:@"%ld",ptr];
    return [p longLongValue] ^ objc_debug_taggedpointer_obfuscator;
}

```

### 3.2 真实的地址


```mm
NSNumber *number1 = @1;
NSNumber *number2 = @2;
NSNumber *number3 = @3;
NSNumber *numberFFFF = @(0xFFFF);

NSLog(@"number1 pointer is %p---真实地址:==0x%lx", number1,_objc_decodeTaggedPointer_(number1));
NSLog(@"number2 pointer is %p---真实地址:==0x%lx", number2,_objc_decodeTaggedPointer_(number2));
NSLog(@"number3 pointer is %p---真实地址:==0x%lx", number3,_objc_decodeTaggedPointer_(number3));
NSLog(@"numberffff pointer is %p---真实地址:==0x%lx", numberFFFF,_objc_decodeTaggedPointer_(numberFFFF));

```

输出


```mm
number1 pointer is 0xfda27e12be89be71---真实地址:==0x127
number2 pointer is 0xfda27e12be89bd71---真实地址:==0x227
number3 pointer is 0xfda27e12be89bc71---真实地址:==0x327
numberffff pointer is 0xfda27e12be764071---真实地址:==0xffff27

```

会发现，不管运行多少次，都是以`27`结尾，我们有理由相信，苹果贡献了`1`个字节（8个bit）来标识这是个特殊的指针，最后1个字节用来标识，这个类指针，判断是否是`TaggedPointer`不同平台判断的方式不一样，但对我们理解根本不影响


```mm
static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}

```

1. `mac`平台最后一个为`1`；
2. `iPhone`和模拟器，为最高位是`1`。


那么剩下的7个字节是不是都用来存放数据呢？

### 3.3 TaggedPointer存储的数字的最大值


```mm
NSNumber *numberF13   = @(0xFFFFFFFFFFFFF);
NSNumber *numberF13_1 = @(0x1FFFFFFFFFFFFF);
NSNumber *numberF13_3 = @(0x3FFFFFFFFFFFFF);
NSNumber *numberF13_7 = @(0x7FFFFFFFFFFFFF);
NSNumber *numberF14   = @(0xFFFFFFFFFFFFFF);

NSLog(@"numberF13 pointer is %p---真实地址:==0x%lx", numberF13,_objc_decodeTaggedPointer_(numberF13));
NSLog(@"numberF13_1 pointer is %p---真实地址:==0x%lx", numberF13_1,_objc_decodeTaggedPointer_(numberF13_1));
NSLog(@"numberF13_3 pointer is %p---真实地址:==0x%lx", numberF13_3,_objc_decodeTaggedPointer_(numberF13_3));
NSLog(@"numberF13_7 pointer is %p---真实地址:==0x%lx", numberF13_7,_objc_decodeTaggedPointer_(numberF13_7));
   
NSLog(@"numberF14 pointer is %p---真实地址:==0x%lx", numberF14,_objc_decodeTaggedPointer_(numberF14));

```

输出如下

```mm
number1 pointer is 0x20f9850034a2e631---真实地址:==0x127
number2 pointer is 0x20f9850034a2e531---真实地址:==0x227
number3 pointer is 0x20f9850034a2e431---真实地址:==0x327
numberffff pointer is 0x20f98500345d1831---真实地址:==0xffff27
numberF13 pointer is 0x2f067affcb5d1821---真实地址:==0xfffffffffffff37
numberF13_1 pointer is 0x3f067affcb5d1821---真实地址:==0x1fffffffffffff37
numberF13_3 pointer is 0x1f067affcb5d1821---真实地址:==0x3fffffffffffff37
numberF13_7 pointer is 0x5f067affcb5d1821---真实地址:==0x7fffffffffffff37

numberF14 pointer is 0x102500210

```

从输出可以看出，到`numberF14`地址已经是真正的`oc`对象的地址了，说明有效存储位置有`56`位，所以`TaggedPointer`所能表达的数字范围为`[0 2^65)`。

## 四、思考：你会如何实现`NSString`的TaggedPointer？

我们现在想做的事情就是如何利用指针来存储我们的字符数据，而指针的大小就是`8`个字节，一共`64`位，如何利用这个`64`位呢？由`NSNumber`的灵感，可以使用低`1`位来表示是`TaggedPointer`类型，其他三位来表示具体哪个类的，对于字符串，需要存储它的长度，再让出`4`位，还剩下`56`位，从而问题转为如何利用这个`56`位。


计算机中存储的就是`0`和`1`，对于字符串的编码有`ASCII`和`非ASCII`：

1. `ASCII`是利用一个字节的大小表示字符的，一共是`128`个（最高位都为0）；
2. 后面为了统一编码出现了`Unicode`编码，`Unicode`是规定了符号的二进制代码，没有规定如何存储，具体如何存储的，后来就出现了，`UTF-16`（字符用两个字节或四个字节表示）、`UTF-32`（字符用四个字节表示）和`UTF-8`（最常用的，兼容了`ASCII`）


**对于非`ASCII`：**

1. 如果是`UTF-32`编码的,要想包含所有`Unicode`，需要`4`个字节，那么最多也只能保存1个字符，没有任何意义；
2. 如果是 `UTF-16`编码的，要想包含所有`Unicode`，也需要`4`个字节，最少也需要`2`个字节，按最少的算，那么`56`位，也只能放`3`个`16`为的字符，还是很少；
3. 如果是`UTF-8`，如果撇开`ASCII`的话，那么也是最多需要`4`个字节，最少`2`个字节，`56`位还是最多放`3`个字节。

对于非`ASCII`我们貌似没有找到一个好的方案来存储，那么我们要实现`TaggedPointer`的话，是不是可以不考虑`非ASCII`的情况，毕竟在实际场景，我们用到`ASCII`的场景的几率还是比`非ASCII`大的多，对于`非ASCII`的还是交给开辟控件的方式。


**对于`ASCII`:**

如果我们不考虑非`ASCII`的话，那么有以下方案可以用来存储数据：

1. 方案一: 使用`8`位存储一个字符，这也是默认计算机存储`ASCII`的方式，由于占用一个字节，那么这种方式`56`位可以放`7`个字节； 
2. 方案二: 使用`7`位存储一个字符，`ASCII`其实真正存储数据的是`7`位，如果是用`7`位表示一个字符的话，那么最多可以放`8`个字节，比方案一多出一个字节；
3. 方案三: 使用`6`位存储，有人可能想`6`位怎么可能，存储`ASCII`最少也得`7`位啊，`6`怎么存储，是的，直接存是不行的，但是我们可以不直接存字符，而是提供一个表格，存索引。`ASCII`一共有`128`个，但是我们常用的根本就没有那么多，那么我们可以不可以选出一些常用的来作为我们的可选值 ？ `6`位的话，最多可以存储`2^ 6 = 64`个不同的字符,所以肯定是不能满查找`ASCII`集合，但是，我们可以找来常见的`64`个字符比如`[a-zA-z0-9./_-]`,这里就有`66`个了，再从这个`66`个里面取出2个不常用的就可以了，这样的话我们就可以存储`9`个字节了；
4. 方案四: 使用`5`位存储，这种的话我们的查找范围就缩小为了`2^5 = 32`个，也就是我们要在方案三的基础上在找出更加常用的`32`个字符，这种方案可以存储`11`个字符；
5. 方案五: 使用`4`位存储，那范围就是`2^4 = 16`个，这种感觉行也行，但是范围太小了
6. 更少的想想不大可能了


下面看下苹果是如何实现的

## 五、对于NSString苹果是如何使用TaggedPointer的？

### 5.1 现象

添加测试如下测试代码

```mm
NSMutableString *imutable = [NSMutableString string];
NSString *immutable;
char c = 'a';
do {
 [imutable appendFormat: @"%c", c++];
 immutable = [imutable copy];
 NSLog(@"源地址：%p 真实地址：0x%lx %@ %@", immutable,_objc_decodeTaggedPointer_(immutable), immutable, object_getClass(immutable));
} while(((uintptr_t)immutable & 1) == 1);

```

输出，这里我省去了源地址，因为这里打印了类的类型更直观写

```mm
真实地址：0x6115                 a               NSTaggedPointerString
真实地址：0x626125               ab              NSTaggedPointerString
真实地址：0x63626135             abc             NSTaggedPointerString
真实地址：0x6463626145           abcd            NSTaggedPointerString
真实地址：0x656463626155         abcde           NSTaggedPointerString
真实地址：0x66656463626165       abcdef          NSTaggedPointerString
真实地址：0x6766656463626175     abcdefg         NSTaggedPointerString
真实地址：0x22038a01169585       abcdefgh        NSTaggedPointerString
真实地址：0x880e28045a54195      abcdefghi       NSTaggedPointerString
真实地址：0xf9eb5f3ca3c376e0     abcdefghij      __NSCFString

```

前面提到过最后一个字节低`4`位标志是`TaggedPointer`信息，高`4`位存放字符串的长度，所以最后一个数字`5`是标志位，倒数一个数字就是字符串的长度。

从上面的输出可以看出：

1. 当字符串的长度`<=7`的时候，苹果是直接存储的字符`ASCII`值，`a`的`ASCII`值是`61`，`b`是`62`...。
2. 当字符串长度大于`7`的时候具体如何做的，我们通过逆向`CoreFoundation.framework`来查看

### 5.2 hopper -> length

先来看下`length`方法，看看是不是和我们猜测的一样

![15471101516030.jpg](https://upload-images.jianshu.io/upload_images/3279997-8a7ea6bb33614cda.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

翻译一下就是

```mm

rdi = self ^ *_objc_debug_taggedpointer_obfuscator; // 解密得到真实地址

if ((di & 14 ) == 14) { 也就是//0b1110 我们的字符串的是5(0x0101)，所以走else了
       rax = (di >> 11) & 0xf;
} else {
       rax =(di >>  4 ) & 0xf;
}  


再简化一下就是

======

rax = (di >>  4 ) & 0xf

```

已经很显然了，就是拿低1字节的高4位的值,证明了我们的猜想。

### 5.3 hopper -> characterAtIndex

苹果是如何将字符转成`NSTaggedPointerString`的，不是很好查，但是我们可以反向思考，通过取数据来反推如何存的，

![15471113839214.jpg](https://upload-images.jianshu.io/upload_images/3279997-b8e698d25e27748f.jpg)


下面开始简化该伪代码，如果你觉得不想看，可以直接跳到`第四次简化`开始看。


`___stack_chk_guard`是为了安全加的，不考虑，前面分析过`((((r8 ^ rdi) & 0xe) == 0xe ? 0x1 : 0x0) << 0x3 | 0x4)`在这里等价于`0x4`，`arg2`就是传进来的`index`


#### 5.3.1 第一次简化


```mm
unsigned short -[NSTaggedPointerString characterAtIndex:](void * self, void * _cmd, unsigned long long arg2) {
    r12 = index;
    rbx = self >>  0x4 & 0xf;
    r8 = self >> 0x4 >> 0x4;
    if (rbx >= 0x8) {
            rdx = rbx;
            if (rbx < 0xa) {
                    do {
                            *(int8_t *)(rbp + rdx + 0xffffffffffffffc7) = *(int8_t *)((r8 & 0x3f) + _sixBitToCharLookup);
                            rdx = rdx - 0x1;
                            r8 = r8 >> 0x6;
                    } while (rdx != 0x0);
            }
            else {
                    do {
                            *(int8_t *)(rbp + rdx + 0xffffffffffffffc7) = *(int8_t *)((r8 & 0x1f) + _sixBitToCharLookup);
                            rdx = rdx - 0x1;
                            r8 = r8 >> 0x5;
                    } while (rdx != 0x0);
            }
    }

    rax = *(int8_t *)(rbp + r12 + 0xffffffffffffffc8) & 0xff;
    
    return rax;
}

```

继续分析这段代码

1. `self >>  0x4 & 0xf;`其实就是字符串的`length`
2. `self >> 0x4 >> 0x4;`其实就是字符串的开始位置
3. `0xffffffffffffffc7`其实是`-0x39 = -57`的补码,`0xffffffffffffffc7`是`-0x38 = -56`的补码


#### 5.3.2 第二次简化

```mm
unsigned short -[NSTaggedPointerString characterAtIndex:](void * self, void * _cmd, unsigned long long arg2) {
    rbx = length;
    r8 = self >> 0x8;
    if (rbx >= 0x8) {
            if (length < 0xa) {
                    do {
                            *(int8_t *)(rbp - 57 + rdx) = *(int8_t *)((r8 & 0x3f) + _sixBitToCharLookup);
                            rdx = rdx - 0x1;
                            r8 = r8 >> 0x6;
                    } while (rdx != 0x0);
            }
            else {
                    do {
                            *(int8_t *)(rbp - 57 + rdx) = *(int8_t *)((r8 & 0x1f) + _sixBitToCharLookup);
                            rdx = rdx - 0x1;
                            r8 = r8 >> 0x5;
                    } while (rdx != 0x0);
            }
    }

    rax = *(int8_t *)(rbp - 56 + index) & 0xff;
    
    return rax;
}


```


1. `bp`其实就是栈指针，这里使用`bp`说明是通过`bp`来操控栈空间的，然后每次循环`dx`都减1，然后`r8`左移`6`位或者`5`位，这个一般都是数组操作了，如果是`5`位的话最多存`11`个字节，所以这里使用一个长度`11`的数组`buffer[11]`，`dx`其实就会游离指针了我们用变量`cursor`表示



#### 5.3.3 第三次简化


```mm
unsigned short -[NSTaggedPointerString characterAtIndex:](void * self, void * _cmd, unsigned long long arg2) {

    int8_t  buffer[11];
    r8 = self >> 0x8;
    
    if (length >= 0x8) {
            base = rbp - 57;
            cursor = length;
            if (length < 0xa) {
                    do {
                            buffer[base + cursor ] = *(int8_t *)((r8 & 0x3f) + _sixBitToCharLookup)
                            cursor = cursor - 0x1;
                            r8 = r8 >> 0x6;
                    } while (rdx != 0x0);
            }
            else {
                    do {
                            buffer[base + cursor ] = *(int8_t *)((r8 & 0x1f) + _sixBitToCharLookup);
                            cursor = cursor - 0x1;
                            r8 = r8 >> 0x5;
                    } while (rdx != 0x0);
            }
    }

    rax = *(int8_t *)(rbp - 56 + index) & 0xff;
    
    return rax;
}


```


`_sixBitToCharLookup`到底是什么呢，其实就是字符串

![15471761831012.jpg](https://upload-images.jianshu.io/upload_images/3279997-672dfc0b86516c8b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


也就是`eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX`


其实程序还少了一段代码，`hopper`翻译伪代码的时候漏掉了

```mm
0000000000060d87         cmp        rbx, 0x8
0000000000060d8b         jb         loc_60dd1 // 当bs < 0x8时
...
loc_60dd1:
0000000000060dd1         mov        qword [rbp+var_38], r8   
```

`var_38`就是`-56`

![15471166175011.jpg](https://upload-images.jianshu.io/upload_images/3279997-6fe714148285a6df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实就是将`r8`的值放到`[bp-56]`的内存处，由于是小端存储，其实就是讲`self>> 8`的内容存放到对应的内存地址，类似于下面的代码，但是是占`8`个字节的


```
 *(uint64_t *)buffer = self >> 8;
```


#### 5.3.4 第四次简化

```mm
unsigned short -[NSTaggedPointerString characterAtIndex:](void * self, void * _cmd, unsigned long long arg2) {

    int8_t  buffer[11];
    r8 = self >> 0x8;
    
    if (length >= 0x8) {
       base = rbp - 57;
       cursor = length;
       _sixBitToCharLookup = 'eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX';
       if (length < 0xa) {
               do {
                   buffer[base + cursor ] = _sixBitToCharLookup[r8 & 0x3f]
                   cursor = cursor - 0x1;
                   r8 = r8 >> 0x6;
               } while (rdx != 0x0);
       } else {
               do {
                   buffer[base + cursor ] = _sixBitToCharLookup[r8 & 0x1f];
                   cursor = cursor - 0x1;
                   r8 = r8 >> 0x5;
               } while (rdx != 0x0);
       }
    } else {
        *(uint64_t *)buffer = self >> 8;
    }

    rax = *(int8_t *)(rbp - 56 + index) & 0xff;
    
    return rax;
}


```

这就显而易见了，对于字符串苹果的处理如下：

1. 对于小于`8`个字符的，使用的是`8`位存储；
2. `[8,10)`的是通过`6`位存储的；
3. `[10,11]`的是通过`5`位存储的。

根据这个结论我们再来看下`5.1`的现象，对于上面的判断条件分别选一个代表

##### 5.3.4.1 小于`8`位代表`0x66656463626165 -> abcdef`

可以看出是直接存储的；

##### 5.3.4.2  `[8,10)`代表：`0x22038a01169585 -> abcdefgh`

去掉后面的`95`剩下`0x22038a011695`，`6`位排列如下


`001000 100000 001110 001010 000000 010001 011010 010101`，每一个就对应这个字符串`eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX`的索引值，为了方便查找做了一个对照表

![15469350941106.jpg](https://upload-images.jianshu.io/upload_images/3279997-9ef6a5e7ac5dc8af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以

`001000 100000 001110 001010 000000 010001 011010 010101`

分别对应

` a     b       c       d       e      f    g       h`

##### 5.3.4.3  `[10,11]`位代表`abcdefghij`

但是这个类是`__NSCFString`并不是我们的`NSTaggedPointerString`，按道理说`5`位的话是可以存放`10`个字节的啊，这是什么原因呢？

原来：不管是`5`位还是`6`位都是查询的同一个字符串`eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX`，也就是上图索引表的颜色区分，`5`位里面没有包含`b`字符，但是我们的`abcdefghij`有`b`字符，所以不行，修改demo如下看看


```mm

NSString *str = [NSString stringWithFormat:@"acdefghijk"];
NSString *str2 = [NSString stringWithFormat:@"acdefghijkm"];
NSString *str3 = [NSString stringWithFormat:@"acdefghijkmn"];
    
NSLog(@"真实地址：0x%lx %@ %@", str,_objc_decodeTaggedPointer_(str), str3, object_getClass(str));
NSLog(@"真实地址：0x%lx %@ %@", str2,_objc_decodeTaggedPointer_(str2), str3, object_getClass(str2));
NSLog(@"真实地址：0x%lx %@ %@", str3,_objc_decodeTaggedPointer_(str3), str3, object_getClass(str3));

```

输出

```mm
真实地址：0x10e5023aa86d2a5 acdefghijk NSTaggedPointerString
真实地址：0x21ca047550da46b5 acdefghijkm NSTaggedPointerString
真实地址：0xc64838cff22b0b46 acdefghijkmn __NSCFString

```

可以看到能够支持`11`个字节了，`0x10e5023aa86d2a5`去掉`0x10e5023aa86d2`，按`5`位排列下看看

`01000 01110 01010 00000 10001 11010 10101 00001 10110 10010`

也就是 `a c d e f g h i j k`


所以我们可以得出能够存`[10,11]`位字符是以所存字符在`eilotrm.apdnsIc ufkMShjTRxgC4013`内为前提的。


最后再来看下苹果对于非`ASCII`是怎么处理的，以汉字`方`（Unicode）编码为`\u65b9`,占3个字节，按道理也是可以放进指针里面的，我们看看苹果有没有这样做


```mm
NSString *notAscii_1 = [NSString stringWithFormat:@"方"];

NSLog(@"源地址：%p  %@ %@", notAscii_1,notAscii_1, object_getClass(notAscii_1));
        
```

输出


```mm
源地址：0x101907df0  方 __NSCFString

```

发现苹果并没有放进指针内，而是真实的`oc`对象。

至此，我们之前的猜测一一验证了。

下面总结一下`TaggedPointer`的特点


## 六、什么样的字符会放进TaggedPointer？

总结了以下表格，注意这个只适用`ASCII`的情况，对于非`ASCII`都是使用的`oc`对象。

![15471895085356.jpg](https://upload-images.jianshu.io/upload_images/3279997-46e0ce57f662e409.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

传入的字符任意一个不在所在行的范围，存的地方就会发生变化。


## 七、一个和TaggedPointer相关的面试题

下面代码会发生什么问题？

```mm

@property (nonatomic, copy) NSString *target;
//.... dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);

// 方式一
for (int i = 0; i < 1000000 ; i++) {
    dispatch_async(queue, ^{
        self.target = [NSString stringWithFormat:@"ksddkjalkjd%d",I];
    });
}



//.... dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
// 方式二
for (int i = 0; i < 1000000 ; i++) {
    dispatch_async(queue, ^{
        self.target = [NSString stringWithFormat:@"ksddkjalkj"];
    });
}


```


先说下结果吧 ，方式一会闪退，方式二正常运行。

分析这个道题，`target`的`set`方法实现


```mm
- (void)setTarget:(NSString *)target {
    if(_target != target) {
        [_target release];
        target = [target retain];
    }
}
```

方式一是真正的`oc`对象，由于是多线程会出现`[_target release];`被调用多次，从而闪退；
方式二不是`oc`对象，而是`TaggedPointer`,在`release`和`retain`的时候都会判断是不是`TaggedPointer`


```c++

objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;

    bool sideTableLocked = false;
    ...
}


ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    if (isTaggedPointer()) return (id)this;

}

```

其他的方式可以加锁解决，就不说了。


## 感谢


1. [http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
2. [https://mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html](https://mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)




















