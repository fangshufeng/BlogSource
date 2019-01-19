---
layout: post
title: "通过命令行控制模拟器--simctl"
date: 2018-07-13 23:09:00 +0800
comments: true
categories: 
tag:
    Objective-C
---




要想实现命令行控制模拟器需要用的到命令是`simctl`: `Simulators Control` 看这命令的意思像是这两个单词的缩写，搭载`xcrun`命令，可以通过这个命令实现以下功能

1. 创建、删除、开启 模拟器
2. 添加图片到模拟器
3. 安装、卸载、打开、关闭APP

<!--more-->
### 帮助命令

```
xcrun simctl --help
```

### 查看当前有哪些模拟器

```
xcrun simctl list
```

截取部分日志如下

```
== Device Types ==
iPhone 4s (com.apple.CoreSimulator.SimDeviceType.iPhone-4s)
iPhone 5 (com.apple.CoreSimulator.SimDeviceType.iPhone-5)

== Runtimes ==
iOS 8.1 (8.1 - 12B411) - com.apple.CoreSimulator.SimRuntime.iOS-8-1
iOS 9.0 (9.0 - 13A344) - com.apple.CoreSimulator.SimRuntime.iOS-9-0
iOS 10.0 (10.0 - 14A345) - com.apple.CoreSimulator.SimRuntime.iOS-10-0
iOS 11.4 (11.4 - 15F79) - com.apple.CoreSimulator.SimRuntime.iOS-11-4

== Devices ==
-- iOS 8.1 --
    iPhone 4s (134D6998-BB21-4E41-BF30-F2475B71518C) (Shutdown)
    iPhone 5 (C284C3E1-6297-452C-AADF-2AFA0D3D9931) (Shutdown)
    iPhone 5s (1417E9B3-FA40-42D3-BF04-F7DD469DDA7E) (Shutdown)
    iPhone 6 (A8BA264E-115F-4B64-85FF-63E1C9B8E4CC) (Shutdown)
    
-- iOS 10.0 --
iPhone 5 (6B088A91-F77C-4FA6-A6DA-0E4722B11DE5) (Shutdown)
iPhone 5s (38D06629-1F4B-46A2-A986-CBE28503E63F) (Shutdown)
iPhone 6 (598F3D6F-5439-4216-808D-D14857B6B68B) (Shutdown)
iPhone 6 Plus (B79E33B8-0944-4509-8DAA-15D00C3FF8E1) (Shutdown)
iPhone 6s (26A06303-C684-46F1-AF9B-9ECEB737B2AA) (Booted)

-- iOS 11.4 --
iPhone 5s (8B31401D-41BE-40F9-8ED1-F17C93C87F8A) (Shutdown)
iPhone 6 (12D46DEB-71A0-4009-A83D-C96A4AACBD3D) (Shutdown)
```
 
搭载查找命令`grep`可以查看正在使用的模拟器

```
➜  ~ xcrun simctl list | grep Booted
    iPhone 6s (26A06303-C684-46F1-AF9B-9ECEB737B2AA) (Booted)
    
```

### 添加一个模拟器

设备名称、设备类型、设备的操作系统、可以定位出一个设备信息，比如我们要增加一个名为`DemoDevice`、`iPhone 7`、`iOS 11.4`命令如下

```
➜  ~ xcrun simctl create DemoDevice com.apple.CoreSimulator.SimDeviceType.iPhone-7  com.apple.CoreSimulator.SimRuntime.iOS-11-4
4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0

```

创建成果的话会返回设备的id这里是`4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0`

`list`下

```
-- iOS 11.4 --
    iPhone 5s (8B31401D-41BE-40F9-8ED1-F17C93C87F8A) (Shutdown)
    iPhone 6 (12D46DEB-71A0-4009-A83D-C96A4AACBD3D) (Shutdown)
    iPhone 6 Plus (40F90FF9-D7F8-4D44-85BB-F69B97909CC3) (Shutdown)
    iPhone 6s (61DFD682-9E00-4120-9AAF-132D6F2A2DCD) (Shutdown)
    iPhone 6s Plus (648433A5-2CDE-4982-BF00-7F70EA20739F) (Shutdown)
    DemoDevice (4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0) (Shutdown)
    
```

也可以通过grep搜索一下

```
➜  ~ xcrun simctl list | grep  4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
    DemoDevice (4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0) (Shutdown)
➜  ~
```

### 开启一个模拟器

以刚才那个自己创建的为例子吧

```
➜  ~ xcrun simctl boot 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```
![15313910142713.jpg](https://upload-images.jianshu.io/upload_images/3279997-a7ab94ee35a70c5b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)


### 写个图片到刚才的模拟器中

桌面准备一张图片`google@2x.png`

```
➜  ~ xcrun simctl addmedia 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0 ~/Desktop/google@2x.png
```

打开相册会发现多了一张图片，就不截图了

### 重置模拟器

重置模拟器的命令

```
➜  ~ xcrun simctl erase 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```

如果你的模拟器已经开启的情况下可能会报下面的错误

```
An error was encountered processing the command (domain=com.apple.CoreSimulator.SimError, code=164):
Unable to erase contents and settings in current state: Booted
```
提示你的模拟器当前的状态是booted的，所有要先关掉模拟器


```
➜  ~ xcrun simctl shutdown 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```

再执行
```
➜  ~ xcrun simctl erase 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```

会发现之前的所有的修改都没了

如果发现设备卡住了，可以关掉再重新`boot`下

### 通过DerivedData来安装app

（1） 在桌面下新建一个名为`TestSimctl`的工程

（2）cd到`TestSimctl`目录下通过`build`命令来得到`DerivedData`路径
    
    
```
xcodebuild  build -scheme 'TestSimctl' CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO -sdk iphonesimulator11.4

```
会得到`~/Library/Developer/Xcode/DerivedData/TestSimctl-heyedthdtyuorceyxhlduzjcoqol/Build/Products/Debug-iphonesimulator/TestSimctl.app`
每个人的机器路径会有些不一样复制一下就好了

（2）通过刚才的`DerivedData`来安装TestSimctl应用


```
xcrun simctl install booted ~/Library/Developer/Xcode/DerivedData/TestSimctl-heyedthdtyuorceyxhlduzjcoqol/Build/Products/Debug-iphoneos/TestSimctl.app
```

![15314471990147.jpg](https://upload-images.jianshu.io/upload_images/3279997-a40a69869e89329c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

（3）通过bundleid来卸载刚才那个app，我这里是`com.TestSimctl`

```
xcrun simctl uninstall booted com.TestSimctl
```
发现哪个app不见了

（4）通过bundleid来启动刚才app, 先保证已经安装了app


```
xcrun simctl launch booted com.TestSimctl
```
会发现已经启动起来了

（4）通过bundleid来关闭已经打开的app

```
xcrun simctl terminate  booted com.TestSimctl
```
    
### 截图功能


```
➜  TestSimctl git:(master) ✗ xcrun simctl io booted screenshot test.png
Detected file type 'PNG' from extension
Wrote screenshot to: test.png

```

会发现当前目录下多了一张照片

```
└── test.png
```

### 日志功能


```
xcrun simctl spawn booted log stream --predicate 'processImagePath endswith "TestSimctl"'
```

改造下`viewcontroller`代码

```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"touchesBegan");
}
```

重新build后可以发现终端可以看到日志了


### 删除模拟器
先shutdown

```
xcrun simctl shutdown 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```
再删除


```
xcrun simctl delete 4F5A92A2-8241-4A8E-8A22-2E5A87FECCC0
```






