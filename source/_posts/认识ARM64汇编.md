---
title: 认识ARM64汇编
date: 2018-12-18 21:04:22
tags:
    汇编
---



之前说过学习汇编就是学习寄存器和指令，查看代码请连接真机。

## 寄存器


在`arm64`汇编中寄存器是`64`bit的，使用`X[n]`表示，低32位以`w[n]`表示

![15450350776264.jpg](https://upload-images.jianshu.io/upload_images/3279997-6168a0ffb12c0df9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在`64`位架构中有`31`个`64`位的通用寄存器。


可以通过`register read`查看

<!-- more -->

```mm
(lldb) register read
General Purpose Registers:
        x0 = 0x0000000000000001
        x1 = 0x00000002805d8a20
        x2 = 0x00000002837e7980
        x3 = 0x00000002805d8a20
        x4 = 0x0000000000000001
        x5 = 0x0000000000000001
        x6 = 0x0000000100d54000  
        x7 = 0x0000000000000000
        x8 = 0x0000200000000000
        x9 = 0x000161a1d63f7945 (0x00000001d63f7945) (void *)0x01d63f7cb0000001
       x10 = 0x0000000000000000
       x11 = 0x000000000000006d
       x12 = 0x000000000000006d
       x13 = 0x0000000000000000
       x14 = 0x0000000000000000
       x15 = 0x00000001ca634e6d  "touchesBegan:withEvent:"
       x16 = 0x000000019c4cd47c  libobjc.A.dylib`objc_storeStrong
       x17 = 0x0000000000000000
       x18 = 0x0000000000000000
       x19 = 0x0000000100f17810
       x20 = 0x00000002837e7920
       x21 = 0x00000002805d8a20
       x22 = 0x00000001ca634e6d  "touchesBegan:withEvent:"
       x23 = 0x0000000100e11a30
       x24 = 0x00000002837e7980
       x25 = 0x0000000100e11a30
       x26 = 0x00000002805d8a20
       x27 = 0x00000001ca62d900  "countByEnumeratingWithState:objects:count:"
       x28 = 0x00000002805d8a20
        fp = 0x000000016f47d730
        lr = 0x00000001009866dc  ArmAssembly`-[ViewController touchesBegan:withEvent:] + 84 at ViewController.m:38
        sp = 0x000000016f47d720
        pc = 0x0000000100986720  ArmAssembly`foo1 + 16 at ViewController.m:46
      cpsr = 0x80000000

(lldb) 

```

### `x0~x7`：一般是函数的参数，大于`8`个的会通过堆栈传参。


```mm
/*
 * ArmAssembly`foo2:
 0x101062760 <+0>:  sub    sp, sp, #0x20             
 0x101062764 <+4>:  str    w0, [sp, #0x1c]
 0x101062768 <+8>:  str    w1, [sp, #0x18]
 0x10106276c <+12>: str    w2, [sp, #0x14]
 0x101062770 <+16>: str    w3, [sp, #0x10]
 0x101062774 <+20>: str    w4, [sp, #0xc]
 0x101062778 <+24>: str    w5, [sp, #0x8]
 0x10106277c <+28>: str    w6, [sp, #0x4]
 ->  0x101062780 <+32>: add    sp, sp, #0x20             
 0x101062784 <+36>: ret

 */
void foo7(int a,int b,int c ,int d ,int e,int f,int g) {
    
}
/*
 *ArmAssembly`foo8:
 0x10024275c <+0>:  sub    sp, sp, #0x20             
 0x100242760 <+4>:  str    w0, [sp, #0x1c]
 0x100242764 <+8>:  str    w1, [sp, #0x18]
 0x100242768 <+12>: str    w2, [sp, #0x14]
 0x10024276c <+16>: str    w3, [sp, #0x10]
 0x100242770 <+20>: str    w4, [sp, #0xc]
 0x100242774 <+24>: str    w5, [sp, #0x8]
 0x100242778 <+28>: str    w6, [sp, #0x4]
 0x10024277c <+32>: str    w7, [sp]
 ->  0x100242780 <+36>: add    sp, sp, #0x20             
 0x100242784 <+40>: ret

 */
void foo8(int a,int b,int c ,int d ,int e,int f,int g,int h) {
    
}


/*
 * ArmAssembly`foo9:
 0x100dc2718 <+0>:  sub    sp, sp, #0x30  
           
 0x100dc271c <+4>:  ldr    w8, [sp, #0x30] ； 大于8个参数的时候 已经是从内存栈中去取数据了
 
 0x100dc2720 <+8>:  str    w0, [sp, #0x2c]
 0x100dc2724 <+12>: str    w1, [sp, #0x28]
 0x100dc2728 <+16>: str    w2, [sp, #0x24]
 0x100dc272c <+20>: str    w3, [sp, #0x20]
 0x100dc2730 <+24>: str    w4, [sp, #0x1c]
 0x100dc2734 <+28>: str    w5, [sp, #0x18]
 0x100dc2738 <+32>: str    w6, [sp, #0x14]
 0x100dc273c <+36>: str    w7, [sp, #0x10]
 0x100dc2740 <+40>: str    w8, [sp, #0xc]
 0x100dc2744 <+44>: add    sp, sp, #0x30             
 0x100dc2748 <+48>: ret

 */
void foo9(int a,int b,int c ,int d ,int e,int f,int g,int h,int i) {
    
}
```

可以看到当参数的个数大于8个的时候就不会从寄存器中去取参数了。

### `x0`：一般表示返回值


```mm
int fooReturnValue() {
    return  10;
}

```

汇编代码


```mm
ArmAssembly`fooReturnValue:
    0x100ece7b4 <+0>: mov    w0, #0xa
->  0x100ece7b8 <+4>: ret    

```

通过`lldb`指令


```mm
(lldb) register read x0
      x0 = 0x000000000000000a
(lldb) 

```

确实是`10`

### `pc`：表示当前执行的指令的地址

这个类似`8086`汇编里面的`ip`


![15450984590679.jpg](https://upload-images.jianshu.io/upload_images/3279997-e4b5894fe6ebb903.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


比如我们断点该改代码处，查看`pc`寄存器的值

```mm

(lldb) register read pc
      pc = 0x0000000100b6e7b8  ArmAssembly`fooReturnValue + 4 at ViewController.m:140
(lldb) 

```

### `lr`:链接寄存器，存放着函数的返回地址

`lr`也就是`x30`，这个里面存放着函数的返回地址

比如有下面2个方法，方法`fooFp`方法内部调用`fooFp2`

```mm
void fooFp() {
    int a = 4;
    int b = 5;
    fooFp2();
}

void fooFp2() {
    int a = 2;
    int b = 3;
}

```

`fooFp`的汇编代码


```mm
ArmAssembly`fooFp:
 0x100dbe76c <+0>:  sub    sp, sp, #0x20             ; =0x20
 0x100dbe770 <+4>:  stp    x29, x30, [sp, #0x10]
 0x100dbe774 <+8>:  add    x29, sp, #0x10            ; =0x10
 
 0x100dbe778 <+12>: mov    w8, #0x5
 0x100dbe77c <+16>: orr    w9, wzr, #0x4
 0x100dbe780 <+20>: stur   w9, [x29, #-0x4]
 0x100dbe784 <+24>: str    w8, [sp, #0x8]
 0x100dbe788 <+28>: bl     0x100dbe798               ; fooFp2 at ViewController.m:154
 
 0x100dbe78c <+32>: ldp    x29, x30, [sp, #0x10]
 0x100dbe790 <+36>: add    sp, sp, #0x20             ; =0x20
 0x100dbe794 <+40>: ret
 
```

`0x100dbe788 <+28>: bl     0x100dbe798 `这行就是在调用方法`fooFp2`，如果调用完`fooFp2`后函数应该要继续执行，那么肯定是要到` 0x100dbe788 <+28>: bl     0x100dbe798`下一行也就是地址`0x100dbe78c`处，我们到`fooFp2`中查看下`lr`寄存器的值


```mm
(lldb) register read lr
      lr = 0x0000000100dbe78c  ArmAssembly`fooFp + 32 at ViewController.m:170
(lldb) 

```

当然， 本质上还是将`lr`的值给了`pc`寄存器了，也就是`ret`指令做的事情。

### `栈寄存器`

分为`sp`栈顶寄存器和`fp`栈底寄存器。（如果熟悉8086汇编的话类似于分别类似于`sp`和`bp`）

还是用上面的方法`fooFp`和`fooFp2`来说明这2个寄存器。

不嫌啰嗦这边还是复制下`fooFp`的汇编代码


```mm

ArmAssembly`fooFp:
 0x100dbe76c <+0>:  sub    sp, sp, #0x20             ; 申请栈空间
 0x100dbe770 <+4>:  stp    x29, x30, [sp, #0x10]     ; 保护寄存器的值
 0x100dbe774 <+8>:  add    x29, sp, #0x10            ; 改变fp寄存器的值，用于执行栈底
 
 0x100dbe778 <+12>: mov    w8, #0x5
 0x100dbe77c <+16>: orr    w9, wzr, #0x4
 0x100dbe780 <+20>: stur   w9, [x29, #-0x4]
 0x100dbe784 <+24>: str    w8, [sp, #0x8]
 0x100dbe788 <+28>: bl     0x100dbe798               ; fooFp2 at ViewController.m:154
 
 0x100dbe78c <+32>: ldp    x29, x30, [sp, #0x10]     ; 恢复之前保存的fp和lr的值
 0x100dbe790 <+36>: add    sp, sp, #0x20             ; 恢复sp指针
 0x100dbe794 <+40>: ret 
 
```

整个过程如下图

![15451047620385.jpg](https://upload-images.jianshu.io/upload_images/3279997-42edd339144e4c4a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2)


### `cpsr`：程序状态寄存器


#### cpsr（Current Program Status Register ）
#### spsr (Saved Program Status Register) 异常状况下使用


### `xzr`:零寄存器

表示`zero register`，一般用来默认值，里面存储的值都是0。

1. `xzr`:64位的
2. `wzr`:32位的

## 指令

### 寻址方式大概规则说明


```mm
ADD R0，R0，＃1  // R0 = R0 + #1 表示寄存器R0的值 + 1再赋值给R0

ADD R0，R1，R2  // R0 = R1＋R2 

ADD R0，R1，[R2] // R0←R1＋[R2]

LDR R0，[R1，＃4]  // ；R0←[R1＋4]

LDR R0，[R1，＃4]！  // R0←[R1＋4]、R1←R1＋4

LDR R0，[R1] ，＃4  // ；R0←[R1]、R1←R1＋4

LDR R0，[R1，R2] // R0←[R1＋R2]

```

1. 有`[]`一般表示是取值的意思，`[R2]`表示取出`R2`所存的内存地址比如是`0x10000`所对应的值比如是`66`。
2. `LDR R0 [R1，＃4]` ：寄存器 `R1` 的内容加上`4`形成操作数的有效地址，从而取得操作数存入寄存器 `R0` 中。
3. `LDR R0，[R1，＃4]！`：将寄存器 `R1` 的内容加上 `4` 形成操作数的有效地址，从而取得操作数存入寄 存器 `R0` 中，然后，`R1` 的内容自增 `4` 个字节。
4. `LDR R0，[R1] ，＃4`:以寄存器 `R1` 的内容作为操作数的有效地址，从而取得操作数存入寄存器 `R0` 中，然后，`R1` 的内容自增 `4` 个字节。
5. `LDR R0，[R1，R2]`:将寄存器 `R1` 的内容加上寄存器 `R2` 的内容形成操作数的有效地址，从而取得 操作数存入寄存器 `R0` 中。

### 计算指令

#### `ADD` 加法 


```mm
ADD R0，R1，R2 // R0 = R1 + R2
ADD R0，R1，#256  // R0 = R1 + 256 

ADD R0，R2，R3，LSL#1 //  R0 = R2 + (R3 << 1)

```

#### `SUB` 减法


```mm
SUB R0，R1，R2 // R0 = R1 - R2
SUB R0，R1，#256 //R0 = R1 - 256 
SUB R0，R2，R3，LSL#1 // R0 = R2 - (R3 << 1)

```

#### 逻辑运算

`AND`逻辑与、`EOR`逻辑异或、`ORR`逻辑或、`LSL`逻辑左移、`LSR`逻辑右移

### 内存指令


一般是`ST`开头的为存数据,比如说`STR`、`STP`、`STUR`
`LD`开头的表示取数据，比如说`LDR`、`LDP`、`lDUR`


```mm
str    w8, [sp, #0x8] // 表示 将w8存放到sp+0x8表示的内存中
stur   w9, [x29, #-0x4] // 表示 将w9存放到x29, #-0x4表示的内存中
stp x1, x2, [sp, #-16] // 表示 从sp-0x16对应地址的开始存放 x1、x2的表示的值

ldp    x29, x30, [sp, #0x10] //表示 从sp+0x10对应地址的开始取出值赋值给x29,x30

```

`P`：可以理解为`pair`；
`u`: 表示负数

### 跳转指令

有`b`、`bl`一般搭配`cmp`指令使用

`b`：无条件跳转，一般是什么函数内部的`if`、`switch`条件判断；
`bl`:带函数返回值的跳转，一般是调用其他的函数；


```mm
0x100432624 <+88>:  cmp    w10, #0x1                 ; =0x1 
0x100432628 <+92>:  b.le   0x100432630               ; 
    
```

<1>. `CMP`:将寄存器 R1 的值与立即数 `0x1` 相减，并根据结果设置 CPSR 的标志位

标志位的可能值如下表

| 条件码 | 助记符后缀 | 标志 | 含义 |
| :-: | :-: | :-: | :-: |
| 0000 | EQ | Z 置位 | 相等 == |
| 0001 | NE | Z 清零 | 不相等 != |
| 0010 | CS | C 置位 | 无符号数大于或等于 |
| 0011 | CC | C 清零 | 无符号数小于 |
| 0100 | MI | N 置位 | 负数 |
| 0101 | PL | N 清零 | 正数或零 |
| 0110 | VS | V 置位 | 溢出 |
| 0111 | VC | V 清零 | 未溢出 |
| 1000 | HI | C 置位 Z 清零 | 无符号数大于 |
| 1001 | LS | C 清零 Z 置位 | 无符号数小于或等于 |
| 1010 | GE | N 等于 V | 带符号数大于或等于 |
| 1011 | LT | N 不等于 V | 带符号数小于 |
| 1100 | GT | Z 清零且（N 等于 V） | 带符号数大于 |
| 1101 | LE | Z 置位或（N 不等于 V） | 带符号数小于或等于 |
| 1110 | AL | 忽略 | 无条件执行 |


<2>. `b.le   0x100432630`：表示如果`w10` <= `0x1`那么就执行`0x100432630`



###  `ret`指令

1. 函数返回
2. 将`lr(x30)`寄存器器的值赋值给`pc`寄存器器。

了解了这些基本知识读一些汇编代码应该没问题了，没有提到的自己查下资料也差不多了。


## ARM-GNU汇编

如果你只会`arm`汇编去看`runtime`汇编源码的时候会发现还是有些东西不明白，比如下面的代码


```mm
#if SUPPORT_TAGGED_POINTERS
	.data
	.align 3
	.globl _objc_debug_taggedpointer_classes
_objc_debug_taggedpointer_classes:
	.fill 16, 8, 0
	.globl _objc_debug_taggedpointer_ext_classes
_objc_debug_taggedpointer_ext_classes:
	.fill 256, 8, 0
#endif

	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
	b.eq	LReturnZero
#endif
	ldr	p13, [x0]		// p13 = isa
	GetClassFromIsa_p16 p13		// p16 = class
LGetIsaDone:
	CacheLookup NORMAL		// calls imp or objc_msgSend_uncached

[...]
```

> ARM汇编开发指用ARM提供的汇编指令，进行ARM程序的开发。
> ARM汇编开发，有两种开发方式，一种是使用ARM汇编，一种是使用ARM GNU汇编。两种汇编开发，使用的汇编指令是完全一样的，区别是宏指令，伪指令，伪操作不一样。其实两种开发方式的区别在于所使用的编译工具不一样。
> 对于ARM汇编，使用的是ARM公司开发的编译器，而ARM GNU汇编，是使用GNU为ARM指令集开发的编译器，也就是arm-gcc。

2种方式的不同之处就是伪操作的不同，苹果遵循的是`GNU`汇编规范的。点击[这个](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/000-Introduction/introduction.html#//apple_ref/doc/uid/TP30000851-CH211-SW1)可以查看各个伪操作的意思，比如：

`.global`:全局声明；
`.macro`:定义一个宏；
`.align`:对齐方式
`.text`:指定程序在哪个段
...


关于汇编还有很多，比如书写汇编代码，内联汇编，有了目前的基础，相信学起来也是很快的。

感谢

1. [https://zh.wikipedia.org/wiki/ARM%E6%9E%B6%E6%A7%8B#cite_note-v8arch-1](https://zh.wikipedia.org/wiki/ARM%E6%9E%B6%E6%A7%8B#cite_note-v8arch-1)
2. [https://www.arm.com/files/downloads/ARMv8_Architecture.pdf](https://www.arm.com/files/downloads/ARMv8_Architecture.pdf)
3. [http://www.lujun.org.cn/?p=3943](http://www.lujun.org.cn/?p=3943)
4. [https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/000-Introduction/introduction.html#//apple_ref/doc/uid/TP30000851-CH211-SW1](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/000-Introduction/introduction.html#//apple_ref/doc/uid/TP30000851-CH211-SW1)

（完）。

[demo地址](https://github.com/fangshufeng/demo_project/tree/master/ArmAssembly)




