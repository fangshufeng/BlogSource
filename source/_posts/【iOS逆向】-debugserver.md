---
title: 【iOS逆向】-- debugserver
date: 2018-12-12 21:21:20
tags:
    [iOS逆向,ASLR]
---


## Xcode 为什么可以调试APP?

平时开发中当我们给代码打断点，调试程序（lldb），这一切都离不开一个媒介`debugserver`，它负责将`lldb`指令给到`app`，然后`app`将结果通过`debugserver`传给`lldb`

`debugserver`一开始是在`Xcode`中的，一旦手机连接`Xcode`信任后，`debugserver`便会安装到手机上

1. `Xcode`目录： `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport/9.1/DeveloperDiskImage.dmg/usr/bin/debugserver`；

2. `iPhone`目录：`/Developer/usr/bin/debugserver`



但是缺少`task_for_pid`权限，通过`Xcode`安装的`debugserver`，只能调试自己的app,要想逆向别人的app，这种肯定是行不通的。前面说到，不能调试别人app的原因是权限不够，要想在没有源码的情况下调试别人的app，就需要修改`debugserver`权限。

<!-- more -->    

## 更改`debugserver` 权限

### 认识debugserver

 放在`/Developer/usr/bin/debugserver`是没有调试其他app的权限的，现在的做法是先把`iPhone`上的`debugserver`放到电脑上修改好权限再放回手机上，已达到可以调试其他app的目的。
 
 拖到`MachOView`中可以看到这是个胖二进制文件
 
![15443398356634.jpg](https://upload-images.jianshu.io/upload_images/3279997-032685ebbe97828a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 给debugserver瘦身

由于`debugserver`是个胖二进制文件，我的越狱手机是`iPhone6plus`、`iOS9.0.1`是`arm64`架构的，我们只要流行`arm64`即可使用`lipo`命令


```

lipo -thin arm64 ./debugserver -o ./debugserver_arm64

```


![15443403611713.jpg](https://upload-images.jianshu.io/upload_images/3279997-ee40ea0fb19742ef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


为了方便将`debugserver_arm64`改为`debugserver`，下面的提到的`debugserver`指的都是`debugserver_arm64`。

###  给debugserver增加 task_for_pid权限

#### 查看`debugserver`原来的权限

通过`ldid`查看原来的命令


```
➜  debugserver ldid -e ./debugserver
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.backboardd.debugapplications</key>
	<true/>
	<key>com.apple.backboardd.launchapplications</key>
	<true/>
	<key>com.apple.diagnosticd.diagnostic</key>
	<true/>
	<key>com.apple.frontboard.debugapplications</key>
	<true/>
	<key>com.apple.frontboard.launchapplications</key>
	<true/>
	<key>com.apple.security.network.client</key>
	<true/>
	<key>com.apple.security.network.server</key>
	<true/>
	<key>com.apple.springboard.debugapplications</key>
	<true/>
	<key>run-unsigned-code</key>
	<true/>
	<key>seatbelt-profiles</key>
	<array>
		<string>debugserver</string>
	</array>
</dict>
</plist>

```


将旧的权限输出到`entitlement.plist`文件中


```
➜  debugserver ldid -e ./debugserver > ./entitlement.plist
```

在`entitlement.plist`增加一个`task_for_pid-allow`



![15443412160922.jpg](https://upload-images.jianshu.io/upload_images/3279997-f390b3dce4853955.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 重签名`debugserver`

将改好的`entitlement.plist`重签`debugserver`


```
➜  debugserver ldid -Sentitlement.plist debugserver

```


再次查看`debugserver`权限确认是否添加成功


```
➜  debugserver ldid -e ./debugserver | grep task_for_pid-allow
	<key>task_for_pid-allow</key>
➜  debugserver

```

说明成功添加


## 将`debugserver` 放回到手机

前面已经赋予了`debugserver`可以调试其他`app`的权限了，接下来放回到手机上使用。

此时我们把修改后的`debugserver`放到手机的`/usr/bin/`目录下，原因有2个

1. 手机上原来的`debugserver`存放的目录`/Developer/usr/bin/debugserver`是只读的；
2. 放到`/usr/bin/`下可以在手机上直接敲`debugserver`方便使用


顺便增加执行权限


```
iPhone:~ root# chmod +x /usr/bin/debugserver
```


```
iPhone:~ root# ls -l /usr/bin/debugserver
-rwxr-xr-x 1 root wheel 4646672 Dec  9 15:49 /usr/bin/debugserver
iPhone:~ root#

```


## 用`debugserver`启动或附加进程

### 启动进程


```
debugserver -x backboard IP:port /path/to/executable
```

debugserver会启动executable，并开启port端口， 等待来自IP的LLDB接入。



先实现一个打开系统的app,iPhone系统应用都放在`/Applications`下面

![15443469873180.jpg](https://upload-images.jianshu.io/upload_images/3279997-4a560add5618c60e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/800/h/500)


已打开系统的计算器为例


```
iPhone:~ root# debugserver -x backboard *:1234 /Applications/Calculator.app/Calculator

```

上面的代码会启动`Calculator.app`，并开启1234端 口，等待任意IP地址的LLDB接入。

### 附加进程



```
debugserver IP:port -a "ProcessName"
```


`debugserver`会附加`ProcessName`，并开启port端 口，等待来自IP的LLDB接入。

已打开手机上的新浪微博为例子

用户安装的app放在`/var/mobile/Containers/Bundle/Application`下

先点击`新浪微博`

```
iPhone:~ root# debugserver *:1234 -a "Weibo"
debugserver-@(#)PROGRAM:debugserver  PROJECT:debugserver-340.3.51.1
 for arm64.
Attaching to process Weibo...
Listening to port 1234 for a connection from *...

```

`-a`是附加的意思

## `lldb`连接`debugserver`


现在手机端的服务已经开启。

先电脑端输入`lldb`指令

```
➜  ~ lldb
(lldb)
```

连接服务


```
process connect connect://192.168.1.102:1234

```

`ip`就是手机的ip地址
`1234`就是端口


稍微等一会（1到2分钟吧）就会出现下面的东西


```
Process 4348 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x000000019b3c4c30 libsystem_kernel.dylib`mach_msg_trap + 8
libsystem_kernel.dylib`mach_msg_trap:
->  0x19b3c4c30 <+8>: ret

libsystem_kernel.dylib`mach_msg_overwrite_trap:
    0x19b3c4c34 <+0>: mov    x16, #-0x20
    0x19b3c4c38 <+4>: svc    #0x80
    0x19b3c4c3c <+8>: ret
Target 0: (Weibo) stopped.
(lldb)
```

此时的app是无法交互的我们输入`c`继续程序


```
(lldb) c
Process 4348 resuming
(lldb)

```

会发现微博可以交互了。


## 一个简单的demo


### 连接手机

```
ssh root@10.9.24.154 
```

你也可以使用usb连接


### 开启手机服务

```
debugserver *:1234 -a 818

```

### mac端输入`lldb`指令连接手机

```
➜  ~ lldb
(lldb) process connect connect://10.9.24.154:1234
Process 818 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x000000019811cc30 libsystem_kernel.dylib`mach_msg_trap + 8
libsystem_kernel.dylib`mach_msg_trap:
->  0x19811cc30 <+8>: ret

libsystem_kernel.dylib`mach_msg_overwrite_trap:
    0x19811cc34 <+0>: mov    x16, #-0x20
    0x19811cc38 <+4>: svc    #0x80
    0x19811cc3c <+8>: ret
Target 0: (Debugserver) stopped.

```

### 计算`-[ViewController btnClick]:`的真实函数地址


#### 查看ASLR

##### 什么是ASLR

全称`ASLR (Address Space Layout Randomization) `每次进程启动时，同一进程的所有模块在虚拟内存中的起始 地址都会产生随机偏移。就拿我们的例子来说，在虚拟内存的起始地址是`0x0`


![15446038665381.jpg](https://upload-images.jianshu.io/upload_images/3279997-17ff3f49071fadf2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们先不考虑随机地址偏移，也就是说明了这个`macho`加载内存的时候`_TEXT`段的起始地址应该是`0x100000000`,但是如果有了偏移量，那么我们加上偏移量就能计算出内存中的真实地址了。

```
(lldb) image list -o -f
[  0] 0x00000000000dc000 /Users/fangshufeng/Library/Developer/Xcode/iOS DeviceSupport/9.0.1 (13A404)/Symbols/usr/lib/dyld
[  1] 0x00000000000b0000 /var/mobile/Containers/Bundle/Application/FA2608EB-E892-4C24-9272-F40908101AA0/Debugserver.app/Debugserver(0x00000001000b0000)
[  2] 0x00000001000c4000 /Library/MobileSubstrate/MobileSubstrate.dylib(0x00000001000c4000)

[...]
```

1. 第一列[X]是模块的序
2. 第二列是模块在虚拟内存中的起始地址因ASLR 而产生的随机偏移（以下简称ASLR偏移）；
3. 第三列 是模块的全路径，括号里是偏移之后的起始地址。

可以看到偏移量为`0x40000`


#### hopper查看再Macho中的值


![15445997853528.jpg](https://upload-images.jianshu.io/upload_images/3279997-32bc9034b5fe6b35.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以真实的函数地址是 = `0x00000000000b0000` + `0x0000000100006738` == `0x1000B6738`;

### 在`-[ViewController btnClick]:`设置断点


```
(lldb) breakpoint set -a 0x1000B6738
Breakpoint 3: where = Debugserver`-[ViewController btnClick], address = 0x00000001000b6738
(lldb)
```

#### 输入`c`继续程序

点击按钮

```
(lldb) breakpoint set -a 0x1000B6738
Breakpoint 3: where = Debugserver`-[ViewController btnClick], address = 0x00000001000b6738
(lldb) c
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
    frame #0: 0x00000001000b6738 Debugserver`-[ViewController btnClick]
Debugserver`-[ViewController btnClick]:
->  0x1000b6738 <+0>:  sub    sp, sp, #0x20             ; =0x20
    0x1000b673c <+4>:  mov    w8, #0x14
    0x1000b6740 <+8>:  mov    w9, #0xa
    0x1000b6744 <+12>: str    x0, [sp, #0x18]
Target 0: (Debugserver) stopped.
(lldb)
```


可以看到已经断到了方法处，可以和`hopper`的内容对比一看是一样的。

#### 简单使用`lldb`指令

```
(lldb) ni
Process 1083 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x000000018818a3e4 UIKit`<redacted> + 100
UIKit`<redacted>:
->  0x18818a3e4 <+100>: cmp    x22, #0x0                 ; =0x0
    0x18818a3e8 <+104>: cset   w0, ne
    0x18818a3ec <+108>: ldp    x29, x30, [sp, #0x20]
    0x18818a3f0 <+112>: ldp    x20, x19, [sp, #0x10]
Target 0: (Debugserver) stopped.
(lldb) ni
Process 1083 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x000000018818a3e8 UIKit`<redacted> + 104
UIKit`<redacted>:
->  0x18818a3e8 <+104>: cset   w0, ne
    0x18818a3ec <+108>: ldp    x29, x30, [sp, #0x20]
    0x18818a3f0 <+112>: ldp    x20, x19, [sp, #0x10]
    0x18818a3f4 <+116>: ldp    x22, x21, [sp], #0x30
Target 0: (Debugserver) stopped.
(lldb)

```

##### 打印寄存器的值


```

(lldb) po $w8
20

(lldb) p $w9
(unsigned int) $1 = 10
(lldb)
```

##### 打印层级

```
(lldb) po [[[UIApplication sharedApplication] keyWindow] recursiveDescription]
<UIWindow: 0x14c63e650; frame = (0 0; 414 736); autoresize = W+H; gestureRecognizers = <NSArray: 0x14c637830>; layer = <UIWindowLayer: 0x14c63d3c0>>
   | <UIView: 0x14c54fe20; frame = (0 0; 414 736); autoresize = W+H; layer = <CALayer: 0x14c54f000>>
   |    | <_UILayoutGuide: 0x14c550280; frame = (0 0; 0 20); hidden = YES; layer = <CALayer: 0x14c54e040>>
   |    | <_UILayoutGuide: 0x14c550c30; frame = (0 736; 0 0); hidden = YES; layer = <CALayer: 0x14c63c690>>
   |    | <UIButton: 0x14c641240; frame = (100 100; 100 100); opaque = NO; layer = <CALayer: 0x14c6411f0>>
   |    |    | <UIButtonLabel: 0x14c63a3c0; frame = (23 39.3333; 54 21.6667); text = '点我啊'; opaque = NO; userInteractionEnabled = NO; layer = <_UILabelLayer: 0x14c633470>>
   |    |    |    | <_UILabelContentLayer: 0x14c651170> (layer)

(lldb)

```

##### 给按钮加一个黄色背景

```
(lldb) po  [0x14c641240 setBackgroundColor:[UIColor yellowColor] ]
0x0000000000000001

(lldb) c
Process 1083 resuming
(lldb)

```


![15446009775607.jpg](https://upload-images.jianshu.io/upload_images/3279997-3003d84f1a89bc02.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)



[demo地址](https://github.com/fangshufeng/demo_project)

（完）
















