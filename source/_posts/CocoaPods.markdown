---
layout: post
title: CocoaPods常见命令
date: 2017-02-09 16:31:17 +0800
comments: true
categories: 
tag:
    Objective-C
---



```
sudo gem install cocoapods
```

通过gem来安装cocoapods(上面命令会访问https://rubygems.org/)然后就被墙了

<!--more-->

```
gem sources -l
```
通过该命令来查看当前的源

```
1.gem sources --remove https://rubygems.org/
2.gem sources -a http://ruby.taobao.org/
```
上面2个命令然后镜像到国内的 都说淘宝的不能用了 如果不行的话试试下面这个
```
gem source -r https://ruby.taobao.org   
gem source -a https://gems.ruby-china.org 
gem source -u # 更新 update  
```
换完了源之后再重新执行 sudo gem install cocoapods

```
pod setup  用来同步master上的库。我们需要经常执行这条命令，否则有新的Pods依赖库的时候执行pod search命令是搜不出来的
```
```
gem list --local | grep cocoapods 查看当前安装了哪些版本
```
```
sudo gem uninstall cocoapods 删除安装过的所有版本
```
```
gem uninstall cocoapods -v xxx 强制卸载某一个版本 
```
```
sudo gem install cocoapods -v xxx.xxx.xxx 安装指定版本
```


```
可以通过 pod --help 查看相关命令
```

```
cd到文件下 pod init 创建Podfile文件 
```
```
pod install 生成的Podfile.lock文件 lock文件会保存pod版本
pod update 重新生成Podfile.lock文件 
```

```
pod install --verbose --no-repo-update 可快速更新忽略本地repo的更新
```

=================cocoapod常见报错===================

#更新gem报错(sudo gem update --system)
```
错误一:
ERROR: While executing gem … (Errno::EPERM) 
Operation not permitted - /usr/bin/xcodeproj 
```

```
执行sudo gem install -n /usr/local/bin cocoapods
```
```
错误二： 
While executing gem … (Errno::EPERM) 
Operation not permitted - /usr/bin/update_rubygems
```
```
安装Homebrew（Homebrew installs the stuff you need that Apple didn’t.） 
objc  /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" 
```


当pod install失败的时候可以尝试先卸载再重新安装试试

