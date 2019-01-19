---
layout: post
title: "【8086汇编】-- 函数的本质"
date: 2018-10-07 15:43:57 +0800
comments: true
categories: 
tag:
    [汇编,函数调用协议]
---

作为开发相信对函数的调用都会很感兴趣吧

<!-- more -->

下面通过汇编的角度来分析下

## 一、函数的入参和返回值

```
assume cs:code    ss:stack       ds:data

stack segment
        
      db 20 dup(0)  
stack ends


data segment 
     result2 db 0h
     result db 0h
data ends    


code segment
         
start: 

     mov ax,data
     mov ds,ax
            
     mov ax,stack       
     mov ss , ax      
     mov sp, 20
      
     
     ;第一种      
     ;mov ax , 1111h  
     ;mov bx, 2222h
     ;call sum
     
     ;第二种  
     ;call sum2
     
     ;第三种 通过poo两次来维持栈平衡
      ;push  1111h
      ;push  2222h
      ;call sum3 
      ;pop si
      ;pop si 
                
      ;第四种  通过移动sp指针来维持栈平衡
      ;push  1111h
      ;push  2222h
      ;call sum3
      ;add sp,4 
        
      ;第五种  通过移动sp指针来维持栈平衡  
      push  1111h
      push  2222h
      call sum4
      
     mov ax, 4c00h
     int 21h 
     
 ;通过寄存器 存储值 速度快   
 ; 入参放到bx,cx， 返回值在ax上
sum:
    mov cx , bx
    add ax ,cx
    
    ret   

;将入参存到ds段中 缺点：全局共享
sum2:

    mov offset result,22h
    
    ret     
        
;将入参放到栈中
sum3:
    
    mov bp,sp ;  由于无法直接修改sp的值 这里利用bp寄存器暂时保存下sp的值
    mov ax,  ss:[bp + 2]
    add ax,  ss:[bp + 4]
    
    ret     ;外平栈  需要调用者主动回收栈空间

;将入参放到栈中
sum4:
    
    mov bp,sp ;
    mov ax,  ss:[bp + 2]
    add ax,  ss:[bp + 4]
    
    ret  4   ;内平栈        
         
code ends     


end start
```

上面展示了一个函数入参的方式有哪些，一般的cpu都是少量参数通过寄存器传参，当参数大于一定个数的时候就会采用栈传递参数,各个cpu不一样。可以通过函数调用协议来制定参数传递的方式的，分别有`__stdcall`、`__cdecl`、`__fastcall`,下面通过`win32`来看看各个关键字有什么不同


### __cdecl


```
int __cdecl sum2(int a, int b) {
	return a + b;
}

int main(int argc, char* argv[])
{

	sum2(1,2);
	return 0;
}


```

运行查看汇编如下


```
main函数汇编截取

...
0040D4E8   push        2
0040D4EA   push        1
0040D4EC   call        @ILT+50(sum2) (00401037)
0040D4F1   add         esp,8
...


sum2函数汇编

00401050   push        ebp
00401051   mov         ebp,esp
00401053   sub         esp,40h
00401056   push        ebx
00401057   push        esi
00401058   push        edi
00401059   lea         edi,[ebp-40h]
0040105C   mov         ecx,10h
00401061   mov         eax,0CCCCCCCCh
00401066   rep stos    dword ptr [edi]

//先关注下面2行代码
00401068   mov         eax,dword ptr [ebp+8]
0040106B   add         eax,dword ptr [ebp+0Ch]


0040106E   pop         edi
0040106F   pop         esi
00401070   pop         ebx
00401071   mov         esp,ebp
00401073   pop         ebp
00401074   ret

```

可以看出`__cdecl`参数的传递方式是通过栈传递，参数入栈的方式是从右到左一次传递，并且是外平栈，也是默认的方式

### __stdcall

修改代码

```

int __stdcall sum2(int a, int b) {
	return a + b;
}

int main(int argc, char* argv[])
{

	sum2(1,2);
	return 0;
}
```

汇编代码如下

```
main函数

0040D4E8   push        2
0040D4EA   push        1
0040D4EC   call        @ILT+55(sum2) (0040103c) // 此时调用者已经没有回收栈了


sum2:
00401050   push        ebp
00401051   mov         ebp,esp
00401053   sub         esp,40h
00401056   push        ebx
00401057   push        esi
00401058   push        edi
00401059   lea         edi,[ebp-40h]
0040105C   mov         ecx,10h
00401061   mov         eax,0CCCCCCCCh
00401066   rep stos    dword ptr [edi]
00401068   mov         eax,dword ptr [ebp+8]
0040106B   add         eax,dword ptr [ebp+0Ch]
0040106E   pop         edi
0040106F   pop         esi
00401070   pop         ebx
00401071   mov         esp,ebp
00401073   pop         ebp

// 和__cdecl不同之处
00401074   ret         8

```

可以得出`__stdcall`从右到左入栈，是内平栈

### __fastcall


再修改代码


```
int __fastcall sum2(int a, int b) {
	return a + b;
}

int main(int argc, char* argv[])
{

	sum2(1,2);
	return 0;
}
```

此时汇编


```
main

0040D4E8   mov         edx,2
0040D4ED   mov         ecx,1
0040D4F2   call        @ILT+60(sum2) (00401041)

sum2:
00401050   push        ebp
00401051   mov         ebp,esp
00401053   sub         esp,48h
00401056   push        ebx
00401057   push        esi
00401058   push        edi
00401059   push        ecx
0040105A   lea         edi,[ebp-48h]
0040105D   mov         ecx,12h
00401062   mov         eax,0CCCCCCCCh
00401067   rep stos    dword ptr [edi]
00401069   pop         ecx

//分别读取参数
0040106A   mov         dword ptr [ebp-8],edx
0040106D   mov         dword ptr [ebp-4],ecx


00401070   mov         eax,dword ptr [ebp-4]
00401073   add         eax,dword ptr [ebp-8]
00401076   pop         edi
00401077   pop         esi
00401078   pop         ebx
00401079   mov         esp,ebp
0040107B   pop         ebp
0040107C   ret

```

可以看出`__fastcall`参数是放在寄存器中的

再修改下代码增加2个参数


```
int __fastcall sum2(int a, int b,int c,int e) {
	return a + b + c + e;
}

int main(int argc, char* argv[])
{

	sum2(1,2,3,4);
	return 0;
}

```
汇编如下


```
main

0040D4E8   push        4
0040D4EA   push        3
0040D4EC   mov         edx,2
0040D4F1   mov         ecx,1
0040D4F6   call        @ILT+70(sum2) (0040104b)

sum2:
00401050   push        ebp
00401051   mov         ebp,esp
00401053   sub         esp,48h
00401056   push        ebx
00401057   push        esi
00401058   push        edi
00401059   push        ecx
0040105A   lea         edi,[ebp-48h]
0040105D   mov         ecx,12h
00401062   mov         eax,0CCCCCCCCh
00401067   rep stos    dword ptr [edi]
00401069   pop         ecx
0040106A   mov         dword ptr [ebp-8],edx
0040106D   mov         dword ptr [ebp-4],ecx
00401070   mov         eax,dword ptr [ebp-4]
00401073   add         eax,dword ptr [ebp-8]
00401076   add         eax,dword ptr [ebp+8]
00401079   add         eax,dword ptr [ebp+0Ch]
0040107C   pop         edi
0040107D   pop         esi
0040107E   pop         ebx
0040107F   mov         esp,ebp
00401081   pop         ebp
00401082   ret         8


```

可以看出`__fastcall`当参数个数大于2时，参数会从右到左入栈，并且是内平栈


综上在win32上可以得出以下结论


| 概要 | __cdecl | __stdcall | __fastcall |
| --- | --- | --- | --- |
| 参数传递方式 | 从右到左 | 从右到左 | 左边开始的两个不大于4字节（DWORD）的参数分别放在ECX和EDX寄存器，其余的参数自右向左压栈传送 |
| 栈清理方式 | 外平栈 | 内平栈 | 大于2个参数内平栈 |
| 场合 | C/C++ | WinApi | 性能要求高的场合 |


## 二、函数栈帧

所谓函数栈帧就是从进入函数，到出函数，内存的变化，也叫函数的执行环境，调用函数要保持堆栈平衡，所谓堆栈平衡就是当你调用函数开始，到调用函数结束，你的内存应该是没有变化的，从哪开始还是要回到那里，其实上面有些程序就已经贴出了完整的堆栈平衡的代码了，下面例子说明


1. 申请一个大小为20个字节的栈空间


```

assume cs:code ds:data   ss:stack

stack segment
       
       db 20 dup (3)
       
stack ends    

code segment

start:
   
  mov ax,stack
  mov ss , ax
  mov sp,20
    
  mov ax , 4c00h
  int 21h
  
sum:
    ret  
    
code ends

end start
```

内存中的表现如下由于此时是空栈所以此时的sp指向`0x07114`

![15388855487540.jpg](https://upload-images.jianshu.io/upload_images/3279997-612b21d5b5acf961.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


也就是下面的这个图


![15388856524514.jpg](https://upload-images.jianshu.io/upload_images/3279997-1d93a39f48c36b95.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/400)


sum函数的作用是计算`a`和`b`的和，我们采用栈的方式传参数

增加下面代码

![15388854800030.jpg](https://upload-images.jianshu.io/upload_images/3279997-1e8a91c1db38038c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


当程序执行到21行的时候内存如下

![15388857827393.jpg](https://upload-images.jianshu.io/upload_images/3279997-a8a58d90d72fefe9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

此时的sp如下

![15388861386780.jpg](https://upload-images.jianshu.io/upload_images/3279997-73a210d41ae3dde2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/h/300)



当执行到21行的时候也就是ret，相当于pop ip 等价于jmp 0011,所以发送2件事情

1. 出栈0011
2. jmp ip

所以程序执行到当时调用函数的下一行此时的sp为`0x07110`

![15388863912044.jpg](https://upload-images.jianshu.io/upload_images/3279997-b5c1c243b27ef5f1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/h/300)




我们发现一个问题，当你的sum函数调用完成后`0x07113~0x07110`这段内存一直没有释放，当再有数据push进来sp又会-2这样内存会越来越少，所以我们写的函数是有问题的，为了解决这个问题，我们增加下面的代码

![15388865888171.jpg](https://upload-images.jianshu.io/upload_images/3279997-abc9b3e9d8ce4587.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/500)


由于我们push参数导致栈sp减少了4个字节那么我们调用完函数主动回到sp之前的位置，这样堆栈就平衡了。

到这里我们还没有使用我们传入的参数，接下来接着开发代码使用传入的2个参数,按照刚才的理解只要将ss:sp的地址+2就可以了，很自然的想到是下面的写法


![15388873275425.jpg](https://upload-images.jianshu.io/upload_images/3279997-d5aeee6896a7394f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/500)

运行之后发现貌似不对，直接这样写是不可以的，这个时候需要借助一个新的寄存器`bp`来暂时保存sp的值

![15388874528384.jpg](https://upload-images.jianshu.io/upload_images/3279997-5db0d30a91f52ecc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


运行之后发现确实保存到`ax`、`bx`中了

![15388875227906.jpg](https://upload-images.jianshu.io/upload_images/3279997-3cd339506c5d7d6c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来我们实现`a`+ `b`了

增加代码，将计算的结果放到ax中

![15388876034083.jpg](https://upload-images.jianshu.io/upload_images/3279997-e5354047b077550e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


最后计算的结果为`4466`

现在我们的需求变了，当我们sum函数要变成下面的样子

```
int sum(int a,int b) {
    int  c = 4;
    int  d = 5;
    int  e = c + d;
    return a + b + e; 
}
```

也就是函数的内部多了2个局部变量，此时又该如何实现呢，做法是先给函数的局部变量申请一定的空间，比如我们这里申请10个字节的长度
![15388886678280.jpg](https://upload-images.jianshu.io/upload_images/3279997-4d0bc4abdefe2ff6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/h/600)


这样就实现了，但是还有问题，我们一开始将sp-10了，那函数调用完毕，应该要恢复,由于bp保存着sp一开始的值所以

![15388887848866.jpg](https://upload-images.jianshu.io/upload_images/3279997-c0b84e97bd9fa365.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实上面的程序还是有些瑕疵的，那就是bp，目前我们是只有一个函数，假如sum函数内部还要掉用其他的函数，那么bp就会指向新函数内部的sp，当新函数调用完成的时候返回到sum函数中，当执行到mov sp, bp的时候那么就会又到了新函数的内部了，那就乱套了，为了解决这个问题，方案如下，就是每次进入一个函数之前先保存旧值，调用完成后恢复

![15388894774363.jpg](https://upload-images.jianshu.io/upload_images/3279997-f564e1962b2ab447.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/h/600)


到这里距离我们完整的函数越来越近了，下面再来优化下，当初我们为局部变量申请的空间，我们会填充一些有意义的东西在win32里面填充的是`CCCC`

![15388897733142.jpg](https://upload-images.jianshu.io/upload_images/3279997-ab968cbd4fcccfc8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/h/600)


还有最后一个地方，当函数内部有用到需要用到的寄存器的时候也要先保护再恢复，套路都是一样的

![15388899344631.jpg](https://upload-images.jianshu.io/upload_images/3279997-b4f86db8856165c7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/h/600)



至此整个一个完整的函数调用就完成了，我们将bp和sp之间的部分叫做某个函数的栈帧，网上关于函数栈帧的图片都大同小异，选了一张来自《深入理解计算机系统》

![15388960276057.jpg](https://upload-images.jianshu.io/upload_images/3279997-976e4421c6c833c9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/500)
































