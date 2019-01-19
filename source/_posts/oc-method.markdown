---
layout: post
title: "OC--如何获取变量和方法的信息"
date: 2018-08-02 23:23:41 +0800
comments: true
categories: 
tag:
    Objective-C
---



这一篇是解决[上一篇](https://www.jianshu.com/p/14b096d6a233)遗留问题中的第2个问题的，如果没有读过的请先阅读上一篇的内容

变量:有`成员变量`和`属性变量`2者是包含关系，这个上篇已经解释过了；
方法：分为`实例方法`和`类方法`；

<!-- more -->

### 准备工作

还是上一篇中的样例代码

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <malloc/malloc.h>
//#import "TestObj.h"

@interface Animal : NSObject {
    @public
    int _age;
    char _a;
    double _b;
    NSString *_name;
    
}

@property (nonatomic, copy) NSString *testProperty;

- (int)testFunction:(int)a;

+ (int)testClassFunction;

@end

@implementation Animal

- (int)testFunction:(int)a {
    
    return 10;
};

+ (int)testClassFunction {
    return 10;
};

@end



int main(int argc, const char * argv[]) {
    @autoreleasepool {

    }
    
    return 0;
}
```


### runtime方法记忆的小技巧

有时候调用`runtime`的方法的时候特别难以记忆，如果对应到`_class_ro_t`类结构，现在是不是很好记忆了

![15331809233311.jpg](https://upload-images.jianshu.io/upload_images/3279997-d6ecf4a9aa89c8b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


看上面这些提示会发现和`类对象`已经`元类对象`的结构正好对应上了

```
struct _class_ro_t {
	unsigned int flags;
	unsigned int instanceStart;
	unsigned int instanceSize;
	const unsigned char *ivarLayout;
	const char *name;
	const struct _method_list_t *baseMethods;
	const struct _objc_protocol_list *baseProtocols;
	const struct _ivar_list_t *ivars;
	const unsigned char *weakIvarLayout;
	const struct _prop_list_t *properties;
};

```
你只要对着这些数据结构敲对应的属性名称，就会找到相应的方法


### 成员变量获取

由上一篇知道成员变量的信息是保存在`类对象`中的，当然是需要通过`类对象`去拿的；

很容易得到类对象关于成员变量的数据结构

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

可以看到变量的信息都存放在装着`_ivar_t`的`ivar_list`数组里面，知道了使用`runtime`的小技巧很容易知道获取成员变量列表会找`Ivar`

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        
        unsigned  count = 0;
        Ivar *ivarList =  class_copyIvarList([animal class], &count); // 获取到成员变量的列表
        }
    
    return 0;
}
```

由`struct _ivar_t ivar_list[5];`就知道这个`ivar_list`装的都是`_ivar_t`类型的变量

`_ivar_t`如下

```
struct _ivar_t {
	unsigned long int *offset;  // pointer to ivar offset location
	const char *name;
	const char *type;
	unsigned int alignment;
	unsigned int  size;
};

```

所以我们把代码写完

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        
        unsigned  count = 0;
        
        Ivar *ivarList =  class_copyIvarList([animal class], &count);
        
        for (int i = 0;  i < count; i++) {
            Ivar ivar =  ivarList[I];
         
            NSLog(@"属性名称:%@, 偏移量:%ld,变量类型编码:%@",
                  [NSString stringWithUTF8String:ivar_getName(ivar)],
                  ivar_getOffset(ivar),
                  [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)]
                  );
        }
        free(ivarList);
    }
    
    return 0;
}
```

输出如下

```
属性名称:_age, 偏移量:8,变量类型编码:i
属性名称:_a, 偏移量:12,变量类型编码:c
属性名称:_b, 偏移量:16,变量类型编码:d
属性名称:_name, 偏移量:24,变量类型编码:@"NSString"
属性名称:_testProperty, 偏移量:32,变量类型编码:@"NSString"

```

这种方式可以拿到所有的成员变量列表以及类型编码，这2者应该没什么难理解的，也许这个偏移量咋一看不知道怎么得出这些数字的，

这里还是要回到一开始存的时候

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

以`_age`为例


```
	{(unsigned long int *)&OBJC_IVAR_$_Animal$_age, "_age", "i", 2, 4},
```

`OBJC_IVAR_$_Animal$_age`其实就是


```
extern "C" unsigned long int OBJC_IVAR_$_Animal$_age __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Animal, _age);
```

而`__OFFSETOFIVAR__`就是

```
#define __OFFSETOFIVAR__(TYPE, MEMBER) ((long long) &((TYPE *)0)->MEMBER)
```

表达的意思就是变量距离一开始的位置是多少（我们也可以通过这种方式拿到各个变量的偏移量，见文章最后的小demo），看过我的[第一篇文章](https://www.jianshu.com/p/3d699f1ae6a3)的应该知道继承`NSObject`的对象都有一个4个字节的`isa`，所以有一下结论:

1. `_age`是从第`8`字节开始的；
2. 由于`_age`占用4个字节，所以`_a`的开始位置为`8` + `4` = `12`;
3. `_b` 为 isa的`8` + age 的`4` + a的`1` 但是由于内存对齐的调整，后面的3位放不下字节长度为`4`的`_b`，所以为isa的`8` + age 的`4` + a的`1` + 空出来的`3`个字节 = `16`；
4. 剩下以此类推。。。


也就是下面这张图

![15331909124115.jpg](https://upload-images.jianshu.io/upload_images/3279997-12d9c6c165dca46e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

读者也可以根据我第一篇的文章看看具体的对象实际分配了多少内存。

### 属性变量获取

属性变量也是放在`类对象`中的所以`class_copyPropertyList`第一个参数传的是`类对象`

还是贴出对应的代码吧


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

`_prop_t`结构


```
struct _prop_t {
	const char *name;
	const char *attributes;
};

```

和成员变量一样


```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        
        unsigned  count = 0;
        
        objc_property_t *propertyList =  class_copyPropertyList([animal class], &count);

        for (int i = 0;  i < count; i++) {
            
          objc_property_t  property =  propertyList[I];
         
            NSLog(@"属性名称:%@,变量类型编码:%@",
                  [NSString stringWithUTF8String:property_getName(property)],
                  [NSString stringWithUTF8String:property_getAttributes(property)]
                  );
        }
        free(propertyList);
    }
    
    return 0;
}

```

输出


```
属性名称:testProperty,变量类型编码:T@"NSString",C,N,V_testProperty
```

### 实例方法

实例方法也是放在`类对象`中的所以`class_...`第一个参数传的也是`类对象`

实例方法代码

```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[3];
} _OBJC_$_INSTANCE_METHODS_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	3,
	{(struct objc_selector *)"testFunction:", "i20@0:8i16", (void *)_I_Animal_testFunction_},
	{(struct objc_selector *)"testProperty", "@16@0:8", (void *)_I_Animal_testProperty},
	{(struct objc_selector *)"setTestProperty:", "v24@0:8@16", (void *)_I_Animal_setTestProperty_}}
};

```

`_objc_method`结构

```
struct _objc_method {
    struct objc_selector * _cmd;
    const char *method_type;
    void  *_imp;
};
```

```

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        
        unsigned  count = 0;
        
        Method *methodList =  class_copyMethodList(object_getClass(animal), &count);

        for (int i = 0;  i < count; i++) {
            
          Method  method =  methodList[I];
            
            NSLog(@"方法名称:%@ ,方法地址:%p,方法类型编码:%@",
                 NSStringFromSelector(method_getName(method)),
                 method_getImplementation(method),
                  [NSString stringWithUTF8String:method_getTypeEncoding(method)]
                  );
        }
        free(methodList);
    }
    
    return 0;
}

```

输出

```
方法名称:testFunction: ,方法地址:0x100000ac0,方法类型编码:i20@0:8i16
方法名称:.cxx_destruct ,方法地址:0x100000b70,方法类型编码:v16@0:8
方法名称:testProperty ,方法地址:0x100000b00,方法类型编码:@16@0:8
方法名称:setTestProperty: ,方法地址:0x100000b30,方法类型编码:v24@0:8@16
```

会发现多了个`.cxx_destruct`，这个应该是框架加上去的，先不管这些，从输出结果拿到了所有的实例变方法

### 类方法

类方法是放在`元类对象`中的所以`class_...`第一个参数传的也是`元类对象`

只需要把获取`实例方法`的一行代码修改下就可以了

将`class_copyMethodList`的第一个参数，由类对象换成元类对象就可以了

```

Method *methodList =  class_copyMethodList(object_getClass(animal), &count);
```

```

Method *methodList =  class_copyMethodList(object_getClass(object_getClass(animal)), &count);
```

输出

```
 方法名称:testClassFunction ,方法地址:0x100000af0,方法类型编码:i16@0:8
```

###__OFFSETOFIVAR__

```
struct TestStruct {
    int a;
    char b;
    double name;
};

#define t__OFFSETOFIVAR__(TYPE, MEMBER) ((long long) &((TYPE *)0)->MEMBER)

int main(int argc, const char * argv[]) {
    @autoreleasepool {
       struct TestStruct temp ;

        NSLog(@"%lld",t__OFFSETOFIVAR__(struct TestStruct, a));
        NSLog(@"%lld",t__OFFSETOFIVAR__(struct TestStruct, b));
        NSLog(@"%lld",t__OFFSETOFIVAR__(struct TestStruct, name));
       }
    
    return 0;
}

```

输出


```
0
4
8
```


