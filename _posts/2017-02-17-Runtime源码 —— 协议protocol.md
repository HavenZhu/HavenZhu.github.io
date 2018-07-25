---
layout: post
title: Runtime源码 —— 协议protocol
category: Runtime源码阅读
tags: [Runtime]
---

之前已经讲过方法加载的全过程，protocol的加载过程与method是一样的，就不再赘述了。不清楚的可以参考前面写的方法加载的过程

那么这篇说些啥呢？
1. protocol在runtime层的表示
2. 举例验证
3. 分析和protocol相关的常用方法的源代码

总体来说protocol相关的内容还是比较简单的，也可能是因为前面分析method的时候打好了基础。

## protocol在runtime层的表示
protocol在runtime层表示为protocol_t，这个结构体是这样的：
```
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;
    ...
}
```
所以protocol本质上也是一个对象，结构体的参数还是挺符合直觉的，一个个说一下。
- mangledName和_demangledName
这个东西来源于c++的name mangling（命名重整）技术，在c++里面是用来区别重载时的函数。

- instanceMethods和optionalInstanceMethods
对应的是实例方法，可选实例方法，可选就是写在@optional之后的方法。

- classMethods和optionalClassMethods
与上面对应，分别是类方法，可选类方法

- instanceProperties
实例属性。奇怪的是这里为什么不区分必须还是可选？

- _classProperties
类属性。挺少见的，举个例子：
```
// 这是常见的属性声明，也就是对象属性
@property (nonatomic, assign) NSInteger count;
// 这是类属性，与类方法一样，通过类名调用
@property (class, nonatomic, copy) NSString *name;
```
- protocols
此协议遵循的协议

结构体本身并不复杂，也没有什么嵌套的数据结构。

## 例子

为了验证protocol的结构，创建一个OS X的项目，添加如下的类：
```
// ZNObject.h
#import <Foundation/Foundation.h>

@protocol ZNProtocol <NSObject>
- (void)protocolMethod;
@end

@protocol ZNProtocolLow <ZNProtocol, NSObject>
- (void)protocolMethodLow;
@end

@interface ZNObject : NSObject

@property (nonatomic, weak) id<ZNProtocolLow> delegate;
- (void)hello;

@end

// ZNObject.m
#import "ZNObject.h"

@implementation ZNObject

- (void)hello {
    [self.delegate protocolMethod];
    [self.delegate protocolMethodLow];
}

@end
```
修改ViewController方法中的viewDidLoad()，其他方法略去：
```
@interface ViewController() <ZNProtocolLow>
@end

- (void)viewDidLoad {
    [super viewDidLoad];

    ZNObject *znObject = [ZNObject new];
    NSLog(@"%p", [ViewController class]);
    
(*) znObject.delegate = self;
    [znObject hello];
}
```
在有(*)标识的地方添加断点，运行程序进入断点，这个时候通过lldb进行操作：
```
// ViewController在内存中的地址
2017-02-17 16:13:29.436610 TestOSX[37980:1706893] 0x100003090
// 如果这一步看不懂，请参考本文最开始给出的方法加载的那篇文章
(lldb) p (class_data_bits_t *)0x1000030b0
(class_data_bits_t *) $0 = 0x00000001000030b0
(lldb) p $0->data()
(class_rw_t *) $1 = 0x00006080000753c0
// 获取ViewController遵循的protocol
(lldb) p $1->protocols
(protocol_array_t) $2 = {
  list_array_tt<unsigned long, protocol_list_t> = {
     = {
      list = 0x00000001000025a0
      arrayAndFlag = 4294976928
    }
  }
}
(lldb) p $2.beginLists()
(protocol_list_t **) $3 = 0x00006080000753e0
(lldb) p **$3
(protocol_list_t) $4 = (count = 1, list = protocol_ref_t [] @ 0x00007f7f52d301d8)
(lldb) p $4.list[0]
(protocol_ref_t) $5 = 4294980080
(lldb) p (protocol_t *)$5
(protocol_t *) $6 = 0x00000001000031f0
// 打印协议的名字，可以看到就是ZNProtocolLow
// 打印protocol_t的全部内容，可以看到所属类是Protocol
(lldb) p *$6
(protocol_t) $7 = {
  objc_object = {
    isa = {
      cls = Protocol
      bits = 4299890208
       = {
        nonpointer = 0
        has_assoc = 0
        has_cxx_dtor = 0
        shiftcls = 537486276
        magic = 0
        weakly_referenced = 0
        deallocating = 0
        has_sidetable_rc = 0
        extra_rc = 0
      }
    }
  }
  mangledName = 0x0000000100001ab5 "ZNProtocolLow"
  protocols = 0x0000000100002558
  instanceMethods = 0x0000000100002578
  classMethods = 0x0000000000000000
  optionalInstanceMethods = 0x0000000000000000
  optionalClassMethods = 0x0000000000000000
  instanceProperties = 0x0000000000000000
  size = 96
  flags = 0
  _extendedMethodTypes = 0x0000000100002598
  _demangledName = 0x0000000000000000 <no value available>
  _classProperties = 0x0000000000000000
}
// 获取实例方法
(lldb) p $7.instanceMethods
(method_list_t *) $8 = 0x0000000100002578
// 打印方法，结果与定义在ZNProtocolLow中的方法相同
(lldb) p $8->get(0)
(method_t) $9 = {
  name = "protocolMethodLow"
  types = 0x0000000100001af9 "v16@0:8"
  imp = 0x0000000000000000
}
// 获取此协议遵循的协议
(lldb) p $8.protocols
(protocol_list_t *) $9 = 0x0000000100002558
(lldb) p *$9
(protocol_list_t) $10 = (count = 2, list = protocol_ref_t [] @ 0x00007f7f552f9878)
// 打印出名字，与ZNProtocolLow的定义是符合的
(lldb) p ((protocol_t *)($10.list[0]))->mangledName
(const char *) $25 = 0x0000000100001ac3 "ZNProtocol"
(lldb) p ((protocol_t *)($11.list[1]))->mangledName
(const char *) $26 = 0x0000000100001ace "NSObject"
```
验证的结果和预期的一样，都还是比较简单的。

## 常用方法的源代码

先来看一下在分析protocol_t结构体时的一个疑问，就是属性为什么不区分是不是可选的？
- protocol_copyPropertyList()
runtime给我们提供了两个方法
```
objc_property_t *
protocol_copyPropertyList(Protocol *proto, unsigned int *outCount)
{
    return protocol_copyPropertyList2(proto, outCount, 
                                      YES/*required*/, YES/*instance*/);
}
// 第一个方法啥事也没干，就调用了第二个方法
objc_property_t *
protocol_copyPropertyList2(Protocol *proto, unsigned int *outCount, 
                           BOOL isRequiredProperty, BOOL isInstanceProperty)
{
    if (!proto  ||  !isRequiredProperty) {
        // Optional properties are not currently supported.
        if (outCount) *outCount = 0;
        return nil;
    }

    rwlock_reader_t lock(runtimeLock);

    property_list_t *plist = isInstanceProperty
        ? newprotocol(proto)->instanceProperties
        : newprotocol(proto)->classProperties();
    return (objc_property_t *)copyPropertyList(plist, outCount);
}
```
这里就奇怪了，看到那行注释了吗，可选属性现在还不支持，但是在NSObject协议中明明白白的有这么一段：
```
@optional
@property (readonly, copy) NSString *debugDescription;
```
写个例子测试了一下，即使在@property之前加一个@optional，在获取的时候还是会当做@required，也就是@optional对其后的属性是不起作用的。

- conformsToProtocol()
```
 +(BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}

 -(BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```
对类或者对象调用此方法，作用是一样的，都是取class进行处理：
```
BOOL class_conformsToProtocol(Class cls, Protocol *proto_gen)
{
    protocol_t *proto = newprotocol(proto_gen);
    
    if (!cls) return NO;
    if (!proto_gen) return NO;

    rwlock_reader_t lock(runtimeLock);
    assert(cls->isRealized());

    for (const auto& proto_ref : cls->data()->protocols) {
        protocol_t *p = remapProtocol(proto_ref);
        if (p == proto || protocol_conformsToProtocol_nolock(p, proto)) {
            return YES;
        }
    }

    return NO;
}
```
实现很简单，把class的protocols取出来，并与传入的protocol做比较，如果地址相同直接返回，或者协议"继承"的层级中满足条件：
```
static bool 
protocol_conformsToProtocol_nolock(protocol_t *self, protocol_t *other)
{
    runtimeLock.assertLocked();

    if (!self  ||  !other) {
        return NO;
    }

    // protocols need not be fixed up
    if (0 == strcmp(self->mangledName, other->mangledName)) {
        return YES;
    }

    if (self->protocols) {
        uintptr_t i;
        for (i = 0; i < self->protocols->count; i++) {
            protocol_t *proto = remapProtocol(self->protocols->list[i]);
            if (0 == strcmp(other->mangledName, proto->mangledName)) {
                return YES;
            }
            if (protocol_conformsToProtocol_nolock(proto, other)) {
                return YES;
            }
        }
    }

    return NO;
}
```
递归处理，对比协议的mangledName，有相同的就返回YES。这个方法总体流程还是很中规中矩的。

protocol的方法还有不少，这里就不罗列了，感兴趣的自己翻出源码看一看吧。

## 总结

本来还想写一写protocol方法的调用流程，因为也很符合直观理解，就不细说了，说白了对protocol方法的调用最终都会转换成遵循该协议的类对方法的调用。

protocol总体还是很简单的。下一篇准备看一看property和iVar。