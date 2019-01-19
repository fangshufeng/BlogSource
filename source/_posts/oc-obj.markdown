---
layout: post
title: "OC--实例对象、类对象和元类对象是如何分工的"
date: 2018-08-02 15:12:08 +0800
comments: true
categories: 
tag:
    Objective-C
---



本文已经默认你已经了解了`实例对象`、`类对象`和`元类对象`，关于什么是`实例对象`、`类对象`和`元类对象`，请自行了解不在本文的范围内。

### *前言

平时开发的过程中我们会创建类，然后给这个类增加属性，方法，甚至添加协议和分类，然后分类有添加方法等等，那么这些添加的东西在内存中是如何存放的呢，当我们调用某个方法的时候又是如何完成调用的呢？

<!--more-->

### *准备工作

创建一个`command line tool`项目， 编辑`main.m`内容如下

```
#import <Foundation/Foundation.h>

#import <objc/runtime.h>
#import <malloc/malloc.h>

@interface Animal : NSObject {
    @public
    int _age;
    char _a;
    double _b;
    NSString *_name;
}

@property (nonatomic, copy) NSString *testProperty;

- (int)testFunction:(int)a;

@end

@implementation Animal

- (int)testFunction:(int)a {
    return 10;
};


@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {

    }
    
    return 0;
}
```

将代码通过`clang`命令重写成 架构为arm64的`c++`代码

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m
```
会在`main.m`同层目录下生成`main.cpp`文件

### *3中对象的生成

查看`main.cpp`文件会看到三个核心的对象，分别对象`实例对象`、`类对象`和`元类对象`

`实例对象`

```
struct Animal_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	int _age;
	char _a;
	double _b;
	NSString *_name;
	NSString *_testProperty;
};
```

`类对象`，代码做了适量的删减

```
struct _class_t OBJC_CLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Animal,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Animal,
};
```

`元类对象`，代码做了适量的删减

```
struct _class_t OBJC_METACLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_NSObject,
	0, // &OBJC_METACLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_METACLASS_RO_$_Animal,
};

```

### *实例对象职责

由`main.m`函数的中`Animal`类的代码可以知道，我们一共给`Animal`对象增加了以下内容

**共有成员变量4个**

```
    int _age;
    char _a;
    double _b;
    NSString *_name;
```

**属性1个**

```
@property (nonatomic, copy) NSString *testProperty;

```

**实例方法1个**

```
- (int)testFunction:(int)a;
```

**类方法1个**

```
+ (int)testClassFunction;
```


由实例对象的结构可以知道

```
struct Animal_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	int _age;
	char _a;
	double _b;
	NSString *_name;
	NSString *_testProperty;
};

```

这个对象拥有了4个成员变量和一个`_testProperty`的属性，除了一个父类的结构体之外并无其他，得出以下结论

1. 实例对象只存放成员变量；
2. 属性会生成`_`+`属性名称`的成员变量 （这也进一步说明@property会生成_的成员变量）

至此成员变量已经有人认领了---> `实例对象`；
---


### *类对象职责

由类对象的数据结构可知是为`_class_t`类型的

```
struct _class_t {
	struct _class_t *isa; // 指向是哪个对象的实例
	struct _class_t *superclass; // 指向自己的父类
	void *cache; // #warning 暂时不清楚
	void *vtable; // 标识为未使用的
	struct _class_ro_t *ro; // 存储内容的详细信息
};

```

再看下`类对象`的结构

```
struct _class_t OBJC_CLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Animal,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Animal,
};
```

类对象的`ro`指向`_OBJC_CLASS_RO_$_Animal`地址，结构如下

```
static struct _class_ro_t _OBJC_CLASS_RO_$_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, __OFFSETOFIVAR__(struct Animal, _age), sizeof(struct Animal_IMPL), 
	0, 
	"Animal",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Animal,
	0, 
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Animal,
	0, w
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Animal,
};
```

可以知道是`_class_ro_t`类型的 `_class_ro_t`结构如下


```
struct _class_ro_t {
	unsigned int flags; // 1 标识元类对象 0 标识类对象
	unsigned int instanceStart; //  #warning所管理的实例对象的开始
	unsigned int instanceSize; // 所管理的实例对象的大小
	const unsigned char *ivarLayout; // #warning 
	const char *name; // 类的名称
	const struct _method_list_t *baseMethods; // 方法列表
	const struct _objc_protocol_list *baseProtocols; // 协议列表
	const struct _ivar_list_t *ivars; // 成员变量列表
	const unsigned char *weakIvarLayout; // #warning 
	const struct _prop_list_t *properties; // 属性列表
};
```
点击`_OBJC_$_INSTANCE_METHODS_Animal`如下


```

static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[3]; // 装着_objc_method结构的方法
} _OBJC_$_INSTANCE_METHODS_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    3,
    {(struct objc_selector *)"testFunction:", "i20@0:8i16", (void *)_I_Animal_testFunction_},
    {(struct objc_selector *)"testProperty", "@16@0:8", (void *)_I_Animal_testProperty},
    {(struct objc_selector *)"setTestProperty:", "v24@0:8@16", (void *)_I_Animal_setTestProperty_}
    } // I 表示instance
};

```

装着所有的实例方法（也可以看出@property会生成对应的set和get方法）。

`_objc_method`结构如下

```
struct _objc_method {
	struct objc_selector * _cmd; // 方法的编号
	const char *method_type; // 方法签名
	void  *_imp; // 真正函数的实现
};

```

至此，实例方法有人认领了---> `类对象`；
---

**成员变量**

指向`_OBJC_$_INSTANCE_VARIABLES_Animal`

```
static struct /*_ivar_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count;
	struct _ivar_t ivar_list[5];
} _OBJC_$_INSTANCE_VARIABLES_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_ivar_t),
	5,
	{(unsigned long int *)&OBJC_IVAR_$_Animal$_age, "_age", "i", 2, 4},
	 {(unsigned long int *)&OBJC_IVAR_$_Animal$_a, "_a", "c", 0, 1},
	 {(unsigned long int *)&OBJC_IVAR_$_Animal$_b, "_b", "d", 3, 8},
	 {(unsigned long int *)&OBJC_IVAR_$_Animal$_name, "_name", "@\"NSString\"", 3, 8},
	 {(unsigned long int *)&OBJC_IVAR_$_Animal$_testProperty, "_testProperty", "@\"NSString\"", 3, 8}}
};
```

`_ivar_t`结构如下

```
struct _ivar_t {
	unsigned long int *offset;  // 变量的相对于开始位置的偏移量
	const char *name; // 变量名称
	const char *type; // 变量的类型编码
	unsigned int alignment; // #warning我暂时没查到资料
	unsigned int  size; // s所占的字节大小
};
```
这里都是对于成员变量的描述信息

**属性变量**

指向`_OBJC_$_PROP_LIST_Animal`

```
static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"testProperty","T@\"NSString\",C,N,V_testProperty"}}
};

```


`_prop_t`结构如下


```
struct _prop_t {
	const char *name;  // 属性名称
	const char *attributes; // 属性的属性信息
};
```

发现只有`testProperty`一个属性，简单说下


```
{{"testProperty","T@\"NSString\",C,N,V_testProperty"}}
```

```
T@"NSString":表示这个属性的类型是 NSString 型
C：表示属性的引用类型是 copy
N：表示 nonatomic
V_testProperty: 表示属性生成的实例变量是：_testProperty,我们可以用_testProperty来访问属性testProperty的值。
```

**结论**
成员变量和属性变量还是有区别的  成员变量包含属性变量

只剩下类方法无人认领了，猜也猜到了---> `元类对象`


### *元类对象职责

由下面代码可以知道`METACLASS`也是`_class_t`类型的，只不过`_class_ro_t`指向的是`_OBJC_METACLASS_RO_$_Animal`的地址

```
struct _class_t OBJC_METACLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_NSObject,
	0, // &OBJC_METACLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_METACLASS_RO_$_Animal,
};

```
**_OBJC_METACLASS_RO_$_Animal** 如下


```
static struct _class_ro_t _OBJC_METACLASS_RO_$_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1, sizeof(struct _class_t), sizeof(struct _class_t), 
	0, 
	"Animal",
	(const struct _method_list_t *)&_OBJC_$_CLASS_METHODS_Animal,
	0, 
	0, 
	0, 
	0, 
};

```

**_OBJC_$_CLASS_METHODS_Animal** 如下


```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CLASS_METHODS_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"testClassFunction", "i16@0:8", (void *)_C_Animal_testClassFunction}} // C表示class
};

```

到此为止`类`方法也有人认领了---> `元类对象`
也可以看出`类对象`和`元类对象`的数据结构是一样的，只不过放的东西不一样罢了

### *3中对象如何关联的

`实例对象`

```
struct Animal_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	int _age;
	char _a;
	double _b;
	NSString *_name;
	NSString *_testProperty;
};
```

`类对象`，代码做了适量的删减

```
struct _class_t OBJC_CLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Animal,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Animal,
};
```

`元类对象`，代码做了适量的删减

```
struct _class_t OBJC_METACLASS_$_Animal __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_NSObject,
	0, // &OBJC_METACLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_METACLASS_RO_$_Animal,
};

```

有必要把上面的代码再写一遍，3中对象的数据结构以及知道了，那这三个是如何关联起来的呢，对于`类对象`和`元类对象`在内存里面只有一份，编译阶段就可以初始化了


```
static void OBJC_CLASS_SETUP_$_Animal(void ) {
	OBJC_METACLASS_$_Animal.isa = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Animal.superclass = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Animal.cache = &_objc_empty_cache;
	OBJC_CLASS_$_Animal.isa = &OBJC_METACLASS_$_Animal;
	OBJC_CLASS_$_Animal.superclass = &OBJC_CLASS_$_NSObject;
	OBJC_CLASS_$_Animal.cache = &_objc_empty_cache;
}
```

而对于`实例对象`，可以有很多多个，只能等到初始化的时候才能指定了


```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!cls->instancesRequireRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        assert(!DisableNonpointerIsa);
        assert(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        assert(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}

```

到这三者已经串联起来了
什么是isa不是本文的重点

最后会有个对外输出的指针供运行时调用


```
static struct _class_t *L_OBJC_LABEL_CLASS_$ [1] __attribute__((used, section ("__DATA, __objc_classlist,regular,no_dead_strip")))= {
	&OBJC_CLASS_$_Animal,
};
```

### *各种情况的调用

获取属性：通过`实例对象`拿的；
获取对象方法：`实例对象`通过`isa`找到对应实例的`类对象`拿的；
获取类方法：`实例对象`通过`isa`找到对应实例的`类对象`拿的；

### 遗留问题

1. `协议`、`分类`方法没有分析，读者可自行分析，后面我也会输出文章；
2. 所有的这些属性存进去了，能不能获取到呢；
3. 对外输出的指针如何被runtime调用的；






