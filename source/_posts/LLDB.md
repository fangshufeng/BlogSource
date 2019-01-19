---
title: LLDB
date: 2018-11-22 21:59:26
tags:
    Objective-C
---



平时开发的过程中使用`Xcode`都是图形化操作习惯了，要是脱离了xcode你还能调试代码吗，恩，`Xcode`已经把我们惯坏了，不管怎样，作为一个开发对于了解图形操作背后的东西应该要了解。

申明：本文的`[]`都表示命令简写

<!-- more -->

### 一、增加方法断点


*  breakpoint set --name[-n]  "**`方法名称`**"
*  breakpoint set --address[-a] **`方法的内存地址`**
    
![15428751580151.jpg](https://upload-images.jianshu.io/upload_images/3279997-ffbc506b30d9a937.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面4个都是等价的，通过`breakpoint list`查看已经添加的断点


* 模糊设置断点 breakpoint set -r **`方法名称`**,r 是正则的意思

    ```
    (lldb) breakpoint set -r foo
    Breakpoint 2: 136 locations.
    ```

    发现有136个地方包含了`foo`字符串的方法,通过`breakpoint list`查看可以知道在哪里有这个方法，输出太多这边就截取一部分。

    ```
    (lldb) breakpoint list
    Current breakpoints:
    1: file = '/Users/fangshufeng/Desktop/thirdPart/image/Image/Image/ViewController.m', line = 24, exact_match = 0, locations = 1, resolved = 1, hit count = 2
    
      1.1: where = Image`-[ViewController touchesBegan:withEvent:] + 77 at ViewController.m:24, address = 0x000000010a4f768d, resolved, hit count = 2 
    
    2: regex = 'foo', locations = 136, resolved = 136, hit count = 2
      2.1: where = Image`-[ViewController foo] + 23 at ViewController.m:30, address = 0x000000010a4f76f7, resolved, hit count = 2 
      2.2: where = libbsm.0.dylib`au_print_xml_footer, address = 0x000000010ce5d665, resolved, hit count = 0 
      2.3: where = libsystem_kernel.dylib`proc_reset_footprint_interval, address = 0x000000010d5254ed, resolved, hit count = 0 
      
     [...]
      
    ```
    
* 开关断点
    * breakpoint disable `断点编号`（让某个断点不可用但是不是删除）
    * breakpoint enable `断点编号`（让断点由不可用到可用）
    * breakpoint delete `断点编号` （删除断点不可恢复，需重新添加，不加编号则让你选择是否全部删除）


![15428758913025.jpg](https://upload-images.jianshu.io/upload_images/3279997-aa7e646070f69751.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 二、线程断点

#### 源码调试

这个平时用的最多了，看下图

* thread continue [continue] [c ] : 程序继续
* thread step-over [next] [n] : 单步运⾏，把子函数当做整体⼀一步执行
* thread step-in [step] [s] : 单步运⾏，遇到子函数会进入⼦函数
* thread step-out [finish] : 直接执行完函数，返回函数调用处

太常用不贴代码了

![15428763316698.jpg](https://upload-images.jianshu.io/upload_images/3279997-18e643a8a70aca3b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



* thread backtrace [bt]:查看调用栈回溯

![15428766389548.jpg](https://upload-images.jianshu.io/upload_images/3279997-b4cb4d10da5d501a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如图有时想查看代码的调用堆栈，但是xcode无法查看全，左边的地方只能看到部分，当然你可以点击这个地方，去查看。

![15428767683217.jpg](https://upload-images.jianshu.io/upload_images/3279997-dbd6792e83653ec4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也可以通过命令`thread backtrace`或者`bt`

![15428770034517.jpg](https://upload-images.jianshu.io/upload_images/3279997-fa8e02dc8e18da66.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`frame`为栈帧的意思，想了解函数栈帧的[点击此处](https://fangshufeng.com/2018/10/07/assembly-function/)

* frame variable :查看函数栈帧内的局部变量的值


```
(lldb) frame variable
(ViewController *) self = 0x00007fe390a04960
(SEL) _cmd = "touchesBegan:withEvent:"
(__NSSetM *) touches = 0x0000600003d30640 1 element
(UITouchesEvent *) event = 0x0000600000f66250
(int) a = 10

```

我们知道`oc`的方法内置了`self`和`_cmd`参数,所以上面有`self`和`_cmd`

#### 汇编代码调试

开启汇编模式

![15428781223072.jpg](https://upload-images.jianshu.io/upload_images/3279997-5846082b5cad0fea.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


和源码的区别是

* ni : 单步运⾏，把子函数当做整体⼀一步执行
* si: 单步运⾏，遇到子函数会进入⼦函数

### 三、内存断点

有时候想看某个内存发生改变的时候触发断点

* watchpoint set variable 变量 
* watchpoint set variable 地址

我们给vc增加一个属性`bar`


```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"fff");
    [self foo];
  
}


- (void)foo {
    NSLog(@"--foo--");
    NSLog(@"--foo2--");
    NSLog(@"--foo3--");
    
    for ( int i = 0; i < 3; i++) {
        self.bar += 10;
    }
}

```

通过该命令，每当内存地址内容被修改都会断到

![15428798696567.jpg](https://upload-images.jianshu.io/upload_images/3279997-5c810caacadbee7d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



还可以配合`watchpoint command`使用

* watchpoint command add 断点编号：当内存断点触发的时候执行一些操作

每次改变都会输出值。
![15428800472134.jpg](https://upload-images.jianshu.io/upload_images/3279997-8bac6cfe15f52ddb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
 watchpoint command add 1 // 1 表示 watchpoint 断点编号
Enter your debugger command(s).  Type 'DONE' to end.
> p self.bar;
> DONE // DONE 表示命令输入结束

```

相关指令如下，见名知意了不需要作介绍了

* watchpoint list
* watchpoint disable 断点编号
* watchpoint enable 断点编号 
* watchpoint delete 断点编号 
* watchpoint command list 断点编号 
* watchpoint command delete 断点编号

### 四、不常用但很实用的指令

场景1：

![15428794369809.jpg](https://upload-images.jianshu.io/upload_images/3279997-bf96628c8fb06e66.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要想代码执行到31行的断点的时候直接返回，不想执行后面的代码，就可以通过`thread return`指令，当代码执行到31行断点时，`thread return`可以实现。


场景2：有时崩溃了直接到`main`函数了，无法知道崩溃的地方

都知道下面的代码会崩溃

![15428804652206.jpg](https://upload-images.jianshu.io/upload_images/3279997-590b234867f682cc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



此时崩溃在下面的main函数处

![15428805038949.jpg](https://upload-images.jianshu.io/upload_images/3279997-db48af4dd528ed27.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

终端输出信息

![15428805244685.jpg](https://upload-images.jianshu.io/upload_images/3279997-b902fb1bd98d02f4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


并不知道函数崩溃在哪一行可以使用`image lookup`指令


```
(lldb) image lookup -a 0x00000001022365db
      Address: Image[0x00000001000015db] (Image.__TEXT.__text + 203)
      Summary: Image`-[ViewController touchesBegan:withEvent:] + 139 at ViewController.m:30
      
```

发现崩溃在`ViewController`的`touchesBegan:withEvent:`在30行处。



`image lookup`周边

* image lookup -t 类：查看类的属性


```
(lldb) image lookup -t UIView
Best match found in /Users/fangshufeng/Library/Developer/Xcode/DerivedData/Image-cncplmezlusnyackyzpyppykazhf/Build/Products/Debug-iphonesimulator/Image.app/Image:
id = {0x00001063}, name = "UIView", byte-size = 8, decl = UIView.h:143, compiler_type = "@interface UIView : UIResponder
@property ( readonly,getter = layerClass,setter = <null selector>,nonatomic,class ) Class layerClass;
@property ( getter = isUserInteractionEnabled,setter = setUserInteractionEnabled:,assign,readwrite,nonatomic ) BOOL userInteractionEnabled;
@property ( getter = tag,setter = setTag:,assign,readwrite,nonatomic ) NSInteger tag;
@property ( readonly,getter = layer,setter = <null selector>,nonatomic ) CALayer * layer;
@property ( readonly,getter = canBecomeFocused,setter = <null selector>,nonatomic ) BOOL canBecomeFocused;
@property ( readonly,getter = isFocused,setter = <null selector>,nonatomic ) BOOL focused;
@property ( getter = semanticContentAttribute,setter = setSemanticContentAttribute:,assign,readwrite,nonatomic ) UISemanticContentAttribute semanticContentAttribute;
@property ( readonly,getter = effectiveUserInterfaceLayoutDirection,setter = <null selector>,nonatomic ) UIUserInterfaceLayoutDirection effectiveUserInterfaceLayoutDirection;
@end"
```

* image lookup -n 方法名称：查看方法的位置


```
(lldb) image lookup -n foo
1 match found in /Users/fangshufeng/Library/Developer/Xcode/DerivedData/Image-cncplmezlusnyackyzpyppykazhf/Build/Products/Debug-iphonesimulator/Image.app/Image:
        Address: Image[0x0000000100001660] (Image.__TEXT.__text + 224)
        Summary: Image`-[ViewController foo] at ViewController.m:33
        
```

* image list -o -f ：可以查看内存的偏移量，这个在逆向中很常用

还有一些常见的指令比如`p`、`po`、`expression`就不说了。

