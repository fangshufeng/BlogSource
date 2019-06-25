---
title: 深入理解fishhook
date: 2019-04-07 11:36:05
tags: 
    [fishhook ,Objective-C]
---




## 一、fishhook能做什么事情？

`c`函数的地址是在编译的时候就已经确定了，位于程序的`TEXT`段，为只读区域：

![15545964839502.jpg](https://upload-images.jianshu.io/upload_images/3279997-8f794d6e41192329.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!-- more -->

如图，当调用的时候直接找到函数的地址执行，貌似我们无法`Hook`函数的实现，当然，除非你直接修改二进制文件，但是`fishhook`做到了，以系统的`strlen`为例如下：


```c++

static int (*orig_strlen)(const char *__s);
int my_strlen(const char *__s) {
    printf("===\n");
    return orig_strlen(__s);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        struct rebinding strlen_rebinding = { "strlen", my_strlen,
            (void *)&orig_strlen };
        
        rebind_symbols((struct rebinding[1]){ strlen_rebinding }, 1);
        
        char *str = "HelloWorld";
        printf("%d\n",strlen(str));
    }
    return 0;
}

```

输出

```
===
10
```


## 二、什么是lazybind

### nl-symbol-ptr 和 la-symbol-ptr

在`oc`中，都是模块化的架构，比如app间共享缓存，编译，链接的时候对于外部的符号并没有指定地址，但是知道在哪里可以找到,可以通过`nm`命令查看


```c
➜  Debug nm -m TestFishhook
0000000100002090 (__DATA,__bss) non-external __ZL16_rebindings_head
00000001000013f0 (__TEXT,__text) non-external __ZL18prepend_rebindingsPP16rebindings_entryP9rebindingm
00000001000014c0 (__TEXT,__text) non-external __ZL24rebind_symbols_for_imageP16rebindings_entryPK11mach_headerl
00000001000018c0 (__TEXT,__text) non-external __ZL25_rebind_symbols_for_imagePK11mach_headerl
00000001000018f0 (__TEXT,__text) non-external __ZL30perform_rebinding_with_sectionP16rebindings_entryP10section_64lP8nlist_64PcPj
0000000100002088 (__DATA,__data) non-external __ZL6indexx
                 (undefined) external __dyld_get_image_header (from libSystem)
                 (undefined) external __dyld_get_image_name (from libSystem)
                 (undefined) external __dyld_get_image_vmaddr_slide (from libSystem)
                 (undefined) external __dyld_image_count (from libSystem)
                 (undefined) external __dyld_register_func_for_add_image (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
                 (undefined) external _dladdr (from libSystem)
0000000100001c60 (__TEXT,__text) external _foo
                 (undefined) external _free (from libSystem)
0000000100001c90 (__TEXT,__text) external _main
                 (undefined) external _malloc (from libSystem)
                 (undefined) external _memcpy (from libSystem)
                 (undefined) external _objc_autoreleasePoolPop (from libobjc)
                 (undefined) external _objc_autoreleasePoolPush (from libobjc)
                 (undefined) external _printf (from libSystem)
0000000100001800 (__TEXT,__text) non-external (was a private external) _rebind_symbols
0000000100001370 (__TEXT,__text) non-external (was a private external) _rebind_symbols_image
                 (undefined) external _strcmp (from libSystem)
                 (undefined) external _strlen (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
                 
```

`_strlen`这个符号的地址是`undefined`，但是我们告诉我们在`libSystem`库中可以找到，那这些符号有什么时候被绑定真实的地址的呢，答案是`dyld`在`main`函数之前完成这一操作，简单提下`dyld`，这是一个动态链接器，当程序启动的时候，他会根据`load command`中信息去加载需要的动态库，然后解析符号，这里我们关注的是`lz-symbol-ptr`和`nl-symbol-ptr`，这个是每个`macho`都会有的处于`__DATA`段中的；

1. `nl-symbol-ptr`：这里记录着在程序的加载的时候需求立刻绑定地址的符号；
2. `lz-symbol-ptr`： 这里记录并不需要立刻绑定符号，而是当第一次使用该符号的时候通过`dyld_stub_binder`绑定符号，后面再调用该符号的时候就直接使用真实地址，不再使用`dyld_stub_binder`了


至于这么做的原因嘛，也是很容易理解，就是如果使用的符号都要立刻解析的话势必会加长应用的启动时间的。


通过`MachOView`可以看下这2个段


![15546003535041.jpg](https://upload-images.jianshu.io/upload_images/3279997-11f63ab53892cd63.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15546003943714.jpg](https://upload-images.jianshu.io/upload_images/3279997-79745f47b1472d10.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 一个简单的例子

将代码改成如下形式

```oc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        char *s = "ssssss";
        strlen(s);
        strlen(s);
    }
    return 0;
}
```
![15546007993764.jpg](https://upload-images.jianshu.io/upload_images/3279997-f80e970b0a6a1c40.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前面我们说到外部的符号编译的时候不会绑定地址的，`oc`这里指向的并不是`strlen`地址，而是一个桩地址，你可以理解为占位符，双击会到

![15546009170035.jpg](https://upload-images.jianshu.io/upload_images/3279997-0d4238e83433c6c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面这个是属于`__stubs`

![15546009691542.jpg](https://upload-images.jianshu.io/upload_images/3279997-03dfd31adb9df368.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接着双击会到`la-symbol-ptr`中的`__strlen_ptr`

![15546010362039.jpg](https://upload-images.jianshu.io/upload_images/3279997-f928abb3eb57be0a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个符号的内容也就是`0x0000000100001DCE`

![15546011364488.jpg](https://upload-images.jianshu.io/upload_images/3279997-c47d1eccdf86a876.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

找到该地址是位于`__stub_helper`


![15546011839752.jpg](https://upload-images.jianshu.io/upload_images/3279997-2a20b096f7cf1917.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里就是上面提到的`dyld_stub_binder`绑定`la-symbol-ptr`的地方，总结下上面提到的东西，我们一共提到了三个`section`分别是:

1. `__stubs`: 这里就是桩地址，真实函数实现的占位符
2. `__stub_helper`: 实现将占位符绑定真实函数的，并修改`la-symbol-ptr`中保存的地址
3. `la-symbol-ptr`: 保存了__stub_helper的执行地址，也可能是真实地址了（当调用过函数就会是真实地址了）

下面来证明下：

![15546016084023.jpg](https://upload-images.jianshu.io/upload_images/3279997-7697194bfa0702b0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过`image list`知道偏移量为0

```c++
(lldb) image list
[  0] F3B691B0-DAE1-32D3-86AB-A9DABD853733 0x0000000100000000 /Users/fangshufeng/Library/Developer/Xcode/DerivedData/TestFishhook-fwswwhkwaochseaznkjwburwtjaz/Build/Products/Debug/TestFishhook 

```


```c++
(lldb) x 0x0000000100002078
0x100002078: ce 1d 00 00 01 00 00 00 00 00 00 00 40 00 00 00  ............@...
0x100002088: ff ff ff ff 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

为什么是`0x0000000100002078`，前面我们看到在`la-symbol-ptr`中`strlen`的位于地址`0x0000000100002078`，又程序运行的偏移量为0，所以是这个地址，里面保存的值`0x100001dce`


过掉`28`行的断点


```
(lldb) x 0x0000000100002078
0x100002078: 00 87 4e 68 ff 7f 00 00 00 00 00 00 40 00 00 00  ..Nh........@...
0x100002088: ff ff ff ff 00 00 00 00 00 00 00 00 00 00 00 00  ................
(lldb) 

```

发现地址已经变成了`0x7fff684e8700`,这个就是`strlen`的真实地址了。

![15546020547203.jpg](https://upload-images.jianshu.io/upload_images/3279997-76ff781aa73d8650.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面整个流程就是`lazybind`。

###  `_dyld_register_func_for_add_image`

这个是`dyld`提供的一个回调，传给你`mach_header`和`vmaddr_slide`让你可以做一些事情，我们的`fishhook`就是用到了这个，源码可以在`dyld`中找到


```c++
void _dyld_register_func_for_add_image(void (*func)(const struct mach_header *mh, intptr_t vmaddr_slide))
{
	if ( dyld::gLogAPIs )
		dyld::log("%s(%p)\n", __func__, (void *)func);
	dyld::registerAddCallback(func);
}


void registerAddCallback(ImageCallback func)
{
	// now add to list to get notified when any more images are added
	sAddImageCallbacks.push_back(func);
	
	// call callback with all existing images
	for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {
		ImageLoader* image = *it;
		if ( image->getState() >= dyld_image_state_bound && image->getState() < dyld_image_state_terminated ) {
			dyld3::ScopedTimer timer(DBG_DYLD_TIMING_FUNC_FOR_ADD_IMAGE, (uint64_t)image->machHeader(), (uint64_t)(*func), 0);
			(*func)(image->machHeader(), image->getSlide());
		}
	}
#if SUPPORT_ACCELERATE_TABLES
	if ( sAllCacheImagesProxy != NULL ) {
		dyld_image_info	infos[allImagesCount()+1];
		unsigned cacheCount = sAllCacheImagesProxy->appendImagesToNotify(dyld_image_state_bound, true, infos);
		for (unsigned i=0; i < cacheCount; ++i) {
			dyld3::ScopedTimer timer(DBG_DYLD_TIMING_FUNC_FOR_ADD_IMAGE, (uint64_t)infos[i].imageLoadAddress, (uint64_t)(*func), 0);
			(*func)(infos[i].imageLoadAddress, sSharedCacheLoadInfo.slide);
		}
	}
#endif
}
```

可以看到每当调用`_dyld_register_func_for_add_image`就会将新添加的方法`push`到队列中，然后遍历所有的镜像执行回调，这点很关键，大家理解下；另外，根据该方法的注释`Later, it is called as each new image  is loaded and bound (but initializers not yet run)`，也就是有新的镜像被加载和绑定的时候这个方法也会被调用。

## 三、fishhook实现原理

而我们的`fishhook`做的事情就是改变上面的`la-symbol-ptr`中`strlen`地址，从而到达`hook`c函数的目的,修改模型如下

![15546032593634.jpg](https://upload-images.jianshu.io/upload_images/3279997-c6b347abe5aba2f9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


整个`fishhook`的代码都在找到对应的地址然后替换掉，那么它是怎么分别找到`la-symbol-ptr`中`strlen`的地址然后替换的呢，`orig_strlen`和`my_strlen`简答编译好了，地址就确定了，那么难点就是找到`strlen`的真实地址以及`la-symbol-ptr`中的桩地址，可以拆分下任务，我们要做下面几件事情：

1. 要找到`la-symbol-ptr`中的`strlen`地址；
2. 找到`strlen`真实地址；
3. 然后按照上图进行交换。

而难点就在找到`la-symbol-ptr`中的`strlen`地址，`fishhook`是怎么找到的呢，搞懂这个也就理解了`fishhook`了,这个是`fishhook`库贴出的如何查找`la-symbol-ptr`中的`strlen`的图

![image](http://upload-images.jianshu.io/upload_images/3279997-9ae6f3dcaca12163?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240);

可以看出有3张表分别为：`Symbol Table`、`Indirect Table`、`String Table`，整个查找流程如下：

1. 通过`load Command`找到`la-symbol-ptr`，然后遍历通过`reserved1`找到某行在 `Indirect Table`中的位置；
2. 通过`Indirect Table`中的`String Table Index` 去`String Table`找到找的那行对应的符号名称；
3. `String Table`找到名称返回，如果和要替换的名称匹配那么就进行替换


关于如何计算基地址然后源码的逐行解读，网上有很多文章，可以自行搜索。

## 四、fishhook的缺陷

关于使用到的外部符号可以通过`fishhook`来实现`hook`，但是对于自己写的c函数还有用吗，试一下就知道了


```c++
static int (*orig_foo)(void);

int foo() {
    printf("___%s__\n",__func__);
    return 0;
}

int my_foo() {
    printf("my_foo");
   return  orig_foo();
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {

        struct rebinding strlen_rebinding = { "foo", my_foo,
            (void *)&orig_foo };

        rebind_symbols((struct rebinding[1]){ strlen_rebinding }, 1);

        foo();
}
```

会发现输出

```
___foo__
```

并没有起作用，说明`fishhook`对我们自己写的c函数并没有作用，其实也很好理想，按照`fishhook`的代码实现，我们自己的这个`foo`根本就不在`la-symbol-ptr`里面怎么可能有用呢


## 五、手动实现fishhook功能

我们知道了`fishhook`到底是做什么，可以借助`MachOView`来自己模拟下这个流程,先修改下代码,这次我们并没有使用`fishhook`框架，就是一个简单的系统调用


```
static int (*orig_strlen)(const char *__s);
int my_strlen(const char *__s) {
    printf("===\n");
    return orig_strlen(__s);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        char *str = "HelloWorld";
        printf("%d\n",strlen(str));
}
```

除了10以外什么都没输出


```
10
```

将刚才的`Macho`文件复制到任意文件夹中方便操作,接下来就使用`MachOView`和`hopper`来操作,通过`hopper`看到`my_strlen`和`orig_strlen`地址分别是`0x100001c50`、`0x100002098`

![15546053131967.jpg](https://upload-images.jianshu.io/upload_images/3279997-18023bcf57cf5c28.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![15546053725255.jpg](https://upload-images.jianshu.io/upload_images/3279997-daa9b46dbabefb1a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先把`_strlen`地址替换为`my_strlen`的地址

![15546054459513.jpg](https://upload-images.jianshu.io/upload_images/3279997-090b3b971895f67e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再将`orig_strlen`的地址改为`0x7fff684e8700`

![15546055900833.jpg](https://upload-images.jianshu.io/upload_images/3279997-148eacf85e421ad8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


保存为一个新的`Macho`,命名为`TestFishhook_change`

![15546056316438.jpg](https://upload-images.jianshu.io/upload_images/3279997-c2d9cc62fed2071e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


执行


```c++
➜  Documents ./TestFishhook_change
===
10

```

会发现没有使用`fishhook`也实现了，哈哈，这就是软件破解了，这里用来给大家加深下理解。

## 六、一些疑问

### 疑问一

对于`fishhook`的实现个人觉得还是有点想法的，通过加一些log可以看出，它并没有确保符号已经替换成功的情况下进行替换


```c
before:
         【0】 (/Users/fangshufeng/Library/Developer/Xcode/DerivedData/TestFishhook-fwswwhkwaochseaznkjwburwtjaz/Build/Products/Debug/TestFishhook)
         【strlen地址】：0x100002088---【strlen存放的值】：0x100001dd4
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x0
         【my_strlen函数地址】：0x100001bc0
after rebind :
         【strlen地址】：0x100002088---【strlen存放的值】：0x100001bc0
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x100001dd4
         【my_strlen函数地址】：0x100001bc0


before:
         【1】 (/Applications/Xcode.app/Contents/Developer/usr/lib/libBacktraceRecording.dylib)
         【strlen地址】：0x1000f31e8---【strlen存放的值】：0x7fff684e8700
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x100001dd4
         【my_strlen函数地址】：0x100001bc0
after rebind :
         【strlen地址】：0x1000f31e8---【strlen存放的值】：0x100001bc0
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x7fff684e8700
         【my_strlen函数地址】：0x100001bc0


before:
         【2】 (/Applications/Xcode.app/Contents/Developer/usr/lib/libMainThreadChecker.dylib)
         【strlen地址】：0x100128290---【strlen存放的值】：0x7fff684e8700
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x7fff684e8700
         【my_strlen函数地址】：0x100001bc0
after rebind :
         【strlen地址】：0x100128290---【strlen存放的值】：0x100001bc0
         【orig_strlen地址】：0x1000020c0---【orig_strlen存放的值】：:0x7fff684e8700
         【my_strlen函数地址】：0x100001bc0
```

以`strlen`为例，这个能够成功的原因是因为从第一个镜像`libBacktraceRecording.dylib`开始，`_strlen`就已经正常绑定了，如果其他的库中没有绑定呢，那不就是一个无效的操作了吗？？？


### 疑问二

我看了下这个工程加载的镜像有几百个，他的做法是遍历了所有，这个性能感觉一般，其实我们要做的是rebind主程序的符号，要去遍历那么多的库实在有些不必要，建议如下，何不在使用之前调用下原来的方法比如：


```c++
   char *s = "ssssss";
   strlen(s); // 先调用一遍确保已经绑定成功
   struct rebinding strlen_rebinding = { "strlen", my_strlen,
       (void *)&orig_strlen };

   rebind_symbols((struct rebinding[1]){ strlen_rebinding }, 1);

   char *str = "HelloWorld";
   printf("%d\n",strlen(str));
```

然后

```c++
static void rebind_symbols_for_image(struct rebindings_entry *rebindings,
                                     const struct mach_header *header,
                                     intptr_t slide) {
    indexx ++;
 ...
    // 符号表
  nlist_t *symtab = (nlist_t *)(linkedit_base + symtab_cmd->symoff);
    
// 字符串表
  char *strtab = (char *)(linkedit_base + symtab_cmd->stroff);

  // Get indirect symbol table (array of uint32_t indices into symbol table)
    // 间接符号表
  uint32_t *indirect_symtab = (uint32_t *)(linkedit_base + dysymtab_cmd->indirectsymoff);

  if (indexx > 0) {// 这里只是打个比方，真实代码当然不是这样写
        return;
 }
    
  cur = (uintptr_t)header + sizeof(mach_header_t);
  ...
}
```

因为看`dyld`可以知道，排在第一位的就是我们的主程序，在确保了符号已经绑定成功的时候再只判断主程序的符号进行替换，就不需要变量所有的镜像岂不是更快？？？

### 疑问三

基于疑问二，真的需要使用方法`_dyld_register_func_for_add_image`来实现吗，下面的伪代码应该也是可以实现的


```

// 1 调用原来的方法让符合到真实地址
// 2 获取主程序，拿到上面提到的各种表进行判断
// 3 正常使用

```

欢迎大家给我留言讨论 


[代码地址](https://github.com/fangshufeng/demo_project/tree/master/TestFishhook)


























