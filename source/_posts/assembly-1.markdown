---
layout: post
title: "【8086汇编】-- 入门知识"
date: 2018-10-06 19:41:46 +0800
comments: true
categories: 
tag:
    汇编
---



## 为什么要学习汇编
---

我们平时写的高级语言经过编译后都会到汇编最终到机器语言，它可以直接操作硬件，对性能高的一些程序采用汇编的方式，比如逆向工程，破解软件的时候都会和汇编打交道，当然对我来说是因为在看runtime源码的过程中，关于`_objc_msgSend`等等一些代码的时候也是汇编实现的，都告诉我是时候学习汇编了，这里并没有直接从`arm`汇编而是从`8086`汇编入门，理由很简单，`简单`，本系列文章是帮助大家汇编入门，更多的细节靠我几篇博客岂能说的清楚，还需要自己大量补课。

<!-- more -->

## 基础知识
---

先搞清楚下面几个概念

*  **【1】cpu**

cpu是个调度中心，它协调电脑的各个部件合作，就是是人类的大脑，大概分为`寄存器`、`运算器`、`控制器`

![15386220855323.jpg](https://upload-images.jianshu.io/upload_images/3279997-27adacbc267f4cfa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/540)


*  **【2】存储器**

 cpu所需要执行的指令和数据都存在存储器中，也就是平时所说的内存。内存的作用仅次于`CPU`磁盘不同于内存。刚刚说到`指令`和`数据`这个对于内存和磁盘在本质上没有区别，都是二进制的`0`和`1`，只是不同的时候对于数据的不同响应而已。内存被划分为一个个的单元，每个单元是一个字节，而一个字节是有8个二进制位构成的。磁盘中的数据以及程序如果读取不到内存就无法被cpu使用,所以知道`CPU`如何和`内存`沟通的非常重要。

*  **【3】总线**

`CPU`和`内存`是通过总线来联系的，`CPU`有很多管脚，每根管脚都和线相连。

![15386229447178.jpg](https://upload-images.jianshu.io/upload_images/3279997-2e909a02ed06778b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总线在逻辑上被分为`地址总线`、`数据总线`、`控制总线`

下图展示了一个具有10根地址总线的cpu向内存发送地址为11时的10根总线的二进制数据情况其实就是`00000 01011`，这些`0`和`1`就是对于的线的低电平和高电平


![15386230896643.jpg](https://upload-images.jianshu.io/upload_images/3279997-353d8ed1644ddc54.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**线的根数成为线的`宽度`，比如上图有10根地址总线，那么地址总线的宽度为`10`**

地址总线的宽度决定了CPU的寻址能力，这个很好理解，比如`8086`CPU地址总线的宽度为`20`那么它的寻址范围为0~ 2^20 - 1，我们称它的寻址能力为，1M

同理，8086的数据总线的宽度为`16`

* 它的宽度决定了CPU的单次数据传送量，也就是数据传送速度
* 8086的数据总线宽度是16，所以单次最大传递2个字节的数据

下面两个图分别展示了`8`和`16`位数据总线的传输次数和速度

![15386244678975.jpg](https://upload-images.jianshu.io/upload_images/3279997-737244435ff3b407.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![15386244927357.jpg](https://upload-images.jianshu.io/upload_images/3279997-4ca5ad625b9264e0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  **【4】8086CPU的寻址方式->段地址:偏移地址**

大家不要被这个吓到，这个只是`8086`CPU用来解决CPU的位数和地址总线的不同的一种手段,`8086`CPU位数是16位的，而地址总线的宽度是20位的，如果CPU直接通过16位寻址，那么就会造成4位的内存浪费，也就是2^20 = 1M的内存实际使用的是2^16 = 64KB,`8086`为了能够充分利用这个内存采用的方式是通过两个16位的地址来合成20位的地址，`物理地址 = 段地址 X 16 + 偏移地址` 。

例如：8086CPU要访问地址为123C8H的内存单元，此时，CPU内部的加法器工作过程如下

![15386251514713.jpg](https://upload-images.jianshu.io/upload_images/3279997-bdb19d7ffd772e2a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



我们更应该关注是解决该问题的手段，以便以后碰到类似的情况能够举一反三。

*  **【5】寄存器**

寄存器是CPU的组成结构，为什么要单独说呢，是因为学习汇编它太重要了，感觉学习汇编就是学习寄存器

![15386255686134.jpg](https://upload-images.jianshu.io/upload_images/3279997-9b99b029990ec8f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不同的CPU寄存器是不一样的，8086CPU的寄存器都是16位的（2个字节），主寄存器一般存放一些基本数据，成为通用寄存器，以`ax`为例，16位又分为高8位和的8位分别为`ah`和`al`

![15386258897451.jpg](https://upload-images.jianshu.io/upload_images/3279997-d32fa98478e85ed9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


下篇将带来寄存器专题，这里不做过多介绍


*  **【6】常用的单位**

由于8086CPU是16位的，对于它只是2个单位，分别为字节（byte）和 字（word）

![15386264449948.jpg](https://upload-images.jianshu.io/upload_images/3279997-b4bc0185b6bf888d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 练习工具
---

![15386265388880.jpg](https://upload-images.jianshu.io/upload_images/3279997-d1d80cc562bafcd2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里是采用的Windows下的`emu8086`，后面大部分的demo都是通过这个来实现的


声明：

本文很多的图片来自王爽的《汇编语言》



























