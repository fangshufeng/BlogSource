---
title: 【iOS逆向】-- theos的安装与使用
date: 2018-12-08 14:14:32
tags:
    iOS逆向
---

[TOC]

## 一、简介

>Theos是一个越狱开发工具包，由iOS越狱界知名 人士Dustin Howett（@DHowett）开发并分享到
>GitHub上。Theos与其他越狱开发工具相比，最大的 特点就是简单：下载安装简单、Logos语法简单、编译发布简单，可以让使用者把精力都放在开发工作上 去。


<!-- more -->

## 二、安装ldid

ldid是专门用来签名iOS可执行文件的工具，用以 在越狱iOS中取代Xcode自带的codesign。

通过`Homebrew`安装

```
brew install ldid #如果没有brew 执行下这个命令 /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## 三、安装dpkg

`deb`是越狱开发安装包的标准格式，`dpkg`是 一个用于操作deb文件的工具，有了这个工具，`Theos` 才能正确地把工程打包成为`deb`文件。


```
brew install dpkg

```

## 四、安装THEOS和配置


配置好环境变量


```
echo "export THEOS=~/theos" >> ~/.bash_profile
```

```
git clone --recursive https://github.com/theos/theos.git $THEOS

```


现在敲下面的指令就有提示了

```
➜  ~ ~/theos/bin/nic.pl
```

要想直接敲`nic.pl`，将`export PATH=~/theos/bin:$PATH`加到`~/.bash_profile`中即可


### 4.1 新建项目

找到你想要放的目录，执行`nic.pl`

```
➜  temp2 nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/activator_event
  [2.] iphone/application_modern
  [3.] iphone/application_swift
  [4.] iphone/cydget
  [5.] iphone/flipswitch_switch
  [6.] iphone/framework
  [7.] iphone/ios7_notification_center_widget
  [8.] iphone/library
  [9.] iphone/notification_center_widget
  [10.] iphone/preference_bundle_modern
  [11.] iphone/tool
  [12.] iphone/tool_swift
  [13.] iphone/tweak
  [14.] iphone/xpc_service
Choose a Template (required): 13
Project Name (required): testtheos
Package Name [com.yourcompany.testtheos]: com.theos.developer
Author/Maintainer Name [fangshufeng]:
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.app.springboard
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]:
Instantiating iphone/tweak in testtheos/...
Done.

```

看到一共有14种模板，逆向中接触最多的是`tweak`，也就是`13`,其他更多的模板请[参照](http://bbs.iosre.com/)


1. 我们选择13（tweak）模板；
2. project Name : 工程名称；
3. package Name : deb的包名；
4. Author Name : 
5. [iphone/tweak] MobileSubstrate Bundle filter: 你要破解的app的`bundleid`;
6. [iphone/tweak] List of applications to terminate upon installation : 输入tweak安装完成后需要重启的应用，以进 程名表示,一般直接回车。


完成以后目录结构如下

```
── testtheos
    ├── Makefile
    ├── Tweak.xm
    ├── control
    └── testtheos.plist
```

#### 4.1.1 Makefile


内容如下

```
include $(THEOS)/makefiles/common.mk

TWEAK_NAME = testtheos
testtheos_FILES = Tweak.xm

include $(THEOS_MAKE_PATH)/tweak.mk

after-install::
	install.exec "killall -9 SpringBoard"

```

##### 4.1.1.1 include $(THEOS)/makefiles/common.mk

echo下这个路径如下

```
~/theos/makefiles/common.mk
```

这里封装了很多内置的命令，一般不做修改



##### 4.1.1.2 TWEAK_NAME = testtheos

插件的名称,之前新建工程的时候写的

##### 4.1.1.3 testtheos_FILES = Tweak.xm

自己写的hook文件，多个话以空格隔开

##### 4.1.1.4 include $(THEOS_MAKE_PATH)/tweak.mk

根据不同的Theos工程类型，通过include命令指 定不同的.mk文件；在逆向工程初级阶段，我们开发 的一般是Application、Tweak和Tool三种类型的程序， 它们对应的.mk文件分别是application.mk、tweak.mk 和tool.mk，可以按需更改。

##### 4.1.1.5 after-install::install.exec "killall -9 SpringBoard"

插件安装成功后重启`SpringBoard`


关于`Makefile`更多[参照](http://www.gnu.org/software/make/manual/html_node/Makefiles.html)


#### 4.1.2 Tweak.xm


```
/* How to Hook with Logos
Hooks are written with syntax similar to that of an Objective-C @implementation.
You don't need to #include <substrate.h>, it will be done automatically, as will
the generation of a class list and an automatic constructor.

%hook ClassName

// Hooking a class method
+ (id)sharedInstance {
	return %orig;
}

// Hooking an instance method with an argument.
- (void)messageName:(int)argument {
	%log; // Write a message about this call, including its class, name and arguments, to the system log.

	%orig; // Call through to the original function with its original arguments.
	%orig(nil); // Call through to the original function with a custom argument.

	// If you use %orig(), you MUST supply all arguments (except for self and _cmd, the automatically generated ones.)
}

// Hooking an instance method with no arguments.
- (id)noArguments {
	%log;
	id awesome = %orig;
	[awesome doSomethingElse];

	return awesome;
}

// Always make sure you clean up after yourself; Not doing so could have grave consequences!
%end
*/


```

这个文件就是用来写hook代码，使用的语法是`Logos`语法，关于`logo`语法后面会写个详细介绍，也可自行[学习](http://iphonedevwiki.net/index.php/Logos)

#### 4.1.3 control


```
Package: com.theos.developer
Name: testtheos 
Depends: mobilesubstrate
Version: 0.0.1
Architecture: iphoneos-arm
Description: An awesome MobileSubstrate tweak!
Maintainer: fangshufeng
Author: fangshufeng
Section: Tweaks

```

这里没啥要介绍的了

#### 4.1.4 testtheos.plist


```
{ Filter = { Bundles = ( "com.app.springboard" ); }; }

```

这里就是hook的app的bundleid

### 4.2 编译、打包、安装


#### 4.2.1 编译 make


```
➜  testtheos make
> Making all for tweak testtheos…
==> Preprocessing Tweak.xm…
==> Compiling Tweak.xm (armv7)…
==> Linking tweak testtheos (armv7)…
==> Generating debug symbols for testtheos…
==> Signing testtheos…
➜  testtheos

```


可以看出`make`指令完成了`预处理`、`编译`、`链接`、`签名`操作,同时多了个`.theos`文件夹


```
➜  testtheos la
total 32
drwxr-xr-x  4 fangshufeng  staff   128B Dec  8 11:05 .theos
-rw-r--r--  1 fangshufeng  staff   181B Dec  8 11:03 Makefile
-rw-r--r--  1 fangshufeng  staff   1.0K Dec  8 11:03 Tweak.xm
-rw-r--r--  1 fangshufeng  staff   219B Dec  8 11:03 control
drwxr-xr-x  3 fangshufeng  staff    96B Dec  8 11:05 obj
-rw-r--r--  1 fangshufeng  staff    57B Dec  8 11:03 testtheos.plist

```

进到`.theos`文件下查看


```
➜  testtheos ls -l ./.theos/obj/debug/
total 200
drwxr-xr-x  6 fangshufeng  staff    192 Dec  8 11:05 arm64
drwxr-xr-x  6 fangshufeng  staff    192 Dec  8 11:05 armv7
-rwxr-xr-x  1 fangshufeng  staff  98704 Dec  8 11:05 testtheos.dylib

```

有个`testtheos.dylib`，这个动态库是插件最核心的部分了。

#### 4.2.2 打包 make package


```
➜  testtheos make package
> Making all for tweak testtheos…
make[2]: Nothing to be done for `internal-library-compile'.
> Making stage for tweak testtheos…
dpkg-deb: building package 'com.theos.developer' in './packages/com.theos.developer_0.0.1-1+debug_iphoneos-arm.deb'.

```

这个步骤就是使用`dpkg`将动态库打包成`deb`包



```
➜  testtheos la
total 32
drwxr-xr-x  8 fangshufeng  staff   256B Dec  8 11:13 .theos
-rw-r--r--  1 fangshufeng  staff   181B Dec  8 11:03 Makefile
-rw-r--r--  1 fangshufeng  staff   1.0K Dec  8 11:03 Tweak.xm
-rw-r--r--  1 fangshufeng  staff   219B Dec  8 11:03 control
drwxr-xr-x  3 fangshufeng  staff    96B Dec  8 11:05 obj
drwxr-xr-x  3 fangshufeng  staff    96B Dec  8 11:13 packages
-rw-r--r--  1 fangshufeng  staff    57B Dec  8 11:03 testtheos.plist

```

发现多了一个`packages`


```
➜  testtheos la ./packages
total 8
-rw-r--r--  1 fangshufeng  staff   2.3K Dec  8 11:13 com.theos.developer_0.0.1-1+debug_iphoneos-arm.deb
➜  testtheos

```

`com.theos.developer_0.0.1-1+debug_iphoneos-arm.deb`就是可以安装到手机上的`deb`最终安装包。


执行完了`make package`除了生成了`packages`文件夹之外，在`.theos`文件下还多了一个`_`文件夹


```
➜  testtheos ls -l ./.theos
total 8
drwxr-xr-x  4 fangshufeng  staff  128 Dec  8 11:13 _
-rw-r--r--  1 fangshufeng  staff    0 Dec  8 11:04 build_session
-rw-r--r--  1 fangshufeng  staff    0 Dec  8 11:13 fakeroot
-rw-r--r--  1 fangshufeng  staff   62 Dec  8 11:13 last_package
drwxr-xr-x  3 fangshufeng  staff   96 Dec  8 11:05 obj
drwxr-xr-x  3 fangshufeng  staff   96 Dec  8 11:13 packages

```

看下这个文件夹中的内容


```
➜  _ tree
.
├── DEBIAN
│   └── control
└── Library
    └── MobileSubstrate
        └── DynamicLibraries
            ├── testtheos.dylib
            └── testtheos.plist

4 directories, 3 files
➜  _

```


`control`内容和我们一开始的`control`差不多，`Library`


```
└── Library
    └── MobileSubstrate
        └── DynamicLibraries
            ├── testtheos.dylib
            └── testtheos.plist


```

这个到时就是到手机上的安装目录


我们对比下生成的`deb`包的内容 


```
➜  testtheos dpkg -c packages/com.theos.developer_0.0.1-1+debug_iphoneos-arm.deb
drwxr-xr-x fangshufeng/staff 0 2018-12-08 11:13 ./
drwxr-xr-x fangshufeng/staff 0 2018-12-08 11:13 ./Library/
drwxr-xr-x fangshufeng/staff 0 2018-12-08 11:13 ./Library/MobileSubstrate/
drwxr-xr-x fangshufeng/staff 0 2018-12-08 11:13 ./Library/MobileSubstrate/DynamicLibraries/
-rwxr-xr-x fangshufeng/staff 98704 2018-12-08 11:13 ./Library/MobileSubstrate/DynamicLibraries/testtheos.dylib
-rw-r--r-- fangshufeng/staff    57 2018-12-08 11:13 ./Library/MobileSubstrate/DynamicLibraries/testtheos.plist

```

#### 4.2.3 安装 


有了第二步的`deb`安装包，有2种安装方式可以装到手机上


##### 4.2.3.1 图形界面

将`deb`安装包通过`iFunBox`复制到手机上，然后通过`iFlie`安装，需要的话重启手机即可。


##### 4.2.3.2 命令的方式make install

图形界面的方式的话人机交互太多，不方便插件开发，下面介绍直接通过命令完成安装

需要现在文件`Makefile`文件的顶部添加手机的ip地址

```
export THEOS_DEVICE_IP=iosip

```

然后再执行`make install`即可


## 五、一个简单的例子

前面一直在将理论，下面来写一个简单的例子来熟悉一下整个过程

###  新建一个iOS工程

我们新建一个空的`iOS`工程用来测试我们的插件

目录结构如下

```
➜  sign tree
.
├── sign
│   ├── AppDelegate.h
│   ├── AppDelegate.m
│   ├── Assets.xcassets
│   │   ├── AppIcon.appiconset
│   │   │   └── Contents.json
│   │   └── Contents.json
│   ├── Base.lproj
│   │   ├── LaunchScreen.storyboard
│   │   └── Main.storyboard
│   ├── Info.plist
│   ├── ViewController.h
│   ├── ViewController.m
│   └── main.m
├── sign.xcodeproj
│   ├── project.pbxproj
│   ├── project.xcworkspace
│   │   ├── contents.xcworkspacedata
│   │   ├── xcshareddata
│   │   │   └── IDEWorkspaceChecks.plist
│   │   └── xcuserdata
│   │       └── fangshufeng.xcuserdatad
│   │           └── UserInterfaceState.xcuserstate
│   └── xcuserdata
│       └── fangshufeng.xcuserdatad
│           └── xcschemes
│               └── xcschememanagement.plist
└── signUITests
    ├── Info.plist
    └── signUITests.m

13 directories, 17 files

```


在`ViewController`上写一个`label`作为展示

```
- (void)viewDidLoad {
    [super viewDidLoad];
    UILabel *num = [[UILabel alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
    num.text = @"30";
    [self.view addSubview:num];

}

```

然后将工程跑到越用手机上备用。

###  新建一个`Tweak`工程

control内容如下


```
Package: com.sign.test
Name: signtest
Depends: mobilesubstrate
Version: 0.0.1
Architecture: iphoneos-arm
Description: An awesome MobileSubstrate tweak!
Maintainer: fangshufeng
Author: fangshufeng
Section: Tweaks

```

由于我们的测试工程的`bundleid`是`com.sign.test`


###  编写hook代码 


现在我们想做一个插件，当测试app打开的时候就弹一个`alert`，那么`hook`代码如下

```
%hook ViewController

- (void)viewDidLoad {
	
	%orig;
	
	    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        UIAlertView *aler = [[UIAlertView
                              alloc] initWithTitle:@"弹框" message:nil delegate:nil cancelButtonTitle:@"取消" otherButtonTitles:nil, nil];
        [aler show];
    });

}

%end

```

###  编译打包

执行`make`和`make package`命令（`make package` 已经包含了`make`命令了，所以直接执行`make package`也是可以的）

执行完成以后会有个`com.sign.test_0.0.1-1+debug_iphoneos-arm.deb`



###  安装

#### 图形界面方式

通过`iFunbox`将`com.sign.test_0.0.1-1+debug_iphoneos-arm.deb`放到iPhone的根目录下，通过 `iFile`查看


![15442482115467.jpg](https://upload-images.jianshu.io/upload_images/3279997-9b1ba8ddab5830af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/500)


点击安装，如果安装成功，打开`Cydia`可以看到，如果看不到，重启一下手机

![15442483443840.jpg](https://upload-images.jianshu.io/upload_images/3279997-6548886efcffa921.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)



点击查看安装的路径

![15442483716762.jpg](https://upload-images.jianshu.io/upload_images/3279997-af050555eb29b97b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


正好使我们之前说的路径，通过`iFunbox`也可以找到该动态库

![15442484389650.jpg](https://upload-images.jianshu.io/upload_images/3279997-b853c8df44dc607d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


此时打开我们的测试程序

![15442485059481.jpg](https://upload-images.jianshu.io/upload_images/3279997-9bb5a1fead8789c1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/h/500)

插件已经生效了


#### 命令方式

如果你已经执行了图形的操作，先在`Cydia`中卸载该插件。

先在`Makefile`文件的顶部增加`export THEOS_DEVICE_IP=192.168.1.102`，地址换成你自己手机的地址

然后执行`make install`命令即可，发现和图形界面是一样的效果

## 六、常见报错

### 6.1 [error] Cowardly refusing to make a project inside $THEOS

有可能是你执行`git clone --recursive https://github.com/theos/theos.git $THEOS`命令 的时候报错了 ，有些代码没下载下来，请检查git clone信息


### 6.2 如果安装的插件一直到手机的根目录（针对命令行安装）不在`/Library/MobileSubstrate/DynamicLibraries/`下

则 将文件$THEOS/makefiles/package/deb.mk 中 53行附近，由

```
$(ECHO_NOTHING)COPYFILE_DISABLE=1 $(FAKEROOT) -r $(_THEOS_PLATFORM_DPKG_DEB) -Z$(_THEOS_PLATFORM_DPKG_DEB_COMPRESSION) -b "$(THEOS_STAGING_DIR)" "$(_THEOS_DEB_PACKAGE_FILENAME)"$(ECHO_END)

```

替换为


```
$(ECHO_NOTHING)COPYFILE_DISABLE=1 $(FAKEROOT) -r dpkg-deb -Zgzip -b "$(THEOS_STAGING_DIR)" "$(_THEOS_DEB_PACKAGE_FILENAME)" $(STDERR_NULL_REDIRECT)$(ECHO_END)
```

具体参考[链接](http://www.iosre.com/t/theos/12762/18)


到这应该感受到了插件的强大了，接下来只要学会分析app和熟悉logo语法就可以写更多好用的插件了。











