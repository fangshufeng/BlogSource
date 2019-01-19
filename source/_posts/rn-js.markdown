---
layout: post
title: "梳理ReactNative中oc到js的数据流动"
date: 2017-09-13 13:54:37 +0800
comments: true
categories: 
tag:
    React-Native
---



以添加`CalendarManager`为例

<!--more-->

.h

```
#import <React/RCTBridgeModule.h>
#import <React/RCTLog.h>

#import <Foundation/Foundation.h>

@interface CalendarManager : NSObject<RCTBridgeModule>

@end

```

.m

```

@implementation CalendarManager

- (instancetype)init {
  if (self = [super init]) {
    
  }
  return self;
}
RCT_EXPORT_MODULE();
RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location)
{
  // Date is ready to use!
  NSLog(@"location1-----%@",location);
}
RCT_EXPORT_METHOD(fangshufeng:(NSString *)name testTwo:(NSString *)location)
{
  // Date is ready to use!
  NSLog(@"location3-----%@",location);
}
RCT_REMAP_METHOD(fangshufengJsName,
                 addEvent:(NSString *)name testTwo:(NSString *)location hh:(NSString*)jj) {
  NSLog(@"location4-----%@",location);
}
@end

```

运行xcode,在下面的地方打上断点

![15033061710856.jpg](http://upload-images.jianshu.io/upload_images/3279997-47763797c0a31b10.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


输出`configJSON`，只保留`CalendarManager`相关的并且json一下就是

```
{
    "remoteModuleConfig": [
        [
            "CalendarManager",
            null,
            [
                "addEvent", // 这里对应的就是第一个方法addEvent:(NSString *)name location:(NSString *)location 下面两个分别对应第二第三个
                "fangshufeng",
                "fangshufengJsName"
            ],
            null,
            null
        ],
        ...
    ]
}
```

这是在debug环境的情况下传的是完整配置信息，而在release环境下这个配置信息就只有module信息，而其他的方法信息都是通过懒加载的形式得到

oc通过方法将jsonstring传给js

```
  [_javaScriptExecutor injectJSONText:configJSON
                  asGlobalObjectNamed:@"__fbBatchedBridgeConfig"
                             callback:onComplete];
```

js如何接收呢

在`NativeModules.js`中

```
 const bridgeConfig = global.__fbBatchedBridgeConfig;
```

通过一个global的key`__fbBatchedBridgeConfig`将数据给到js

由

```
let NativeModules : {[moduleName: string]: Object} = {};
```
可以知道`NativeModules`本身是一个对象


我们再`debugger`

```
(bridgeConfig.remoteModuleConfig || []).forEach((config: ModuleConfig, moduleID: number) => {
    // Initially this config will only contain the module name when running in JSC. The actual
    // configuration of the module will be lazily loaded.
    const info = genModule(config, moduleID);
    debugger;
    if (!info) {
      return;
    }

    if (info.module) {
      NativeModules[info.name] = info.module;
    }
    // If there's no module config, define a lazy getter
    else {
      defineLazyObjectProperty(NativeModules, info.name, {
        get: () => loadModule(info.name, moduleID)
      });
    }
  });
}
```
可以过滤出`CalendarManager`如下图所示


![](http://upload-images.jianshu.io/upload_images/3279997-447f36f2a1442a40.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体看看这个方法做的事情可以知道每个item都会经过`genModule`方法，先来看看

```
function genModule(config: ?ModuleConfig, moduleID: number): ?{name: string, module?: Object} {
  if (!config) {
    return null;
  }

  const [moduleName, constants, methods, promiseMethods, syncMethods] = config;
  invariant(!moduleName.startsWith('RCT') && !moduleName.startsWith('RK'),
    'Module name prefixes should\'ve been stripped by the native side ' +
    'but wasn\'t for ' + moduleName);

  if (!constants && !methods) {
    // Module contents will be filled in lazily later
    return { name: moduleName };
  }

  const module = {};
  methods && methods.forEach((methodName, methodID) => {
    const isPromise = promiseMethods && arrayContains(promiseMethods, methodID);
    const isSync = syncMethods && arrayContains(syncMethods, methodID);
    invariant(!isPromise || !isSync, 'Cannot have a method that is both async and a sync hook');
    const methodType = isPromise ? 'promise' : isSync ? 'sync' : 'async';
    module[methodName] = genMethod(moduleID, methodID, methodType);
  });
  Object.assign(module, constants);

  if (__DEV__) {
    BatchedBridge.createDebugLookup(moduleID, moduleName, methods);
  }

  return { name: moduleName, module };
}

```

这个又用到了`genMethod`方法，好吧先来看看

```
function genMethod(moduleID: number, methodID: number, type: MethodType) {
  let fn = null;
  if (type === 'promise') {
    fn = function(...args: Array<any>) {
      return new Promise((resolve, reject) => {
        BatchedBridge.enqueueNativeCall(moduleID, methodID, args,
          (data) => resolve(data),
          (errorData) => reject(createErrorFromErrorData(errorData)));
      });
    };
  } else if (type === 'sync') {
    fn = function(...args: Array<any>) {
      return global.nativeCallSyncHook(moduleID, methodID, args);
    };
  } else {
    fn = function(...args: Array<any>) {
      console.log('----', args , 'moduleID:--' +  moduleID , 'methodID:---' + methodID);
      const lastArg = args.length > 0 ? args[args.length - 1] : null;
      const secondLastArg = args.length > 1 ? args[args.length - 2] : null;
      const hasSuccessCallback = typeof lastArg === 'function';
      const hasErrorCallback = typeof secondLastArg === 'function';
      hasErrorCallback && invariant(
        hasSuccessCallback,
        'Cannot have a non-function arg after a function arg.'
      );
      const onSuccess = hasSuccessCallback ? lastArg : null;
      const onFail = hasErrorCallback ? secondLastArg : null;
      const callbackCount = hasSuccessCallback + hasErrorCallback;
      args = args.slice(0, args.length - callbackCount);
      BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
    };
  }
  fn.type = type;
  return fn;
}
```

方法分为三种，由oc传给js的时候也有体现的这里我们先直接看`aync`情况的，这里做了三件事情

1. 定义了一个接收多个参数额方法；
2. 在参数列表中取出最后的两个作为js的回调；
3. 最终调用的是原生的方法这里又引出了另一个方法`BatchedBridge.enqueueNativeCall`


```
enqueueNativeCall(moduleID: number, methodID: number, params: Array<any>, onFail: ?Function, onSucc: ?Function) {
    if (onFail || onSucc) {
      if (__DEV__) {
        const callId = this._callbackID >> 1;
        this._debugInfo[callId] = [moduleID, methodID];
        if (callId > DEBUG_INFO_LIMIT) {
          delete this._debugInfo[callId - DEBUG_INFO_LIMIT];
        }
      }
      onFail && params.push(this._callbackID);
      /* $FlowFixMe(>=0.38.0 site=react_native_fb,react_native_oss) - Flow error
       * detected during the deployment of v0.38.0. To see the error, remove
       * this comment and run flow */
      this._callbacks[this._callbackID++] = onFail;
      onSucc && params.push(this._callbackID);
      /* $FlowFixMe(>=0.38.0 site=react_native_fb,react_native_oss) - Flow error
       * detected during the deployment of v0.38.0. To see the error, remove
       * this comment and run flow */
      this._callbacks[this._callbackID++] = onSucc;
    }

    if (__DEV__) {
      global.nativeTraceBeginAsyncFlow &&
        global.nativeTraceBeginAsyncFlow(TRACE_TAG_REACT_APPS, 'native', this._callID);
    }
    this._callID++;

    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);

    if (__DEV__) {
      // Any params sent over the bridge should be encodable as JSON
      JSON.stringify(params);

      // The params object should not be mutated after being queued
      deepFreezeAndThrowOnMutationInDev((params:any));
    }
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    if (global.nativeFlushQueueImmediate &&
        (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS ||
         this._inCall === 0)) {
      var queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      global.nativeFlushQueueImmediate(queue);
    }
    Systrace.counterEvent('pending_js_to_native_queue', this._queue[0].length);
    if (__DEV__ && this.__spy && isFinite(moduleID)) {
      this.__spy(
        { type: TO_NATIVE,
          module: this._remoteModuleTable[moduleID],
          method: this._remoteMethodTable[moduleID][methodID],
          args: params }
      );
    } else if (this.__spy) {
      this.__spy({type: TO_NATIVE, module: moduleID + '', method: methodID, args: params});
    }
  }
```
这个方法一共做了以下几件事情：

1. 判断如果后面两个参数有回调的话就会为每个回调生成一个唯一的ID这里叫做`_callbackID`，并将js方法装进一个`_callbacks`的数组中；
2. 这里有个`_queue`对象他的结构如下

    ```
        _queue: [Array<number>, Array<number>, Array<any>, number];
        
        在js中分别以`MODULE_IDS` `METHOD_IDS` 和 `PARAMS`来表示数组的第0、1、2号元素，之所以这样命名是达到为了见名之意的效果
    ```
    完成了第一步的`_callbackID`的映射，在分别吧对应的模块id`moduleID`、方法id`methodID`和参数`params`分别装进`_queue`对应的数组中
    
    ```
    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    this._queue[PARAMS].push(params);
    ```
3. 回调原生的`global.nativeFlushQueueImmediate`方法，对应原生的

    
    ```
    context[@"nativeFlushQueueImmediate"] = ^(NSArray<NSArray *> *calls){
      RCTJSCExecutor *strongSelf = weakSelf;
      if (!strongSelf.valid || !calls) {
        return;
      }
      
      RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"nativeFlushQueueImmediate", nil);
      [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
      RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"js_call");
    };

    ```
    
值得注意的是这里只是做方法的申明，并没有调用。

那我们在回到最开始的`genModule`方法，也就是说经过`genModule`方法以后以我们的`CalendarManager`来说的话也就是返回一个下面形式的对象

```
{
    name:"CalendarManager",
    {
       addEvent: {
             fn(...){},
             type:"async"
        },
        fangshufeng: {
             fn(...){},
             type:"async"
        },
        fangshufengJsName: {
             fn(...){},
             type:"async"
        }
    }
}
```

再回到一开始的方法

```
if (info.module) {
      NativeModules[info.name] = info.module;
    }
```
从而经过一大圈的变换最后得到的结果就是返回一个`NativeModules`对象，还是以`CalendarManager`为例，得到的`NativeModules`的对象如下

```
CalendarManager: 
    {
       addEvent: {
             fn(...){},
             type:"async"
        },
        fangshufeng: {
             fn(...){},
             type:"async"
        },
        fangshufengJsName: {
             fn(...){},
             type:"async"
        }
    }
```
我们自定义的`CalendarManager`作为了`NativeModules`的一个属性，`CalendarManager`所对应的方法列表成了`CalendarManager`属性
这里只是用`CalendarManager`为例子来讲的，实际上最后`NativeModules`是有很多的类似`CalendarManager`的属性

![15033248335271.jpg](http://upload-images.jianshu.io/upload_images/3279997-7f11269582b19ee9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是说每一个原生模块都对应`NativeModules`一个属性


到这里也就说完了原生创建一个模块到js的演变过程，我们来总结一下，当我们要创建一个原生模块的时候需要做些什么；

1. 要继承<RCTBridgeModule>这个协议；
2. 要在.m中加入RCT_EXPORT_MODULE();还记得之前那个发给js的那个原生模块的jsonString吧这个目的其实是要将当前模块加入配置表中也就是下面的这个方法
    
    ```
    void RCTRegisterModule(Class moduleClass)
    {
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
        RCTModuleClasses = [NSMutableArray new];
      });
    
      RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
                @"%@ does not conform to the RCTBridgeModule protocol",
                moduleClass);
    
      // Register module
      [RCTModuleClasses addObject:moduleClass];
    }
    ```
3. 对于要暴露给js的oc方法需要使用`RCT_EXPORT_METHOD`这个宏
   
    ```
    RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location)
    ```
    
    展开以后就是下面这个样子
    
    ```
    + (NSArray<NSString *> *)__rct_export__300 { 
            return @[@"", 
            @"addEvent:(NSString *)name location:(NSString *)location "]; 
            } 
    - (void)addEvent:(NSString *)name location:(NSString *)location;
    ```
    到后面要查找方法列表的时候就是通过`__rct_export__`前缀来查找的，查找的方法如下
    
    ```
    - (NSArray<id<RCTBridgeMethod>> *)methods
{
  if (!_methods) {
    NSMutableArray<id<RCTBridgeMethod>> *moduleMethods = [NSMutableArray new];

    if ([_moduleClass instancesRespondToSelector:@selector(methodsToExport)]) {
      [moduleMethods addObjectsFromArray:[self.instance methodsToExport]];
    }

    unsigned int methodCount;
    Class cls = _moduleClass;
    while (cls && cls != [NSObject class] && cls != [NSProxy class]) {
      Method *methods = class_copyMethodList(object_getClass(cls), &methodCount);

      for (unsigned int i = 0; i < methodCount; i++) {
        Method method = methods[i];
        SEL selector = method_getName(method);
        if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
          IMP imp = method_getImplementation(method);
          NSArray<NSString *> *entries =
            ((NSArray<NSString *> *(*)(id, SEL))imp)(_moduleClass, selector);
          id<RCTBridgeMethod> moduleMethod =
            [[RCTModuleMethod alloc] initWithMethodSignature:entries[1]
                                                JSMethodName:entries[0]
                                                 moduleClass:_moduleClass];

          [moduleMethods addObject:moduleMethod];
        }
      }

      free(methods);
      cls = class_getSuperclass(cls);
    }

    _methods = [moduleMethods copy];
  }
  return _methods;
}
    ```
    这个方法做了两个事情，一个是查找已`__rct_export__`开头的方法，一个是以`__rct_export__`方法开头的第一个参数为js方法的名称。

4. 由3可知以这个宏`RCT_EXPORT_METHOD`的话默认给js的方法名称是参数的第一位，可以知道都是空的，可是我们最后是需要给js一个方法一个名称的这里oc做了默认处理了

    ```
    - (NSString *)JSMethodName
{
  NSString *methodName = _JSMethodName;
  if (methodName.length == 0) {
    methodName = _methodSignature;
    NSRange colonRange = [methodName rangeOfString:@":"];
    if (colonRange.location != NSNotFound) {
      methodName = [methodName substringToIndex:colonRange.location];
    }
    methodName = [methodName stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
    RCTAssert(methodName.length, @"%@ is not a valid JS function name, please"
              " supply an alternative using RCT_REMAP_METHOD()", _methodSignature);
  }
  return methodName;
}

    ```
    也就是如果`JSMethodName`的长度为空的话就会截取第二个参数的`:`以前的字符串作为方法，也就是
    `RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location)`对应`addEvent`，`RCT_EXPORT_METHOD(fangshufeng:(NSString *)name testTwo:(NSString *)location)`对应`fangshufeng`，那这样的话问题就来了，万一还有个这样的方法`RCT_EXPORT_METHOD(fangshufeng:(NSString *)name)`按照默认的规则岂不是就和`RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location)`冲突了吗，rn为了避免这种情况，我们可以使用这个宏`RCT_REMAP_METHOD`，也就是这种用法
    
    ```
    RCT_REMAP_METHOD(fangshufengJsName,
                 addEvent:(NSString *)name testTwo:(NSString *)location hh:(NSString*)jj) {
  NSLog(@"location4-----%@",location);
}
    ```
    这样的话之前那个数组的参数第一个值就有值了对应`fangshufengJsName`
5. 在研究js源码的时候我们发现，js对参数的解析是有要注意的地方的，如果原生想要使用js的回调的话只能放在参数的后两位

    ```
    RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location action:(RCTResponseSenderBlock)action)
{
  // Date is ready to use!
  NSLog(@"location1-----%@",location);
}
    ```
    这样的话就是错的
    
    ```
    RCT_EXPORT_METHOD(addEvent:(NSString *)name   action:(RCTResponseSenderBlock)action location:(NSString *)location)
    {
      // Date is ready to use!
      NSLog(@"location1-----%@",location);
    }
    ```

接下来将要讲的是当自定义的模块方法被调用时数据是如何流动的。

首先我们先把`CalendarManager`用起来

```
import  React,{Component} from 'react';
import {
    NativeModules,
    Text,
    TouchableOpacity,
    NativeAppEventEmitter,
    View,
} from 'react-native'

import  FadeInView from './FadeInView'

let CalendarManager = NativeModules.CalendarManager;

export  default  class TestAnimal extends  Component {

    render() {
        return (
            <TouchableOpacity
                 onPress={() => {CalendarManager.fangshufeng('name','ggg')}}
            >
            <FadeInView style={{width: 250, height: 50, backgroundColor: 'powderblue'}}

            >
                <Text style={{fontSize: 28, textAlign: 'center', margin: 10}}>Fading in</Text>
            </FadeInView>
            </TouchableOpacity>
        )
    }

}
```

运行的结果

![15033660664537.jpg](http://upload-images.jianshu.io/upload_images/3279997-779325b8ae47b29e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击`fading in `也就是调用`fangshufeng`方法，之前我们说过`enqueueNativeCall`方法只是做方法申明，这个时候就是调用之前定义的方法，在我的这个工程里面可以debug到，`fangshufeng`这个方法的`moduleID`是`1`，`methodID`是`1`

![15033664429808.jpg](http://upload-images.jianshu.io/upload_images/3279997-b5a19aa23f5ecf1f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后debug到`enqueueNativeCall`中

![15033665264435.jpg](http://upload-images.jianshu.io/upload_images/3279997-7ad874b2227b4e87.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到最终传给oc的`queue`是什么

![15033665894650.jpg](http://upload-images.jianshu.io/upload_images/3279997-bd53b0911643cf7d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果只看`CalendarManager`的话当你点击`Fading in`的时候传给oc的queue就是

```
[[1],[1],[['name','ggg']]]
```

然后通过`nativeFlushQueueImmediate`方法给到oc,会进到下面的这个方法


```
- (void)handleBuffer:(NSArray *)buffer
{
  NSArray *requestsArray = [RCTConvert NSArray:buffer];

  NSArray<NSNumber *> *moduleIDs = [RCTConvert NSNumberArray:requestsArray[RCTBridgeFieldRequestModuleIDs]];
  NSArray<NSNumber *> *methodIDs = [RCTConvert NSNumberArray:requestsArray[RCTBridgeFieldMethodIDs]];
  NSArray<NSArray *> *paramsArrays = [RCTConvert NSArrayArray:requestsArray[RCTBridgeFieldParams]];

  int64_t callID = -1;

  if (requestsArray.count > 3) {
    callID = [requestsArray[RCTBridgeFieldCallID] longLongValue];
  }

    // 1
  NSMapTable *buckets = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory
                                                  valueOptions:NSPointerFunctionsStrongMemory
                                                      capacity:_moduleDataByName.count];

    //2
  [moduleIDs enumerateObjectsUsingBlock:^(NSNumber *moduleID, NSUInteger i, __unused BOOL *stop) {
    RCTModuleData *moduleData = self->_moduleDataByID[moduleID.integerValue];
    dispatch_queue_t queue = moduleData.methodQueue;
    NSMutableOrderedSet<NSNumber *> *set = [buckets objectForKey:queue];
    if (!set) {
      set = [NSMutableOrderedSet new];
      [buckets setObject:set forKey:queue];
    }
    [set addObject:@(i)];
  }];

    //3
  for (dispatch_queue_t queue in buckets) {
    dispatch_block_t block = ^{
      NSOrderedSet *calls = [buckets objectForKey:queue];
      @autoreleasepool {
        for (NSNumber *indexObj in calls) {
          NSUInteger index = indexObj.unsignedIntegerValue;
          [self callNativeModule:[moduleIDs[index] integerValue]
                          method:[methodIDs[index] integerValue]
                          params:paramsArrays[index]];
        }
      }
    };
        [self dispatchBlock:block queue:queue];
    }

  _flowID = callID;
}
```
在需要的解释的地方做了个标记

1. 由于下面要用到`object->object` 所以这里使用`map`也就是`NSMapTable`来保存数据；
2. 这个地方可能有点绕，如果对数据结构不清楚的话比较难以理解，还是以`CalendarManager`为例子，先来假设其他的数据如下所示：
    假设js传给oc的queue是这个样子的
    
    ```
    [
        [38,19,41,41,1], // 最后一个1表示`CalendarManager`模块
        [19,1,4,3,1], // 最后一个1表示addEvent: location: 方法
        [[...],[...],[...],[...],['name','ggg']] //['name','ggg']表示方面方法的入参
    ]
    ```
    
    我们通过断点来跟踪一下数据
    
    `modules`
    
    ```
    <__NSArrayI 0x7b048e30>(
    38,
    19,
    41,
    41,
    1
    )
    
    ```
    
    `buckets`
    
    ```
    po buckets
    NSMapTable {
    [10] <OS_dispatch_queue: com.facebook.react.CalendarManagerQueue[0x7b652310]> -> {(
        4
    )}
    [46] <OS_dispatch_queue: com.facebook.react.ShadowQueue[0x7b19cb90]> -> {(
        0,
        2,
        3
    )}
    [47] <null> -> {(
        1
    )}
    }
    ```
    
    
    ```
    po methodIDs
    <__NSArrayI 0x7b048da0>(
    19,
    1,
    4,
    3,
    1
    )
    ```
    
    
    ```
    po paramsArrays
    <__NSArrayI 0x7ee5f7c0>(
    <__NSArray0 0x7a648bd0>(
    
    )
    ,
    <__NSSingleObjectArrayI 0x7b04c8d0>(
    47
    )
    ,
    <__NSSingleObjectArrayI 0x7b04daf0>(
    3
    )
    ,
    <__NSArrayI 0x7ee55570>(
    4,
    3,
    {
        frames =     (
            0,
            "0.008888888888888889",
            "0.03555555555555556",
            "0.08000000000000002",
            "0.1422222222222222",
            "0.2222222222222223",
            "0.3200000000000001",
            "0.4355555555555557",
            "0.5644444444444444",
            "0.6799999999999999",
            "0.7777777777777777",
            "0.8577777777777778",
            "0.9199999999999999",
            "0.9644444444444443",
            "0.9911111111111111",
            1,
            1
        );
        iterations = 1;
        toValue = 1;
        type = frames;
    },
    13
    )
    ,
    <__NSArrayI 0x7b2630e0>(
    name,
    ggg
    )
    
    ```
    做的事情就是：
    
    遍历模块的数组 ->  在`_moduleDataByID`取出`moduleData` -> 以`moduleData`中的串行队列作为key set作为value,而set中装着的对应的索引，由于module、method、Params都是索引值是一一对应的，这个索引值即是当前module在modules数组的索引值，也是method、Params的索引值。
    
3. 上面已经给出了bucket的具体内容，循环后也就是调用下面的方法
    
    ```
    [self callNativeModule:[moduleIDs[index] integerValue]
                              method:[methodIDs[index] integerValue]
                              params:paramsArrays[index]];
    
    // 具体就是
    [self callNativeModule:1 //可以知道这个1就是`CalendarManager`
                    method:1 // 这个1就是表示addEvent: location: 方法
                    params:['name','ggg'] //表示方面方法的入参
                    ];

    //具体上面怎么解析的下面会做分析
    ```
    

oc拿到数据得到模块名称、方法名称、参数，然后执行

