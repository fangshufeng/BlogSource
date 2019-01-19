---
title: 如何使用DSYM文件来符号化崩溃信息
date: 2019-01-18 21:20:45
tags: [Objective-C,DSYM,ATOS,symbolicatecrash,dwarfdump]
---



## 一、为什么要符号化？

对应线上`app`闪退日志，闪退的堆栈都是以下格式

![15476967747621.jpg](https://upload-images.jianshu.io/upload_images/3279997-f742e6f5f55ea24a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这种信息开发肯定无法找到具体的报错的地方的，本文就是这将这些转成下面这种可读的形式，方便查找问题的

![15476967966597.jpg](https://upload-images.jianshu.io/upload_images/3279997-0529b6d7f0a88def.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<!-- more -->

## 二、获取`DSYM`文件

### 2.1 为什么不能符号化

不是发包的开发的电脑是无法符号化崩溃信息的原因是没有一个叫`.DSYM`的文件，这是个[`Macho`](https://fangshufeng.com/2018/11/29/%E8%AE%A4%E8%AF%86MachO/)格式的文件,打`release`包的时候会生成`xxx.xcarchive`文件，文件内容如下


```mm
.
├── Info.plist
├── Products
│   └── Applications
│       └── xxx.app
│ ...
├── SCMBlueprint
│   └── ...
└── dSYMs
    └── xxx.app.dSYM
```

而函数的地址和函数名称的对应关系就保存在`xxx.app.dSYM`中，而这个文件只要发包的电脑才有。

### 2.2 debug环境如何获取`dsym`

发`release`版本的时候会生成`xxx.app.dSYM`，但是`debug`则没有生成，这完全是因为`xcode`的配置的原因，若想`debug`版本也会生成`xxx.app.dSYM`文件，按下图做就可以了。

![15477038170416.jpg](https://upload-images.jianshu.io/upload_images/3279997-3b72cf6d133c0bc1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


运行后`~/Library/Developer/Xcode/DerivedData/Dsym_demo-cydxdahtbniagjaawvdbciptpkpb/Build/Products/Debug-iphoneos`，会出现


```
├── Dsym_demo.app
    ...
└── Dsym_demo.app.dSYM
    ...
```


### 2.3 从`ipa`包内提取出`dsym`

从已有的可执行文件中提取`DSYM`


```
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/dsymutil ~/Dsym_demo.app/Dsym_demo -o manual.DSYM

```


## 三、手动分析奔溃.crash文件

### 3.1 认识.crash文件

将手机上的`设置` -> `隐私` -> `分析` 中的`.ips`文件改为`.crash`即可，文件内容如下


```mm
{"app_name":"Dsym_demo","timestamp":"2019-01-16 18:07:48.82 +0800","app_version":"1.0","slice_uuid":"e0cffa85-f688-305e-b85d-4ed9a320f8c7","adam_id":0,"build_version":"1","bundleID":"com.dianrong.Dsym-demo2","share_with_app_devs":true,"is_first_party":false,"bug_type":"109","os_version":"iPhone OS 12.0.1 (16A404)","incident_id":"6FE3A02C-A9D0-433C-B855-741684067E51","name":"Dsym_demo"}
Incident Identifier: 6FE3A02C-A9D0-433C-B855-741684067E51
CrashReporter Key:   3e873c922c362f374a3bda3e084de5a0bd41dd58
Hardware Model:      iPhone7,2
Process:             Dsym_demo [12548]
Path:                /private/var/containers/Bundle/Application/112179B6-F1BB-4C6A-9097-7C39AD59A84C/Dsym_demo.app/Dsym_demo
Identifier:          com.dianrong.Dsym-demo2
Version:             1 (1.0)
Code Type:           ARM-64 (Native)
Role:                Non UI
Parent Process:      launchd [1]
Coalition:           com.dianrong.Dsym-demo2 [2261]


Date/Time:           2019-01-16 18:07:48.4121 +0800
Launch Time:         2019-01-16 18:07:47.6977 +0800
OS Version:          iPhone OS 12.0.1 (16A404)
Baseband Version:    7.10.00
Report Version:      104

Exception Type:  EXC_CRASH (SIGABRT)
Exception Codes: 0x0000000000000000, 0x0000000000000000
Exception Note:  EXC_CORPSE_NOTIFY
Triggered by Thread:  0

Application Specific Information:
abort() called

Last Exception Backtrace:
(0x1e50ffef8 0x1e42cda40 0x1e500fee0 0x100fde768 0x100fde700 0x211941380 0x2119417b0 0x211f3c2d0 0x211f3c878 0x206278cd0 0x211f4d86c 0x211efd480 0x211f02e1c 0x2117a3c18 0x2117ac6f0 0x2117a3894 0x2117a4234 0x2117a2334 0x2117a1fe0 0x2117a71a0 0x2117a8100 0x2117a7058 0x2117abd9c 0x211f01314 0x211aecc34 0x1e7b2f890 0x1e7b3a658 0x1e7b39d50 0x1e4b38484 0x1e4adbe58 0x1e7b6e640 0x1e7b6e2cc 0x1e7b6e8e8 0x1e508e5b8 0x1e508e538 0x1e508de1c 0x1e5088ce8 0x1e50885b8 0x1e72fc584 0x211f04bc8 0x100fde814 0x1e4b48b94)

Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x00000001e4c95104 0x1e4c72000 + 143620
1   libsystem_pthread.dylib       	0x00000001e4d100e0 0x1e4d0e000 + 8416
2   libsystem_c.dylib             	0x00000001e4becd78 0x1e4b95000 + 359800
3   libc++abi.dylib               	0x00000001e42b4f78 0x1e42b3000 + 8056
4   libc++abi.dylib               	0x00000001e42b5120 0x1e42b3000 + 8480
5   libobjc.A.dylib               	0x00000001e42cde48 0x1e42c7000 + 28232
6   libc++abi.dylib               	0x00000001e42c10fc 0x1e42b3000 + 57596
7   libc++abi.dylib               	0x00000001e42c1188 0x1e42b3000 + 57736
8   libdispatch.dylib             	0x00000001e4b38498 0x1e4ad7000 + 398488
9   libdispatch.dylib             	0x00000001e4adbe58 0x1e4ad7000 + 20056
10  FrontBoardServices            	0x00000001e7b6e640 0x1e7b23000 + 308800
11  FrontBoardServices            	0x00000001e7b6e2cc 0x1e7b23000 + 307916
12  FrontBoardServices            	0x00000001e7b6e8e8 0x1e7b23000 + 309480
13  CoreFoundation                	0x00000001e508e5b8 0x1e4fe2000 + 705976
14  CoreFoundation                	0x00000001e508e538 0x1e4fe2000 + 705848
15  CoreFoundation                	0x00000001e508de1c 0x1e4fe2000 + 704028
16  CoreFoundation                	0x00000001e5088ce8 0x1e4fe2000 + 683240
17  CoreFoundation                	0x00000001e50885b8 0x1e4fe2000 + 681400
18  GraphicsServices              	0x00000001e72fc584 0x1e72f1000 + 46468
19  UIKitCore                     	0x0000000211f04bc8 0x211621000 + 9321416
20  Dsym_demo                     	0x0000000100fde814 0x100fd8000 + 26644
21  libdyld.dylib                 	0x00000001e4b48b94 0x1e4b48000 + 2964

Thread 1:
0   libsystem_pthread.dylib       	0x00000001e4d1ccfc 0x1e4d0e000 + 60668

Thread 2:
0   libsystem_pthread.dylib       	0x00000001e4d1ccfc 0x1e4d0e000 + 60668

Thread 3:
0   libsystem_pthread.dylib       	0x00000001e4d1ccfc 0x1e4d0e000 + 60668

Thread 4:
0   libsystem_pthread.dylib       	0x00000001e4d1ccfc 0x1e4d0e000 + 60668

...

Thread 0 crashed with ARM Thread State (64-bit):
    x0: 0x0000000000000000   x1: 0x0000000000000000   x2: 0x0000000000000000   x3: 0x0000000280883937
    x4: 0x00000001e42c4b81   x5: 0x000000016ee26560   x6: 0x000000000000006e   x7: 0xffffffffffffffec
    x8: 0x0000000000000800   x9: 0x00000001e4d148d8  x10: 0x00000001e4d0ff64  x11: 0x0000000000000003
   x12: 0x0000000000000072  x13: 0x0000000000000000  x14: 0x0000000000000010  x15: 0x000000000000000d
   x16: 0x0000000000000148  x17: 0x0000000000000000  x18: 0x0000000000000000  x19: 0x0000000000000006
   x20: 0x00000001012fab80  x21: 0x000000016ee26560  x22: 0x0000000000000303  x23: 0x00000001012fac60
   x24: 0x0000000000001d03  x25: 0x0000000000002503  x26: 0x0000000000000000  x27: 0x0000000000000000
   x28: 0x0000000281aac508   fp: 0x000000016ee264c0   lr: 0x00000001e4d100e0
    sp: 0x000000016ee26490   pc: 0x00000001e4c95104 cpsr: 0x00000000

Binary Images:
0x100fd8000 - 0x100fdffff Dsym_demo arm64  <e0cffa85f688305eb85d4ed9a320f8c7> /var/containers/Bundle/Application/112179B6-F1BB-4C6A-9097-7C39AD59A84C/Dsym_demo.app/Dsym_demo
0x101200000 - 0x10120bfff libobjc-trampolines.dylib arm64  <62573b18de65349aa12efd1340ea3ce3> /usr/lib/libobjc-trampolines.dylib
0x101260000 - 0x1012c3fff dyld arm64  <8f53711b46f3359ea4d6479c85ada9ce> /usr/lib/dyld

...

EOF

```


很多字段都是见面知意的，介绍下几个重要的

1.  `Incident Identifier` : 对奔溃文件的唯一key值；
2.  `CrashReporter Key`: 设备号的唯一值，同一个是手机的所有`.ips`文件中的这个值是相同的；
3.  `Identifier` : `bundleID`;
4.  `Thread xx name: xxx` : 奔溃线程的编号和名称；
5.  `Thread 0 crashed with ARM Thread State (64-bit)` : 当时的寄存器的值；
6.  `Binary Images` : 模块地址

单独解释下这个`0x100fd8000 - 0x100fdffff Dsym_demo arm64  <e0cffa85f688305eb85d4ed9a320f8c7> /var/containers/Bundle/Application/112179B6-F1BB-4C6A-9097-7C39AD59A84C/Dsym_demo.app/Dsym_demo`

1. `0x100fd8000 - 0x100fdffff`: 这个是`Dsym_demo`[`ASLR`](https://fangshufeng.com/2018/12/12/【iOS逆向】-debugserver/#查看ASLR)后的开始和结束地址,通过该地址可以计算出函数在安装包中的地址；
2. `Dsym_demo`: 应用的名称
3. `arm64`: 应用的架构
4. `e0cffa85f688305eb85d4ed9a320f8c7`: `uuid`的值，这个用来和`dysm`一一对应；
5. `/var/containers/Bundle/Application/112179B6-F1BB-4C6A-9097-7C39AD59A84C/Dsym_demo.app/Dsym_demo`:应用的安装路径。


### 3.2 如何保证`dsym`和`crash`文件的一致性

`dsym`和`crash`是通过`UUID`来保持一致的，在`3.1`中已经介绍了如何查看`uuid`，下面介绍下如何在`dsym`中获取`UUID`的值

#### `otool`

```mm
➜  Crash otool -l Dsym_demo.app.DSYM/Contents/Resources/DWARF/Dsym_demo | grep uuid
    uuid E0CFFA85-F688-305E-B85D-4ED9A320F8C7
    
```

#### `dwarfdump`

```mm
➜  Crash dwarfdump --uuid Dsym_demo.app.DSYM
UUID: E0CFFA85-F688-305E-B85D-4ED9A320F8C7 (arm64) Dsym_demo.app.DSYM/Contents/Resources/DWARF/Dsym_demo

```

#### `MachOView`

![15477069257018.jpg](https://upload-images.jianshu.io/upload_images/3279997-421e3e1485d46ff7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



只要文件`crash`和`dsym`的`uuid`相等2者就是一直的。


### 3.3 获取崩溃信息在二进制包中的地址

现在已经找到了`crash`和`dsym`了，下面来分析下如何通过地址来找到具体的崩溃点

拿下面这个`Dsym_demo`的`0x0000000100fde814`地址为例，看看到底是哪个函数的地址


```mm
19  UIKitCore                     	0x0000000211f04bc8 0x211621000 + 9321416
20  Dsym_demo                     	0x0000000100fde814 0x100fd8000 + 26644
21  libdyld.dylib                 	0x00000001e4b48b94 0x1e4b48000 + 2964

```

先介绍下每列的含义

1. 第一列：是序号
2. 第二列：是实际运行的时候崩溃的地址
3. 第三列+第四列的值就是第二列

我之前的写的介绍[`ASLR`](https://fangshufeng.com/2018/12/12/【iOS逆向】-debugserver/#查看ASLR)的文章中有介绍，对于`arm64`结构如果没有`ASLR`的话开始地址是`0x100000000`,而从`3.1`的介绍中可以知道，运行内存的开始地址是`0x100fd8000`，所以偏移了`0xFD8000 = 0x100fd8000 - 0x100000000`；

所以`0x0000000100fde814`在二进制包中的地址为`0x100006814 = 0x0000000100fde814 - 0xFD8000`；

### 3.4 通过崩溃地址在二进制包中找到对应的函数

使用`hopper`来查看`dsym`，搜索我们`3.3`计算出的地址`0x100006814`

![15477079830293.jpg](https://upload-images.jianshu.io/upload_images/3279997-7488715ed5afd8e2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个是在方法`main`下面的

![15477080081473.jpg](https://upload-images.jianshu.io/upload_images/3279997-a5940e36bbc49857.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以找到了`0x0000000100fde814`是崩溃在`main`函数里面的。

### 3.5 简单介绍下`DSYM`文件

![15477081778279.jpg](https://upload-images.jianshu.io/upload_images/3279997-a3fd81060b080bce.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. `Symbols`存放着方法的存在哪里，开始地址是什么，比如`-[ViewController viewDidLoad]`方法就是存放在`_TEXT`segment中的`__test`section中，开始地址为`0x00000001000066A4`,偏移量为`0`，而`-[ViewController testFunc]`的偏移量为`104`，也就是`0x00000001000066A4 + 0x68（104） = 0x10000670C`，所以`-[ViewController testFunc]`的开始地址为`0x10000670C`，而`-[ViewController viewDidLoad]`地址范围就是`0~0x10000670b`


`section64(_DWARF,__debug_line)`存放在具体的行号信息，`DWARF`是一种存储格式，行号就是以这种形式存储的。

![15477089651405.jpg](https://upload-images.jianshu.io/upload_images/3279997-0a6b39ee92f7a335.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


要想读取出具体的行号信息有以下方式

#### `dwarfdump`


```mm
➜  Crash dwarfdump --arch arm64 Dsym_demo.app.DSYM --lookup 0x100006814
----------------------------------------------------------------------
 File: Dsym_demo.app.DSYM/Contents/Resources/DWARF/Dsym_demo (arm64)
----------------------------------------------------------------------
Looking up address: 0x0000000100006814 in .debug_info... found!

0x000400c2: Compile Unit: length = 0x0000007d  version = 0x0004  abbr_offset = 0x00000000  addr_size = 0x08  (next CU at 0x00040143)

0x000400cd: TAG_compile_unit [120] *
             AT_producer( "Apple LLVM version 10.0.0 (clang-1000.11.45.5)" )
             AT_language( DW_LANG_ObjC )
             AT_name( "/Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo/main.m" )
             AT_stmt_list( 0x00009287 )
             AT_comp_dir( "/Users/fangshufeng/Desktop/demo_project/Dsym_demo" )
             AT_APPLE_major_runtime_vers( 0x02 )
             AT_low_pc( 0x0000000100006798 )
             AT_high_pc( 0x000000a4 )

0x000400f4:     TAG_subprogram [125] *
                 AT_low_pc( 0x0000000100006798 )
                 AT_high_pc( 0x000000a4 )
                 AT_frame_base( reg29 )
                 AT_name( "main" )
                 AT_decl_file( "/Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo/main.m" )
                 AT_decl_line( 12 )
                 AT_prototyped( true )
                 AT_type( {0x0004012a} ( int ) )
                 AT_external( true )
Line table dir : '/Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo'
Line table file: 'main.m' line 14, column 9 with start address 0x0000000100006814

Looking up address: 0x0000000100006814 in .debug_frame... not found.

```

#### `atos`


```mm
➜  Crash atos -o Dsym_demo.app.DSYM/Contents/Resources/DWARF/Dsym_demo -arch arm64 0x100006814 // 这个地址是真实的在二进制安装包中的地址

main (in Dsym_demo) (main.m:14)

==还可以使用函数的真实地址==

➜  Crash atos -o Dsym_demo.app.DSYM/Contents/Resources/DWARF/Dsym_demo -arch arm64 -l 0x100fd8000 0x0000000100fde814 // 前面的`0x100fd8000`是`Binary Images`起始地址，0x0000000100fde814是栈地址
main (in Dsym_demo) (main.m:14)

```


**注意：上面的`arm64`需要自己选择对应的架构**


## 四、符号化`DSYM`手段

上面分析了如何分析一个`DSYM`，知道不管是命令还是图形工具背后的原理，其实只要算出地址和行数就搞定了，自己都可以写脚本或者工具了，下面就开始介绍轮子了。

将之前的`crash`和`.dsym`文件放到任意文件夹下


```
.
├── Dsym_demo-2019-01-16-180748.ips
├── Dsym_demo.app.DSYM
│   └── Contents
│       ├── Info.plist
│       └── Resources
│           └── DWARF
│               └── Dsym_demo

```

### 方式一 `symbolicatecrash`

`symbolicatecrash`这个命令是`xcode`安装包内的，先看下这个工具的放在哪的

```
➜  ~ find /Applications/Xcode.app -name 'symbolicatecrash'
/Applications/Xcode.app/Contents/Developer/Platforms/WatchSimulator.platform/Developer/Library/PrivateFrameworks/DVTFoundation.framework/symbolicatecrash
/Applications/Xcode.app/Contents/Developer/Platforms/AppleTVSimulator.platform/Developer/Library/PrivateFrameworks/DVTFoundation.framework/symbolicatecrash
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/Library/PrivateFrameworks/DVTFoundation.framework/symbolicatecrash
/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash

```

有很多个，我这里用的是`/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash`

复制到`Crash`文件夹下

```
cp /Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash ~/Desktop/Crash
```

使用


```
➜  Crash ./symbolicatecrash Dsym_demo-2019-01-16-180748.ips Dsym_demo.app.DSYM > tmp.crash
Error: "DEVELOPER_DIR" is not defined at ./symbolicatecrash line 69.

发现少了`DEVELOPER_DIR`环境变量
增加  export DEVELOPER_DIR="/Applications/XCode.app/Contents/Developer"
再次运行即可
```


![15476967966597.jpg](https://upload-images.jianshu.io/upload_images/3279997-0529b6d7f0a88def.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



甚至还可以这样


```
➜  Crash ./symbolicatecrash Dsym_demo-2019-01-16-180748.ips > tmp.crash

```

因为`symbolicatecrash`实际是用`uuid`做索引来查找`.dsym`文件，只要`spotlight`能搜到对应`uuid`的`.dsym`文件就会拿来解析

![15477122433283.jpg](https://upload-images.jianshu.io/upload_images/3279997-78140532f0ca4259.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`spotline`的命令版本就是`mdfind`


```
➜  Crash mdfind e0cffa85f688305eb85d4ed9a320f8c7 | grep DSYM
~/Desktop/Crash/Dsym_demo.app.DSYM

```

还是蛮好用的

### 方式二 `dwarfdump`

分两步，获取奔溃的方法是哪个


```
dwarfdump -e --debug-info YourPath/YourApp.dSYM/Contents/Resources/DWARF > info-e.txt
```

这里的内容完全可以描述一个函数

```
0x0004005f:     function [122] *
                low pc( 0x000000010000670c ) // 函数的起始地址
                high pc( 0x0000008c ) // 相对于起始地址的偏移量可以算出函数的结束地址
                frame base( reg29 )
                object pointer( {0x00040078} )
                name( "-[ViewController testFunc]" ) // 函数名称
                decl file( "/Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo/ViewController.m" ) // 函数位于哪个文件
                decl line( 22 ) // 行数定义域行文件的行数
                prototyped( true )

0x00040078:         formal parameter [123]  
                    location( fbreg -8 )
                    name( "self" ) // 函数参数
                    type( {0x0004009f} ( const ViewController* ) )// 参数类型
                    artificial( true )

0x00040084:         formal parameter [123]  
                    location( fbreg -16 )
                    name( "_cmd" ) 
                    type( {0x000400a9} ( SEL ) )
                    artificial( true )

0x00040090:         variable [124]  // 函数内部的变量 
                    location( breg31 +24 )
                    name( "a" ) // 变量名称
                    decl file( "/Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo/ViewController.m" )
                    decl line( 24 ) // 变量位于的行数
                    type( {0x000400bc} ( NSArray* ) ) // 变量的类型
                    
```

这里可以定位到具体的方式名称`testFunc`，接下来要找出崩溃发生在改方法的那一行


```
dwarfdump -e --debug-line Dsym_demo.app.DSYM > line-e.txt

```


```
0x00000001000066a4     17 /Users/fangshufeng/Desktop/demo_project/Dsym_demo/Dsym_demo/ViewController.m
0x00000001000066cc     18
0x00000001000066f0     19
0x00000001000066f4     19
0x0000000100006700     20
0x000000010000670c     22
0x0000000100006720     23
0x0000000100006744     24
0x0000000100006748     24
0x000000010000675c     25
0x000000010000677c     26
0x0000000100006798     26

```

这样就可以找到行数


当然你也可以一步到位上面`3.5`介绍过了。

看来`.dsym`中包含的信息可真多啊，这个和`symbolicatecrash`比起来对于符号化来说感觉差了些，但是如果是分析的话还是很好用的。


### 方式三 `atos`

这个在`3.5`也介绍过了，好处是可以不用手动计算地址

**就符号化而言个人最喜欢的还是`symbolicatecrash`**

### 方式四、`Xcode`自动符号化

脚本的话一般是做一些自动化的时候用的，最简单的还是`xcode`的图像工具，没有`.dsym`的可以让别人发给你就好了。

![15477140853263.jpg](https://upload-images.jianshu.io/upload_images/3279997-a1c368911434f9e7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




[demo地址](https://github.com/fangshufeng/demo_project/tree/master/Dsym_demo)


















