---
layout: post
title: "使用CocoaPods建立私有仓库"
date: 2017-03-14 18:33:10 +0800
comments: true
categories: 
tag:
    Objective-C
---



大致分为以下几步:

>1.本地建立一个索引库Spec Repo,映射到远程仓库(将来使用该仓库里面的.podspec文件定位到相应的代码)

>2.创建pod工程(实现具体的组件代码)

>3.生成spec文件

>4.向本地的Spec Repo提交spec文件

>5.pod新的文件

<!--more-->

###(一).本地创建索引库
官方的pod其实就是一个仓库里面放了很多开源的Spec Repo(关于如何创建cocoapods这里就不做说明了)假设你已经安装好了cocoapods  当cd 到~/.cocoapods/repos可发现如下截图那个master中就是官方的Spec Repo所在

![C127BB3E-7C72-480C-9E5F-59DCE4CAF72B.png](http://upload-images.jianshu.io/upload_images/3279997-d5c130a195791cfa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到这个目录之后我们要做的第一步就是在repos目录下建立属于自己的私有Spec Repo用来管理我们的代码,开始接下来的操作
1.在终端运行
```
pod repo add [本地repos的名称] [远程仓库的地址,这里可以用开源中国的,公司自己的代码管理平台地址等等,用来管理自己的私有源的地址]
pod repo add DemoSpec  https://XXX.git
```
运行完上面的代码重新cd 到~/.cocoapods/repos可以发现刚刚建立的文件以及可以看到了


###(二).创建pod工程 
这里没什么好说的,就是自己本地找个目录创建一个新的工程,可以使用pod的,有旧的项目的话更简单了(这里不做过多的讲解)

###(三).生成spec文件 
自己建一个pod工程（具体不做详解了，不知道的自行百度）这里只讲如何创建spec文件
```
pod spec create [项目sepc的名字] [代码的远程仓库的地址和第一步的地址最后不要一样]
pod spec create PodTestSpec https://xxx.git

```
成功以后会看到 
```
Pod::Spec.new do |s|

 

  s.name         = "PodTestSepec" #podsepc名称

  s.version      = "1.0.0"#版本号

  s.summary      = "year descr."#框架的妙手

  s.homepage     = "https://www.baidu.com"#写你主页的地址  这里是我随意写的

  s.license      = "MIT"  #通行证

  s.author             = { "fangshufeng" => "1039640335@qq.com" }

    s.platform     = :ios, "7.0"

  s.source       = { :git => "地址二", :tag => "1.0.0" }#地址二写的就是步骤3写的地址 tag是版本号

  s.source_files  = "podTest/**/*.{h,m}"#文件的目录

   s.resource  = "podTest/podTest.bundle"#文件的资源 包括图片什么的

   s.requires_arc = true
```
这里列举的是一些常用的

1.先编辑你的spec的文件写成你自己的项目相关的
2.然后再git上打上tag注意这里要和你spec里面写的一样（sourcetree什么都可以）
3.在当前工程的目录下输入下面的命令验证sepc文件
```
pod lib lint
```
###四.向本地的Spec Repo提交spec文件
```
pod repo push DemoSpec PodTestSpec.podspec  #前面是本地Repo名字 后面是podspec名字
```
一般如果使用subspec文件的话，可能会依赖其他的三方库，有时候第三方库有警告可能会导致验证不过，可以使用下面的命令来屏蔽
```
pod repo push DemoSpec PodTestSpec.podspec --use-libraries --allow-warnings --sources='https://github.com/CocoaPods/Specs'
```
如果下面的错误
```
The repo `YourSpecs` at `../../../.cocoapods/repos/YourSpecs ` is not clean 错误
```
可以按照下面的步骤解决
```
cd ~/.cocoapods/repos/YourSpecs   使用git clean -f
```

有时候私有源要指定地址，可以使用`sources `跟上源的地址，有多个源的话以逗号隔开

完成之后这个组件库就添加到我们的私有Spec Repo中了，可以进入到~/.cocoapods/repos/DemoSpec目录下查看
```
├── LICENSE
├── DemoSpex
│   └── 1.0.0
│       └── PodTestSepc.podspec
└── README.md
```
再search命令查看你是否制作成功了（需要花点时间，耐心等候）
```
$ pod search PodTestSpec
 
-> PodTestSpec (0.1.0)
   year descr.
   pod 'PodTestSpec', '~> 1.0.0'
   - Homepage: https://www.baidu.com
   - Source:   地址二
   - Versions: 1.0.0 [DemoSpec repo]

```
到此我们的私有pod以及制作好了 

###(五).pod的使用  
另外新建一个工程
在podfile中加入

```
source "地址一"//就是第一步写的源地址
 platform :ios, '7.0'

target 'hh' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!

  pod 'PodTestSpec', '~> 1.0.0'
  
end
```

关于pod的一些常见的操作请看[这个](http://www.jianshu.com/p/c4f00deaa660)

