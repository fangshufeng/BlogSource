---
title: 【iOS逆向】- Cycript
date: 2018-11-07 21:52:45
tags:
    iOS逆向
---




## 简介

>Cycript allows developers to explore and modify running applications on either iOS or Mac OS X using a hybrid of Objective-C++ and JavaScript syntax through an interactive console that features syntax highlighting and tab completion.
(It also runs standalone on Android and Linux and provides access to Java, but without injection.)

该语言可以用来动态调试我们的程序，查看内存中的状态，[官网地址](http://www.cycript.org/)，点开手机的`Cydia`可以看到这个插件

<!-- more -->

## 常见用法

### 开启/关闭调试


```
cycript -p 进程名称

或者

cycript -p 进程ID
```

比如我们想调试我们的桌面程序`SpringBoard`

![15415960608710.jpg](https://upload-images.jianshu.io/upload_images/3279997-7be33e7dc2757c11.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


退出: 键盘`Ctrl + D`


### 简单使用

(1). UIApp / [UIApplication sharedApplication]


![15415964138396.jpg](https://upload-images.jianshu.io/upload_images/3279997-91311f55433f5140.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`UIApp`和`[UIApplication sharedApplication]`是等价的

(2). 获取keyWindow 和 rootViewController

![15415965250824.jpg](https://upload-images.jianshu.io/upload_images/3279997-a71554bdbdc3e1ed.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(3). 通过地址获取对象

语法：

```
    #对象的地址
```

![15415966170483.jpg](https://upload-images.jianshu.io/upload_images/3279997-b451f3bba135467a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


(4). 获取已经加载的所有类


```
ObjectiveC .classes
```

内容太长不截图了


(5).获取对象的所有成员变量

语法

```
* 对象
```

![15415973015883.jpg](https://upload-images.jianshu.io/upload_images/3279997-01285e6f8f0df23f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(6). 获取当前view中某个类型的子类型


```
choose(类型)
```

![15415975280306.jpg](https://upload-images.jianshu.io/upload_images/3279997-7ed3baead1f7ceaa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


(7). 获取当前展示的界面的所有view的子控件


```
view.recursiveDescription().toString()
```


![15415976387392.jpg](https://upload-images.jianshu.io/upload_images/3279997-1a671bf516347fde.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



每次都敲这些东西很麻烦，你可以定义方法，写到文件.cy结尾放到手机的`/usr/lib/cycript0.9`下

![15415977934401.jpg](https://upload-images.jianshu.io/upload_images/3279997-254c14f09f10e9c8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
//根控制器
LDRootVC = () => UIApp.keyWindow.rootViewController;

LDKeyWindow = () => UIApp.keyWindow;


LDBundleId = [NSBundle mainBundle].bundleIdentifier;

// 沙盒目录
LDDocumentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES)[0];


// 沙盒目录
LDCachePath = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES)[0];

// 获取当前展示的viewController
LDCurrentViewController = _ => _innerCurrentVC(LDRootVC());

const _innerCurrentVC = (fromViewController) =>  {

if ([fromViewController isKindOfClass:[UINavigationController class]]) {
    return _innerCurrentVC(fromViewController.viewControllers.lastObject());
} else if([fromViewController isKindOfClass:[UITabBarController class]]) {
    return _innerCurrentVC(fromViewController.selectedViewController);
} else if(fromViewController.presentedViewController) {
    return _innerCurrentVC(fromViewController.presentedViewController);
} else {
    return fromViewController;

};
}

//获取某个类的所有子类
LDSubClassWithView = (classView) => choose(classView);


//获取某个view的所有子view
LDSubViews = (view) => view.recursiveDescription() .description;


//对外输出的接口提示使用者用 没有具体含义
exports.LDInterface = [
'LDRootVC',
'LDkeyWindow',
'LDBundleId',
'LDDocumentPath',
'LDCachePath',
'LDCurrentViewController',
'LDSubClassWithView'
];

```

用法如下

![15415980079446.jpg](https://upload-images.jianshu.io/upload_images/3279997-e1a796a66befdda2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这样就不用每次都写了，不使用该文件也是可以的，无非就是麻烦些而已






















