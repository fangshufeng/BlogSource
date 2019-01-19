---
layout: post
title: "【8086汇编】-- 常用寄存器"
date: 2018-10-07 15:36:21 +0800
comments: true
categories: 
tag:
    汇编
---


学习汇编最重要的是就是学习寄存器和指令，8086汇编拥有14个16位的寄存器，分别`AX、BX、CX、DX、SI、DI、BP、SP、IP、CS、DS、SS、ES、PSW`
，该篇将来介绍各个寄存器作用。

<!-- more -->

## 通用寄存器
---

`AX、BX、CX、DX`一般存放一些一般的数据，被称为通用寄存器，分别拥有高8位和低8位，以`AX`寄存器为例

![15387929970381.jpg](https://upload-images.jianshu.io/upload_images/3279997-108fd3aae4b83a42.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`H`和`L`分别表示高和低的意思

来个简单的汇编指令

```

mov ax, 1234H

```

上面的意思就是将16进制的1234送入到`ax`寄存器中，其他的通用寄存器类似`ax`

**备注：在8086中 在数字后面加H表示十六进制**


## CS和IP寄存器
---

这2个寄存器是8086CPU中最重要的寄存器，这2个指令决定了CPU将要读取的指令地址。`CS`为代码段寄存器，`IP`为指针指令寄存器。任意时刻cs:ip对应的内存地址的内容都会被当做指令执行，上篇说到，数据和指令没有本质区别，只是特定时刻当做不同用途罢了。


本文采取了`王爽的《汇编语言》`的图来解辅助解释cs和ip寄存器

下图展示了8086CPU读取和执行指令的过程

此时的cs和ip分别是2000和0000。

![15387938093937.jpg](https://upload-images.jianshu.io/upload_images/3279997-37199e9be84eefc1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图2.12 经过地址加法器计算出物理地址为`20000` 

![15387945081888.jpg](https://upload-images.jianshu.io/upload_images/3279997-095024735551d8f7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图2.13 将计算出的物理地址送到输入输出控制电路

![15387947385805.jpg](https://upload-images.jianshu.io/upload_images/3279997-475447300e6b38b4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图2.14 将物理地址20000送到地址总线

![15387948448393.jpg](https://upload-images.jianshu.io/upload_images/3279997-5a3ef1152def0c25.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


找到了物理地址为`20000`的内容为`B8`则吞3个字节内容为`B8 23 01`，对应的汇编指令为`mov ax,0123H`，通过数据总线返回到CPU

![15387949124754.jpg](https://upload-images.jianshu.io/upload_images/3279997-b868f2a4835cae81.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图2.16 通过输入输出电路将内容送到指令缓冲器

![15387950283272.jpg](https://upload-images.jianshu.io/upload_images/3279997-163dfc65f0fc83ae.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图2.17展示此时的IP位增加3B,因为刚才哪个指令增加了3个字节

![15387951153112.jpg](https://upload-images.jianshu.io/upload_images/3279997-f9854c22dc6aafcb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图2.18 接下来执行指令缓存器中的指令

![15387951457944.jpg](https://upload-images.jianshu.io/upload_images/3279997-0f560d0980839f6e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来重复刚才的过程

到目前为止一直都在纸上谈兵，我们通过工具`emu8086`简单看下是不是这样的

新建一个空的工程，这里就写了2条简单的指令

![15387955290141.jpg](https://upload-images.jianshu.io/upload_images/3279997-6d5f9c4a640ca990.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击`emulate`

![15387956280058.jpg](https://upload-images.jianshu.io/upload_images/3279997-e9a4f015729ee5a4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到：

1. cs = 0100,ip = 0000
2. mov ax , 0123h ,对应的指令为`B8 23 01 `, mov bx,0003h, 对应的指令为`BB 03 00`，至于为什么是2301 这是大小端的问题，不知道大小端的同学自行补下课


这时我们点击下`single step`发现代码执行到 第四行 `AX`的已经为`0123`了，并且ip的值已经增加了3，即将执行第四行指令了

![15387958372346.jpg](https://upload-images.jianshu.io/upload_images/3279997-446ee7474438eae3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**所以：cs：ip决定了下面执行哪行代码，也知道了它的重要性了**

那么我们如何修改主动修改cs和ip的值呢❓❓❓ 

遗憾的是8086没有提供直接修改cs和ip的方法，但是可以通过另一个指令`jmp`来达到修改的目的

若想同时修改cs和ip的值，使用`jmp 段地址:偏移地址`


```
jmp 2AE3:3 执行后 cs=2AE3H, IP= 0003H 
```

若想单独修改ip的值，使用`jmp 某一个合法的寄存器` ，


```
mov ax , 0003H

jmp ax 执行前 cs = 2000h ip = 0000h
       执行后 cs = 2000h ip = 0003h


```
 
 执行下面的代码会发现当代码执行到`05`行的时候按道理应该要执行`06`行，然而我们改变了ip，所以代码会又会回到`03`行
 
![15387966459705.jpg](https://upload-images.jianshu.io/upload_images/3279997-453b67606721e5dc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![2018-10-06 12_43_05.gif](https://upload-images.jianshu.io/upload_images/3279997-0b919c852e6a3005.gif?imageMogr2/auto-orient/strip)

## ds寄存器
---

cpu离开了内存是没有用的，那cpu是如何读取内存的数据的呢，ds寄存器就派上用场了，要读取内存，首先的知道物理地址，在8086CPU中，物理地址 = 段地址 X 16 + 偏移地址得到的

加入我们要读取内存地址为`20000H`的内存中放的内容,写法如下

![15387978500515.jpg](https://upload-images.jianshu.io/upload_images/3279997-aac40325007d60ed.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



你可能想说为啥不直接写 mov ds , 2000:0,很遗憾的告诉你不支持，需要通过通用寄存器转一道，ds存放是段地址，`[...]`存放的是偏移地址，那`mov al ,[0]` 不是只有偏移地址吗，嗯，执行的时候会自动读取ds的内容

继续刚才的demo我们把内存为`20000`的内容改成`BB`


![15387980538315.jpg](https://upload-images.jianshu.io/upload_images/3279997-93ee9409ee8d9d9d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行后会发现`al`为`BB`

![15387981081684.jpg](https://upload-images.jianshu.io/upload_images/3279997-9f787cec1f45eee2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





## ss和sp寄存器
---

8086CPU提供了栈来访问内存，有入栈(push)和出栈（pop）指令，可以指定栈的大小，ss指向栈段，sp指向栈顶的指针，栈是非常重要的，8086CPU栈有以下特点：

1. 遵循先进后出的原则
2. push和pop的数据长度以字为单位（2个字节）
3. push导致sp = sp - 2
4. pop 寄存器   导致sp = sp + 2,并将pop的原素赋值给 寄存器

![15388693518737.jpg](https://upload-images.jianshu.io/upload_images/3279997-820d59b89e388e1f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应的指令如下


```
mov ax ,0123h
push ax

mov bx,2266h
push bx

mov cx , 1122h
push cx

pop ax ; 做了2个事情 1. sp = sp + 2 ； 2. ax = 1122h 也就是将出栈的原素赋值给了ax
pop bx
pop cx

```

![15388699177032.jpg](https://upload-images.jianshu.io/upload_images/3279997-befc9cde7920f5fa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击运行之后看视频


![15388963634977.gif](https://upload-images.jianshu.io/upload_images/3279997-e33638015ec81012.gif?imageMogr2/auto-orient/strip)


## 小结
---

至此，已经介绍了9个常用的寄存器了，还剩下5个后面的文章将会一一介绍到的








































