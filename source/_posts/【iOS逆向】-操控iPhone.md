---
title: 【iOS逆向】- 操控iPhone
date: 2018-10-28 22:37:00
tags:
    [iOS逆向,WiFi-SSH,USB-SSH]
---


要想逆向app肯定是要对手机进行操控的，远程登录这里采用的是`SSH`协议，这是一个应用层的协议是基于`tcp`的，那至于数据具体是如何从`Mac`到`iPhone`的，从物理层协议来说的话有2个

1. WiFi的方式
2. USB的方式

<!-- more -->

## 一、OpenSSH

先通过`cydia`安装插件`OpenSSH`，他为我们远程登录提供了条件

![15407321062485.jpg](https://upload-images.jianshu.io/upload_images/3279997-09495852eaf1c7b0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)



点击`OpenSSH Access How-To`可以查看使用方式 ，`iPhone`为我们提供了2个账户`root`和`mobile`，默认的密码都是`aipine`,修改默认的密码方式如下

![15407323754486.jpg](https://upload-images.jianshu.io/upload_images/3279997-4492484262c752c8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)


我们是通过`SSH`来链接手机的，具体的方式

```console

ssh 用户名@主机地址

```

例如,我手机的ip地址是`192.168.1.103`那么


```console
ssh root@192.168.1.103

```

## 二、WiFi-SSH

`SSH`是应用层的协议,它是基于`TCP`协议的。

> SSH 为建立在应用层基础上的安全协议。SSH 是目前较可靠，专为远程登录会话和其他网络服务提供安全性的协议。
> 

`SSH`整个过程分为3个步骤

1. 建立连接
2. 客户端登录
3. 数据传输

### 1、建立连接

对于客户端和服务端其实是相对的，谁提供服务那谁就是服务端，我们既然想通过mac来控制手机那么，这里的手机就是服务器，mac就是客户端

**对于服务端**：路径`/etc/ssh/ssh_host_dsa_key.pub`为公钥 `ssh_host_dsa_key`为私钥，比如我这台iPhone的公钥就是

![15407326270589.jpg](https://upload-images.jianshu.io/upload_images/3279997-78f1358b75fdb8d1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**对于客户端**：`~/.ssh/known_hosts`目录下存放着已经主动同意连接的主机信息，对于从未连接过的主机第一次连接会出现像下面的一样的提示信息


![15407310612953.jpg](https://upload-images.jianshu.io/upload_images/3279997-9955214adf9237de.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当你同意以后就会将对应的公钥信息存放到`~/.ssh/known_hosts`文件中，比如上面的我们输入`yes`那么`~/.ssh/known_hosts`内容如下，可以看到正好是上面手机端的公钥信息

![15407312659938.jpg](https://upload-images.jianshu.io/upload_images/3279997-7765484009a08642.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当你再次输入`ssh root@192.168.1.103`的时候则不会再让你同意，而是直接让你输入密码了

![15407313401107.jpg](https://upload-images.jianshu.io/upload_images/3279997-4c43f297b30585f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


要是想删除已经添加的服务端的信息，直接的办法就是直接打开`~/.ssh/known_hosts`然后删除对应的主机信息，另一种方式就是


```console
ssh-keygen -R 服务端IP地址

```

当然服务端和客户端通信还要保证2者的版本号保持一致，还有使用的是什么端口，这些信息保存在


```console
客户端：/etc/ssh/ssh_config
服务端：/etc/ssh/sshd_config
```


### 2、客户端登录

客户端登录有2种方式

> 第一种级别（基于口令的安全验证）
> 只要你知道自己帐号和口令，就可以登录到远程主机。所有传输的数据都会被加密，但是不能保证你正在连接的服务器就是你想连接的服务器。可能会有别的服务器在冒充真正的服务器，也就是受到“中间人”这种方式的攻击。
> 第二种级别（基于密匙的安全验证）
> 需要依靠密匙，也就是你必须为自己创建一对密匙，并把公用密匙放在需要访问的服务器上。如果你要连接到SSH服务器上，客户端软件就会向服务器发出请求，请求用你的密匙进行安全验证。服务器收到请求之后，先在该服务器上你的主目录下寻找你的公用密匙，然后把它和你发送过来的公用密匙进行比较。如果两个密匙一致，服务器就用公用密匙加密“质询”（challenge）并把它发送给客户端软件。客户端软件收到“质询”之后就可以用你的私人密匙解密再把它发送给服务器。
> 用这种方式，你必须知道自己密匙的口令。但是，与第一种级别相比，第二种级别不需要在网络上传送口令。
> 第二种级别不仅加密所有传送的数据，而且“中间人”这种攻击方式也是不可能的（因为他没有你的私人密匙）。但是整个登录的过程可能需要10秒 [2]  。


**第一种**

![15407337723625.jpg](https://upload-images.jianshu.io/upload_images/3279997-933ca1b04940697e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是每次连接服务器 都要 输入 密码

这种方式有2个不好第一是每次都要输入密码，很烦人，第二就是还有安全问题，可能会受到中间人攻击

**第二种**
关于这个上面已经解释很清楚了，需要客户端生成一对公钥和私钥

通过`ssh-keygen`命令生成一对rsa的公钥和私钥

![15407339295368.jpg](https://upload-images.jianshu.io/upload_images/3279997-41a44dd755ca612e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后一路回车到底然后再 `~/.ssh/`目录下生成 `id_rsa`和`id_rsa.pub`文件


```console
➜  .ssh tree
.
├── id_rsa
├── id_rsa.pub

➜  .ssh

```

我们需要把客户端的公钥给到服务端，通过命令

```console
scp-copy-id 登录名称@服务器地址

```

![15407342504144.jpg](https://upload-images.jianshu.io/upload_images/3279997-71dfa30a5453a0f9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


输入密码得到

![15407343044292.jpg](https://upload-images.jianshu.io/upload_images/3279997-4cb03da08a0cf8d6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时我们再在终端输入


```console
ssh root@192.168.1.103

```

直接就链接成功了，无需输入密码了

![15407343334532.jpg](https://upload-images.jianshu.io/upload_images/3279997-1c0531c5cfda9403.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 3. 数据传输

数据传输，其实我们上面就已经是在传输数据了，我们可以通过`scp`命令将mac上的文件拷贝到iPhone上，比如想将mac上的 `~/.ssh/id_rsa.pub`复制到手机的`~`下


```console
scp ~/.ssh/id_rsa.pub 登录名@服务器主机地址:~

scp ~/.ssh/id_rsa.pub root@192.168.1.103:~

```

![15407348198003.jpg](https://upload-images.jianshu.io/upload_images/3279997-5af926c36578b10e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看下手机的目录
[图片上传失败...(image-2979a2-1540736735720)]



## 三、 USB-SSH的方式

上面说的数据都是通过`WIFI`传输的，`mac`和`iPhone`都要联网。而且传输速度也是有限的，mac还提供了一种方式就是通过`usb`来传输数据，首先明白这个：22端口提供SSH服务（可以查看/etc/ssh/sshd_config的Port字段），Mac上有个服务程序usbmuxd（它会开机自动启动），可以将Mac的数据通过USB传输到iPhone.可以通过以下路径找到 


```console
/System/Library/PrivateFrameworks/MobileDevice.framework/Resources/usbmuxd

```


要想使用该服务，需要下载[文件](https://cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz)，里面有个python脚本`tcprelay.py`


这个脚本可以将本地的端口和手机的`22`端口进行映射 

通过命令将本地的`10001`端口和`22`端口进行映射，`10001`根据自己定，只要不是保留端口就可以了


```console
python ~/jailbreak/Tcprelay.py -t 22:10001

```
![15407353414928.jpg](https://upload-images.jianshu.io/upload_images/3279997-744b8443d6710282.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样就映射成功了

然后新开一个终端通过下面命令登录iPhone


```console
 ssh  root@localhost -p 10001
```

发现也登录成功了,在逆向工程里面还是用的usb的方式多一些，我们可以把上面的命令写成shell脚本的形式，方式执行


当然也可以拷贝文件到手机上


```console
scp -P 10010 ~/.ssh/id_rsa.pub  root@localhost:~/

```

注意`P`大写
























