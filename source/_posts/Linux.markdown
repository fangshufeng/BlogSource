---
layout: post
title: "Linux下常见命令"
date: 2017-12-31 11:44:49 +0800
comments: true
categories: 
tag:
    其他
---


本文旨在记录平时开发中常见的Linux的命令

<!--more-->

## （一）文件操作相关    
<1>  查看文件信息
        `ls`
        
| 参数 | 意义 |
| :-: | :-: |
| -l | 显示目录下所有的文件（不包含隐藏文件夹） |
| -a | 显示目录下所有的文件，包含隐藏文件夹 |
| -h | 显示文件的大小 |
![](http://upload-images.jianshu.io/upload_images/3279997-da32913d93972be2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/3279997-e62b08eba63924a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/3279997-7b8774c7dc6a4377.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/3279997-80f19c082744688e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15146470558654.jpg](http://upload-images.jianshu.io/upload_images/3279997-2ad4942392654042.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15146471567477.jpg](http://upload-images.jianshu.io/upload_images/3279997-24235839fe064dd0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


支持正则

![15146476046274.jpg](http://upload-images.jianshu.io/upload_images/3279997-6682ecc8eb4ce845.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15146476921271.jpg](http://upload-images.jianshu.io/upload_images/3279997-35e3e773aad1df11.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



<2>  重定向 `>`

可以通过这个命令将前一个命令的输出内容输出到后面的文件中
![15146481045487.jpg](http://upload-images.jianshu.io/upload_images/3279997-ee9f0a9b6cefff8c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15146483656968.jpg](http://upload-images.jianshu.io/upload_images/3279997-dc640f01469d2509.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



<3>  查看文件的内容 `cat` `more`

  区别在于`cat`查看的是文件的全部内容，`more`查看的是文件的部分内容，可以通过`b`,`f`进行分页查看
    

4 管道 `|`

管道我们可以理解现实生活中的管子，管子的一头塞东西进去，另一头取出来，这里“ | ”的左右分为两端，左端塞东西(写)，右端取东西(读)。


![15146493036610.jpg](http://upload-images.jianshu.io/upload_images/3279997-3743c3f1fb5183a1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<4>  文件的增删改查

![15146494753832.jpg](http://upload-images.jianshu.io/upload_images/3279997-12717a0f5282c00b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)!


<4>  文件夹的增删改查

![15146494753832.jpg](http://upload-images.jianshu.io/upload_images/3279997-ec2321473c32b674.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15146500486221.jpg](http://upload-images.jianshu.io/upload_images/3279997-d03f250e247f966e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<5>  给文件建立连接`ln`


```
    链接文件分为软链接和硬链接。

    软链接：软链接不占用磁盘空间，源文件删除则软链接失效。

    硬链接：硬链接只能链接普通文件，不能链接目录。
    使用格式：

    ln 源文件 链接文件          
    ln -s 源文件 链接文件
    
```


如果没有-s选项代表建立一个硬链接文件，两个文件占用相同大小的硬盘空间，即使删除了源文件，链接文件还是存在，所以-s选项是更常见的形式。

![15146504737615.jpg](http://upload-images.jianshu.io/upload_images/3279997-4948ca94706b68bc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15146505874211.jpg](http://upload-images.jianshu.io/upload_images/3279997-97c21f29dd436411.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15146507756887.jpg](http://upload-images.jianshu.io/upload_images/3279997-3f947f4203a30aee.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



<5>  查找 `grep`

    Linux系统中grep命令是一种强大的文本搜索工具，grep允许对文本文件进行模式查找。如果找到匹配模式， grep打印包含模式的所有行。

grep一般格式为：

```
grep [-选项] ‘搜索内容串’文件名
```

在grep命令中输入字符串参数时，最好引号或双引号括起来。例如：grep‘a ’1.txt。

常用选项说明：

选项	含义

```
-v	         显示不包含匹配文本的所有行（相当于求反）
-n	         显示匹配行及行号
-i	         忽略大小写

```


![15146512298338.jpg](http://upload-images.jianshu.io/upload_images/3279997-3af02d82b0b23b28.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<6>  查看某个命令的执行路径`which`


![15146517445369.jpg](http://upload-images.jianshu.io/upload_images/3279997-c6909461218b7e90.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<7>  归档管理:`tar`
    该命令不会对文件进行压缩，只是归档
    
| 参数 | 含义 |
| --- | --- |
| -c | 归档 |
| -x | 解档 |
| -v | 压缩的过程中显示文件 |
| -f | 指定档案文件名称，f后面一定是.tar文件，<font color="#ff6f39">所以必须放选项最后</font> |


![15146888668722.jpg](http://upload-images.jianshu.io/upload_images/3279997-75edc70c213a1e08.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用`tar`实现压缩功能

![15146897346686.jpg](http://upload-images.jianshu.io/upload_images/3279997-c9815147d8f07f4c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)!

<8>文件压缩解压：zip、unzip
![15146911301358.jpg](http://upload-images.jianshu.io/upload_images/3279997-053b3c7cbb1960c1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## （二）权限相关
说到修改文件的权限，先看看表达的意思

[15147758007357.png](http://upload-images.jianshu.io/upload_images/3279997-66673a03c5075eb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


| rwx | 含义 |
| --- | --- |
| r | read 表示可读取，对于一个目录，如果没有r权限，那么就意味着不能通过ls查看这个目录的内容。 |
| w | write 表示可写入，对于一个目录，如果没有w权限，那么就意味着不能在目录下创建新的文件。 |
| x | excute 表示可执行，对于一个目录，如果没有x权限，那么就意味着不能通过cd进入这个目录。 |


`-rw-r--r--`：可分为4个部分分别为1，3，3，3；
1. 第一位：`d`表示是当前文件是文件夹,`-`表示是文件；
2. 第2-4位：整个三位表示文件只有者的权限
3. 第5-7位：表示用户组的权限；
4. 后面三位：表示其他用户的权限

![15147767262771.jpg](http://upload-images.jianshu.io/upload_images/3279997-e881c18ff97d299b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)!

修改文件权限：`chmod`

chmod 修改文件权限有两种使用格式：`字母法`与`数字法`.

（1）.`字母法`：chmod  u/g/o/a  +/-/=  rwx 文件

| u/g/o/a  | 含义 |
| --- | --- |
| u | user 表示该文件的所有者 |
| g | group 表示与该文件的所有者属于同一组( group )者，即用户组|
| o | other 表示其他以外的人 |
| a | all 表示这三者皆是|


|  +-=   | 含义 |
| --- | --- |
| + | 增加权限                                          |
| - | 撤销权限                                          |
| = | 设定权限                                          |

![15147774483062.jpg](http://upload-images.jianshu.io/upload_images/3279997-561a79cb5ee4f2fd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15147775067476.jpg](http://upload-images.jianshu.io/upload_images/3279997-22793bd3810daf14.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![15147776573072.jpg](http://upload-images.jianshu.io/upload_images/3279997-1f8e1de6435128b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也可以同时设定


![15147778327004.jpg](http://upload-images.jianshu.io/upload_images/3279997-dd9d7dbdd59dd08a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（2）.数字法

| 字母  | 含义 |
| --- | --- |
| r | 读取权限，数字代号为 "4"|
| w | 写入权限，数字代号为 "2"|
| x | 执行权限，数字代号为 "1" |
| - | 不具任何权限，数字代号为 "0"|

如执行：chmod u=rwx,g=rx,o=r filename 就等同于：chmod u=7,g=5,o=4 filename

chmod 751 file：

文件所有者：读、写、执行权限
同组用户：读、执行的权限
其它用户：执行的权限

![15147780482708.jpg](http://upload-images.jianshu.io/upload_images/3279997-3d03e2ede9d400b7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其他的一些操作

control+a，跳到命令行开始位置；control+e，跳到命令行结尾位置。
option+f，向后跳一个word；option+b，向前跳一个word。
option+d，向后删除一个word；option+delete，向前删除一个word。
control+_，撤销之前一次编辑操作。
control+k，删除到行尾；control+u，删除到行首。

