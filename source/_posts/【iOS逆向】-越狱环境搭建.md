---
title: 【iOS逆向】-越狱环境搭建
date: 2018-10-28 22:30:53
tags: 
    [iOS逆向,Objective-C]
---



本篇会记录整个逆向的过程中使用到的工具和插件，会持续修改

<!-- more -->

## 一、越狱设备

1. 一台已经完美越狱的iPhone手机，我的设备是`iPhone6plus` (iOS 9.1)，想查看那些设备可以完美越狱[戳这个 ](http://jailbreak.25pp.com/ios/)
2. 一台MacBook


## 二、常用软件 

#### iPhone上

1. Apple File Conduit "2"（可以访问整个文件系统）
2. iFile （可以在手机上查看文件）
3. PP助手 （安装软件用的）
4. OpenSSH （远程连接使用的）
5. Vi IMproved （让手机支持 vim）

`iFile`截图如下

![15407167705373.jpg](https://upload-images.jianshu.io/upload_images/3279997-51d47cfa2b72aa16.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700/h/500)

#### Mac上

1. iFunBox （可以在Mac上访问手机的目录，类似手机上的iFile）
2. PP助手

`iFunBox`如下

![15407167979219.jpg](https://upload-images.jianshu.io/upload_images/3279997-1628cdda80bbdc3c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)


## 三、常见安装问题

先明白下面2个东西

1. 通过Cydia安装的安装包是`deb`格式的，一般是装插件的
2. 通过PP助手安装的安装包是`ipa`格式的，一般是装app的


要是在`Cydia`上安装插件失败，可以在网上找对应的`deb`格式的文件放到目录`/var/root/Media/Cydia/AutoInstall`下,然后重启手机即可




