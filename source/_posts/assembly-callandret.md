---
title: 【8086汇编】-- call 和 ret 指令 的应用和本质
date: 2018-10-07 15:20:44
tags:
    汇编
---


在高级语言开发中会把一些功能封装成方法然后调用，下面我们来用汇编实现一个打印`hello world`的方法

<!-- more -->

先实现打印的功能

```
assume cs:code    ds:data   ;assume是伪指令 作用是申明本程序中cs为code ds为data


data segment                ;segment .... ends 定义一个段

   string db 'Hello World!$'  
    
data ends


code segment 
    
start:                      ;start 伪指令地址的标号

      
      mov ax, data
      mov ds, ax
          
       ;ds + dx 才能给打印函数传参
      mov dx,offset string
      
      ;当ah == 9h 的时候 调用系统的print函数 
      mov ah,9h  
      int 21h    
       
      ;当ah == 4ch 的时候 调用系统的退出函数
      mov ax, 4c00h
      int 21h

      
code ends

end start
```

发现是正常输出的

![15373275595442.jpg](https://upload-images.jianshu.io/upload_images/3279997-87175bd4507997b3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



如果想把打印封装成一个函数，在汇编里面通过`call`和`ret`指令配合使用的



```
assume cs:code    ds:data    ss:stack
   

stack segment
       db 10 dup(0) ;申请10个字节，并且默认都是0 
stack ends       

data segment

   string db 'Hello World!$'  
    
data ends



code segment 
    
start:
      
      mov ax, data
      mov ds, ax 
      
      mov ax , stack
      mov ss ,ax
      mov sp , 10h
      
      call printFunc    
                      
      mov ax , 1122h                
      ;当ah == 4ch 的时候 调用系统的退出函数
      mov ax, 4c00h
      int 21h

printFunc: ; printFunc为伪指令，值为下面一行的地址
      
      ;ds + dx 才能给打印函数传参
      mov dx,offset string
      
      ;当ah == 9h 的时候 调用系统的print函数 
      mov ah,9h  
      int 21h    
             
       ret      
      
code ends

end start

```




通过`call`和`ret`指令就可以实现一个方法调用了，相信你已经发现了下面的代码增加了一个`stack`段，这就涉及到`call`和`ret`指令的本质了

## `call`和`ret`指令的本质

`call`指令，相当于 

> 1. push IP // 具体应该说是call下面一行的ip
> 2. jmp 标号处

`ret`指令，相当于

> pop IP

### call指令
我们来查看程序执行过程来证明下

点击下图执行代码

![15373376970904.jpg](https://upload-images.jianshu.io/upload_images/3279997-506ba2888bfb3c09.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先来看看`call`代码的下一行的对应的cs:ip是多少，看下图

![15373385706754.jpg](https://upload-images.jianshu.io/upload_images/3279997-b4a8f06cafe156e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



标号`printFunc`的cs:ip如图
![15373387353222.jpg](https://upload-images.jianshu.io/upload_images/3279997-63802b89d889a2f6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



由上图可以知道`30`那行代码的cs:ip为`0712:0010`，看下此时栈的内容

![15373382064902.jpg](https://upload-images.jianshu.io/upload_images/3279997-35e3d2c01960d92c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


由于代码还没到那行，此时的栈为空栈所以此时的`sp`为`000A`，当代码执行到`call printFunc`时，之前说到`call`指令，那么过了28行代码应该有入栈操作


![15373384702103.jpg](https://upload-images.jianshu.io/upload_images/3279997-51bfea23f3592986.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们发现了2个事情：

1. 确实将`30`行的ip == 0010 入栈了；
2. 代码通过cs:ip即 0712：0018 确实到了标号处

从而验证了`call`指令的正确性

### ret指令

还是上面的代码当执行到`ret`指令时

![15373390067711.jpg](https://upload-images.jianshu.io/upload_images/3279997-51afc637980bee1a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


此时的cs:ip为0712:001F,安照对`ret`指令的描述，那么此时应该是出栈操作并赋值给ip，我们过掉44行，发现代码执行到该行，如图

![15373391951998.jpg](https://upload-images.jianshu.io/upload_images/3279997-43e9d290e8ea021d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


和我们猜想的一样，再看看此时栈中的情况

![15373392362966.jpg](https://upload-images.jianshu.io/upload_images/3279997-cec64959deae50c2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发现栈定的指针已经+2了，这也符合我们的逾期，至此，已经证明了`ret`指令的结论

## 自己实现函数调用

既然我们清楚了`call`和`ret`的本质，我们是可以不使用这2个指令也可以实现函数调用和返回的修改代码如下


```
assume cs:code    ds:data    ss:stack
   

stack segment
       db 10 dup(0)
stack ends       

data segment

   string db 'Hello World!$'  
    
data ends



code segment 
    
start:
      
      mov ax, data
      mov ds, ax 
      
      mov ax , stack
      mov ss ,ax
      
      mov sp , 10
      
      ;call printFunc
      push    0011h
      jmp    printFunc
                      
      mov ax , 1122h                
      ;当ah == 4ch 的时候 调用系统的退出函数
      mov ax, 4c00h
      int 21h

printFunc: ; printFunc为伪指令，值为下面一行的地址
      
      ;ds + dx 才能给打印函数传参
      mov dx,offset string
      
      ;当ah == 9h 的时候统的print函数  调用系
      mov ah,9h  
      int 21h    
             
       ;ret
       pop ax
       jmp ax      
      
code ends

end start

```


我将`call printFunc`换成了 


```
 push    0011h // 0011h是 mov ax , 1122h 的ip
jmp    printFunc
      
```

`ret`换成了


```
 pop ax
 jmp ax  
```

同样实现了函数调用和返回
















