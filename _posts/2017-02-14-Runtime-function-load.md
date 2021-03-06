---
layout: post
title: Runtime源码 —— 方法加载的过程
category: Runtime
tags: [Runtime]
---

在上一篇文章中分析过类的结构体。

是这个样子的：
```
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}
```
那一篇主要是分析isa的源码，这些字段并没有深究，这一篇就来深入研究一下。我还是会先对源码进行分析，再结合例子进行验证。

从字面上来看，前两个字段的意思是很容易理解的：
- Class superclass
父类的指针。
- cache_t cache;
一个缓存，官方文档注释写明了用于缓存指针和虚表。但这个缓存是如何起作用在后续的文章中再讲。

下面就来重点看一看这个不太看得懂的字段：
- class_data_bits_t bits
注释中讲了这个字段实际上就是class_rw_t *加上自定义的rr/alloc标志，rr/alloc标志是指含有这些方法：retain/release/autorelease/retainCount/alloc等。

那么就来看看class_rw_t这个结构体：
```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    Class firstSubclass;
    Class nextSiblingClass;
    char *demangledName;
}
```
除开明显的方法，属性，协议等字段，这个结构体中有一个奇怪的字段，const class_ro_t *ro;这个结构体是这样定义的：
```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;
    const uint8_t * ivarLayout;
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;
    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
}
```
感觉这两个结构体还是比较类似的，这时候合理猜测一下，rw应该是指readwrite，ro是指readonly。也就是说在可读可写的结构体中存放了一个只读的结构体，而且这两个结构体很相似。

结合oc是一门动态语言再猜测一下，class_ro_t是不是存放在编译期就确定的信息，class_rw_t用来存放在运行期添加的信息呢?只能通过代码来验证一下。
>注：这里面还有一些很关键的字段，比如instanceStart，instanceSize。本文不做扩展。

## 例子
首先把之前的TestObject扩展一下，添加一个hello方法：
```
// TestObject.h
#import <Foundation/Foundation.h>
@interface TestObject : NSObject

- (void)hello;

@end

// TestObject.m
#import "TestObject.h"
@implementation TestObject

- (void)hello {
    NSLog(@"hello");
}

@end
```

接着获取一下TestObject在内存中的地址：
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"%p", [TestObject class]);
    }
    return 0;
}

输出：0x100001168
```
>只要代码不变，这个类在内存中的地址就不会变

这个时候在void _objc_init(void)添加一个断点：
<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_14/1.png" />
</p>
然后利用上面的地址就可以看一看这个时候class_ro_t的内容。

```
(lldb) p (objc_class *)0x100001168
(objc_class *) $0 = 0x0000000100001168
// 根据objc_class的结构，isa:8字节，superclass:8字节，cache:16字节
// 所以偏移32字节来获取class_data_bits_t
(lldb) p (class_data_bits_t *)0x100001188
(class_data_bits_t *) $1 = 0x0000000100001188
(lldb) p $1->data()
(class_rw_t *) $2 = 0x00000001000010e8
// 在这个时候，class_rw_t实际上是class_ro_t，后面会验证
(lldb) p (class_ro_t *)$2
(class_ro_t *) $3 = 0x00000001000010e8
(lldb) p *$3
(class_ro_t) $4 = {
  flags = 128
  instanceStart = 8
  instanceSize = 8
  reserved = 0
  ivarLayout = 0x0000000000000000 <no value available>
  name = 0x0000000100000f99 "TestObject"
  baseMethodList = 0x00000001000010c8
  baseProtocols = 0x0000000000000000
  ivars = 0x0000000000000000
  weakIvarLayout = 0x0000000000000000 <no value available>
  baseProperties = 0x0000000000000000
}
```
可以看到class_ro_t结构体中name和baseMethodList已经有内容了，可以打印一下看看是不是TestObject类中的hello方法：
```
(lldb) p $4.baseMethodList
(method_list_t *) $5 = 0x00000001000010c8
(lldb) p $5->get(0)
(method_t) $6 = {
  name = "hello"
  types = 0x0000000100000fb0 "v16@0:8"
  imp = 0x0000000100000eb0 (debug-objc`-[TestObject hello] at TestObject.m:13)
}
```
没有问题，从name可以看出就是hello方法。method_t结构体也非常简单：
```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
}
```
types那一串乱七八糟的字符串可以参考苹果的文档：[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

扯远了，刚刚验证了在编译期，类的相关信息会存放到class_ro_t中，那么看看运行期是如何把信息添加到class_rw_t中的。这时候就需要看看这个方法了：static Class realizeClass(Class cls)

为什么会突然跳到这个方法，中间过程有些复杂，概括来说就是在研究void _objc_init(void)方法时，通过这么个路径（省略函数签名）map_2_images() -> map_images_nolock() -> _read_images() -> realizeClass()，看到了这个方法，主要是看到了这个方法的注释：
```
* realizeClass
* Performs first-time initialization on class cls, 
* including allocating its read-write data.
* Returns the real class structure for the class. 
* Locking: runtimeLock must be write-locked by the caller
```
这里面提到的read-write data指的就是class_rw_t了，当然实际方法的调用栈并不是上面的路径。在realizeClass方法中添加一个断点：
<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_14/2.png" />
</p>
左侧可以看到方法的调用栈，切换到4那一步：
<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_14/3.png" />
</p>
可以看到是因为在main函数中打印log时调用了class方法，才一步步进入了realizeClass方法。
>猜测：某个类的realizeClass方法是在类被首次调用的时候才会调用。

关于方法的调用，或者说是消息的转发，并不是本文的重点，下一篇讲消息转发机制的时候再具体说。下面看看realizeClass方法是如何实现的：
```
static Class realizeClass(Class cls)
{
    const class_ro_t *ro;
    class_rw_t *rw;
    ...
    ro = (const class_ro_t *)cls->data();
    // Normal class. Allocate writeable class data.
    rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
    rw->ro = ro;
    rw->flags = RW_REALIZED|RW_REALIZING;
    cls->setData(rw);
    ...
    methodizeClass(cls);

    return cls;
}
```
删掉了不少代码，只看关键部分，首先看到这一步：
- ro = (const class_ro_t *)cls->data();
验证了之前我们的猜想，在这个时候，class_rw_t实际上class_ro_t，所以有这一步强转。
- rw->ro = ro;
把ro赋值给rw中的ro字段
- cls->setData(rw);
最后把rw赋值回去，这一步完成之后rw和ro就被正确的设置了，但rw中的方法、属性、协议列表还是空的。
- methodizeClass(cls);
这一步会把ro中的方法、属性、协议拷贝到rw中。另外会把此类所有的category中附加的方法、属性、协议也拷贝进去。oc之所以能在运行时做各种事情，其实都是基于runtime的这些支持。

看看关键的methodizeClass(cls)方法是如何实现的：
```
static void methodizeClass(Class cls)
{
    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }

    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    // Root classes get bonus method implementations if they don't have 
    // them already. These apply before category replacements.
    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
    }

    // Attach categories.
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);
}
```
这个方法结构还是很清楚的，就是通过attachLists()方法把ro中的内容拷贝到了rw中，最后通过attachCategories()方法把分类中的内容也添加进去，这里就不再深挖了。

在methodizeClass()方法结束后，rw中的方法、属性、协议就有内容了，用代码验证一下：
```
(lldb) p (objc_class *)0x100001168
(objc_class *) $5 = 0x0000000100001168
(lldb) p (class_data_bits_t *)0x100001188
(class_data_bits_t *) $6 = 0x0000000100001188
(lldb) p $6->data()
(class_rw_t *) $7 = 0x00000001008022e0
// 之前在这里强转成class_ro_t，现在这个时候已经不需要了，直接获取属性
(lldb) p $7->methods
(method_array_t) $8 = {
  list_array_tt<method_t, method_list_t> = {
     = {
      list = 0x00000001000010c8
      arrayAndFlag = 4294971592
    }
  }
}
// method_array_t结构体中有如下的方法来获取method_list_t的二维数组
(lldb) p $8.beginCategoryMethodLists()[0][0]
(method_list_t) $9 = {
  entsize_list_tt<method_t, method_list_t, 3> = {
    entsizeAndFlags = 26
    count = 1
    first = {
      name = "hello"
      types = 0x0000000100000fb0 "v16@0:8"
      imp = 0x0000000100000eb0 (debug-objc`-[TestObject hello] at TestObject.m:13)
    }
  }
}
```
可以看到这个时候hello方法已经存在了。

## 总结
- 在编译期，类的相关方法，属性，协议会被添加到class_ro_t这个只读的结构体中。
- 在运行期，类第一次被调用的时候，class_rw_t会被初始化，category中的内容也是在这个时候被添加进来的。
- 最开始的猜测不完全正确，class_rw_t不仅仅用来存放运行时添加的信息，编译期确定下来的信息也会被拷贝进去。