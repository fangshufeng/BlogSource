---
layout: post
title: "网络编程必备调试工具 WireShark 、 Cisco Packet Tracer"
date: 2018-02-01 14:17:29 +0800
comments: true
categories: 
tag:
    Cisco
---



## WireShark
如何要想了解网络那块的知识这是一个很好的查看工具 
[下载地址](https://www.wireshark.org/download.html) ，貌似要翻墙，可根据自己电脑的类型下载对应的版本

下面提供一个Tcp三次握手的截图

![15174541729110.jpg](http://upload-images.jianshu.io/upload_images/3279997-0dbc6c8dd9d1e525.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**一些逻辑运算 比如  == || && and  or > < 都是可以的**

<!--more-->
#### 按照ip过滤
1. 查看目的地址为192.168.33.10 ： ip.dst == 192.168.33.10
2. 同理查看源ip地址： ip.src == 192.168.33.10
3. ip.dst == 192.168.33.10 || ip.src == 192.168.33.10

#### 按照端口过滤
1. tcp.port == 7768 ->  查看tcp协议  端口号为7768的 
2. tcp.port == 7768 || udp.port == 1321


## Cisco Packet Tracer

这玩意貌似没有mac版的，Windows下也特别难找，如何你有麻烦给我留言，谢谢，一堆广告，烦死，如果你闲麻烦可以用我这个，[点击下载](https://pan.baidu.com/s/1c346hkk)
![15174550211009.jpg](http://upload-images.jianshu.io/upload_images/3279997-d7258fbf995fae28.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 解压后放到对应的地方，找到截图的文件打开`PacketTracer5`，当然你也可以自己另行下载

这个软件的使用需要具备一定的网络协议的了解，建议使用前先看看`阮一峰的`这两篇文章，不然可能有些难度

> [互联网协议入门（一）](http://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html)
> [互联网协议入门（二）](http://www.ruanyifeng.com/blog/2012/06/internet_protocol_suite_part_ii.html)

---
下面是一个路由器组网的截图

![15174554251720.jpg](http://upload-images.jianshu.io/upload_images/3279997-0d84971452a4ce8e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15174555013444.jpg](http://upload-images.jianshu.io/upload_images/3279997-13e3a5015514424d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
---
**下面分别说说如何搭建网络**

工具基本介绍如下
![15174561075956.jpg](http://upload-images.jianshu.io/upload_images/3279997-139f4862f22c2234.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 两个pc组网

步骤

1 增加2个pc端

![15174561966601.jpg](http://upload-images.jianshu.io/upload_images/3279997-5afd053b663a58e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)          


2 配置网络信息

![15174563247689.jpg](http://upload-images.jianshu.io/upload_images/3279997-ef109365a3250725.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![15174563571952.jpg](http://upload-images.jianshu.io/upload_images/3279997-2023a2efd7a348dd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



pc0 : ip :192.168.1.1  子网掩码：255.255.255.0 对应的网段为192.168.1.0
pc1 : ip :192.168.1.2  子网掩码：255.255.255.0 对应的网段为192.168.1.0

因为是同一网段可以直接通信

点击pc0 ping 192.168.1.2

![15174565239009.jpg](http://upload-images.jianshu.io/upload_images/3279997-00dacfceabcf4f52.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15174565437229.jpg](http://upload-images.jianshu.io/upload_images/3279997-ea666f26cca6db7c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


表示通信完成

在点击之前先执行下面的操作

![15174567253385.jpg](http://upload-images.jianshu.io/upload_images/3279997-10dc9a22db88aff7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



可以看到具体的协议名称

![15174567667460.jpg](http://upload-images.jianshu.io/upload_images/3279997-e366c4a3f6813f4c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击信封可以查看对应的信息，现在你可以开始学习对应的协议了

点击下图可以执行下一步
![15174568055244.jpg](http://upload-images.jianshu.io/upload_images/3279997-94b8edfc34c8a994.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


结合这2个工具，在结合书本，相信理解tcp/ip无压力

### 2个路由器组网

怎么拉出控件就不说了，最后是这样的

![15174637134612.jpg](http://upload-images.jianshu.io/upload_images/3279997-a2b78f2c809e4712.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


为了不同网段可以通信

还要对路由器进行[静态路由](https://baike.baidu.com/item/%E9%9D%99%E6%80%81%E8%B7%AF%E7%94%B1/100778?fr=aladdin)配置

> 基本的静态路由举例如图所示，由两个路由器R1和R2组成（接口号和IP地址在图中给出），它们分别连接了各自的网络：R1连接了子网192.168.0.0/24，R2连接了子网192.168.2.0/24[1]  。
在没有配置静态路由的情况下，这两个子网中的计算机A、B之间是不能通信的。从计算机A发往计算机B的IP包，在到达R1后，R1不知道如何到达计算机B所在的网段192.168.2.0/24（即R1上没有去往192.168.2.0/24的路由表），同样R2也不知道如何到达计算机A所在的网段192.168.0.0/24，因此通信失败。
此时就需要管理员在R1和R2上分别配置静态路由来使计算机A、B成功通信。
在R1上执行添加静态路由的命令ip route 192.168.2.0 255.255.255.0 192.168.1.1。它的意思是告诉R1，如果有IP包想达到网段192.168.2.0/24，那么请将此IP包发给192.168.1.1（即和R1的2号端口相连的对端）。
同时也要在R2上执行添加静态路由的命令ip route 192.168.0.0 255.255.255.0 192.168.1.2。它的意思是告诉R2，如果有IP包想达到网段192.168.0.0/24，那么请将此IP包发给192.168.1.2（即和R2的3号端口相连的对端）。
通过上面的两段配置，从计算机A发往计算机B的IP包，能被R1通过2号端口转发给R2，然后R2转发给计算机B。同样地，从计算机B返回给计算机A的IP包，能被R2通过3号端口转发给R1，然后R1转发给计算机A，完成了一个完整的通讯过程。

![15174640310909.jpg](http://upload-images.jianshu.io/upload_images/3279997-7494a95f292adb0e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


告诉路由器1 3网段的走向下个路由器 192.168.2.2

![15174638101921.jpg](http://upload-images.jianshu.io/upload_images/3279997-e94cfb300b33f625.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



同理 路由器2也要配置

![15174639026800.jpg](http://upload-images.jianshu.io/upload_images/3279997-d24349e6f37391ed.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

### 组件http服务器

其他的和2个路由器组网配置是一样的
![15174642109194.jpg](http://upload-images.jianshu.io/upload_images/3279997-3187cae120f67239.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


改为`hello world`
![15174642783562.jpg](http://upload-images.jianshu.io/upload_images/3279997-c2c6a18651e434b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击第一个pc

![15174643271039.jpg](http://upload-images.jianshu.io/upload_images/3279997-bd479ad507844dfd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



浏览器输入192.168.3.2
![15174644997280.jpg](http://upload-images.jianshu.io/upload_images/3279997-3c2f8213b80c71c5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---

### 组件DNS服务器

再增加一台域名解析服务器（DNS）做到浏览器访问test.com 就可以访问刚才的192.168.3.3的内容也就是`hello world`

![15174647781746.jpg](http://upload-images.jianshu.io/upload_images/3279997-07b63cf95664f0b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15174648961413.jpg](http://upload-images.jianshu.io/upload_images/3279997-5518af94aa675077.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



同样打开pc1可以看到
![15174649577457.jpg](http://upload-images.jianshu.io/upload_images/3279997-f746aa4e4026e02e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---
***该文只是带大家入个门，具体每个协议好需要自己好好研究，相信有了这个工具，理解起来会好很多,希望能帮助到大家***




















