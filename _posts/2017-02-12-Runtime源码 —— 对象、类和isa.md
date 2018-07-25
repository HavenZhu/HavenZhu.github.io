---
layout: post
title: Runtime源码 —— 对象、类和isa
category: Runtime源码阅读
tags: [Runtime]
---

类、对象、方法和属性算是写OC代码时接触的最多的部分了。本篇就以对象为切入点，分析一下对象和类在runtime层面的表示。

犹记得当初学习C++的时候，买过一本侯捷老师的《STL源码剖析》，书里的内容基本没看，就记得最前面有句话：
>源码面前，了无秘密

## 对象
继承于NSObject的类所生成的对象在runtime中的表示是这样的：
```
struct objc_object {
    isa_t isa;
}
```
很简单，就一个isa_t结构体，从名字也可以看出来这个结构体指明了这个对象是什么，也就是所属的类，isa_t结构体的定义如下：
```
union isa_t {
    Class cls;
    ...
}
(当然不止这么点内容，后面会详细的分析)
```
可以看到这个结构体中有个类型是Class的属性cls，看起来里面应该存有关于这个对象的类的相关信息，看看Class是如何定义的。

## 类
```
typedef struct objc_class *Class;
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}
```
Class就是结构体objc_class，但是objc_class继承于objc_object，那就是说类其实也是一个对象，只不过比通常我们理解的对象多了一些属性，比如superclass等。
>关于其他属性的分析不是本文重点，会在后续文章中结合方法(method)的实现进行分析。

先不看这些属性，这里还有一个很奇怪的问题，既然类也是一个objc_object，那就是说类也有一个isa指针，那类的isa指针指向哪里呢？查看了不少资料，这篇讲的挺好：[classes and metaclasses](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)。

大致的意思是在class之上，还有叫做元类(meta class)的存在，而class的isa指针就是指向对应的meta class。

我们都知道class中存储的是描述对象的相关信息，那么相应的meta class中存放的就是描述class相关信息。说的更直白一点，在我们写代码时，通过对象来调用的方法（实例方法）都是存储在class中的，通过类名来调用的方法（类方法）都是存储在meta class中的。

到这里对象和类的关系已经比较清楚了，但是如果细细思考一下，会发现还有一个问题，就是meta class也是有isa指针的，那么这个isa又指向了哪里呢？在上面给出的那篇文章里面有这么一张图：
![class diagram](http://47.100.168.106/assets/images/2017_02_12/class_diagram.jpeg)

这张图解释的非常清楚，meta class的isa指向了root meta class(绝大部分情况下root class就是NSObject)，root meta class的isa指向自身，isa的链路就是这样了。

## isa_t结构体分析
先看看isa_t的完全版，因为运行环境是osx，所以只截取x86_64部分，arm64的区别只在于部分字段的位数不同，字段是完全相同的：
```
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#   __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; 
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };
}
```
看这个定义只能大概看出个框架，下面从isa的初始化过程来看看isa_t究竟是如何存储类或者元类的相关信息。
```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        isa_t newisa(0);
        newisa.bits = ISA_MAGIC_VALUE;
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
        isa = newisa;
    }
}
```
上来就看不懂，nonpointer是个什么，为什么在这里传的是true？在之前那位大神的另一篇文章中也有解释：[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)。

大概的意思是在64位系统中，为了降低内存使用，提升性能，isa中有一部分字段用来存储其他信息。这也解释了上面isa_t的那部分结构体。
>这有点像taggedPointer，两者之间有什么区别？备注一下后面再研究。

现在知道了nonpointer为什么是true，那么把initIsa方法先简化一下：
```
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    isa_t newisa(0);
    newisa.bits = ISA_MAGIC_VALUE;
    newisa.has_cxx_dtor = hasCxxDtor;
    newisa.shiftcls = (uintptr_t)cls >> 3;
    isa = newisa;
}

#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
```
一共三部分：
1. newisa.bits = ISA_MAGIC_VALUE;
从ISA_MAGIC_VALUE的定义中可以看到这个字段初始化了两个部分，一个是magic字段(6位:111011)，一个是nonpointer字段(1位:1)，magic字段用于校验，nonpointer之前已经详细分析过了。
2. newisa.has_cxx_dtor = hasCxxDtor;
这个字段存储类是否有c++析构器。
3. newisa.shiftcls = (uintptr_t)cls >> 3;
将cls右移3位存到shiftcls中，从isa_t的结构体中也可以看到低3位都是用来存储其他信息的，既然可以右移三位，那就代表类地址的低三位全部都是0，否则就出错了，补0的作用应该是为了字节对齐。

因为nonpointer的缘故，isa并不只是用来存储类地址了，所以需要提供一个额外的方法来返回真正的地址：
```
inline Class 
objc_object::ISA() 
{
    return (Class)(isa.bits & ISA_MASK);
}

#   define ISA_MASK        0x00007ffffffffff8ULL
```
其实就是取isa_t结构体的shiftcls字段。

### 其他字段
还有一些其他的字段，把上面那篇文章中相关部分翻译过来放在下面：
>// 是否曾经或正在被关联引用，如果没有，可以快速释放内存
uintptr_t has_assoc         : 1;

>// 对象是否曾经或正在被弱引用，如果没有，可以快速释放内存
uintptr_t weakly_referenced : 1;

>// 对象是否正在释放内存
uintptr_t deallocating      : 1;

>// 对象的引用计数太大，无法存储
uintptr_t has_sidetable_rc  : 1;

>// 对象的引用计数超过1，比如10，则此值为9
uintptr_t extra_rc          : 8;

## 例子
下面通过代码验证一下之前关于isa的链路，先创建一个用于测试TestObject类，相关代码如下：
```
// TestObject.h
#import <Foundation/Foundation.h>
@interface TestObject : NSObject

@end

// TestObject.m
#import "TestObject.h"
@implementation TestObject

@end
```
为了方便打条件断点，先通过log获取TestObject在内存中的位置：0x100001180，这个时候main函数是这个样子的：
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        TestObject *testObj = [TestObject new];
        NSLog(@"%p", [testObj class]);
        NSLog(@"%p", [TestObject class]);
        NSLog(@"%p", [NSObject class]);
    }
    return 0;
}
```
>只要代码不变，这个类在内存中的地址就不会变

所以在initIsa()方法中添加一个条件断点，并重新运行：
![1](http://47.100.168.106/assets/images/2017_02_12/1.png)

运行程序，当进入断点的时候可以看到方法的调用栈是这样的：
![2](http://47.100.168.106/assets/images/2017_02_12/2.png)

找到2 _class_createInstanceFromZone()，在方法最后打个断点，继续运行程序进入此断点，输出obj的内存地址：
```
Printing description of obj:
<TestObject: 0x101301090>
```
接下来通过这个地址来测试一下isa的链路：
```
//方法中的obj类型是id，id就是objc_object*，所以强转一下
(lldb) p (objc_object *)0x101301090
(objc_object *) $3 = 0x0000000101301090
(lldb) p $3->isa 
(objc_class *) $4 = 0x001d800100001181 // 对象的isa
(lldb) p (objc_object *)0x100001180 // 根据上面isa_t结构体，找到shiftcls的地址，也就是类的真实地址
(objc_object *) $5 = 0x0000000100001180 // TestObject类的真实地址，可以看到与之前打印的[TestObject class]地址是相同的
(lldb) p $5->isa
(objc_class *) $6 = 0x001d800100001159 // 类的isa
(lldb) p (objc_object *)0x100001158
(objc_object *) $7 = 0x0000000100001158 // TestObject元类的真实地址
(lldb) p $7->isa
(objc_class *) $8 = 0x001d8001004a0e49 // 根元类的isa
(lldb) p (objc_object *)0x1004a0e48
(objc_object *) $9 = 0x00000001004a0e48
(lldb) p $9->isa
(objc_class *) $10 = 0x001d8001004a0e49 // 可以看到根元类的isa确实指向自身
(lldb) 
```
测试结果与图class diagram.jpeg给出的完全相同。

在main函数的return行添加断点，运行程序进入断点，有如下输出：
```
NSLog(@"%p", [testObj class]); 
NSLog(@"%p", [TestObject class]);
NSLog(@"%p", [NSObject class]); 

log输出：
0x100001180
0x100001180
0x1004a0e98
```
前两个log结果相同，稍后再分析，这里先看一下NSObject的元类isa：
```
(lldb) p (objc_object *)0x1004a0e98
(objc_object *) $11 = 0x00000001004a0e98
(lldb) p $11->isa
(objc_class *) $12 = 0x001d8001004a0e49
```
可以看到此处$12的值与上方$10是完全相同的，也就验证了NSObject的meta class就是一般类的root meta class。

下面再来看看上面那两个相同的输出，也就是
```
TestObject *testObj = [TestObject new];
NSLog(@"%d", [testObj class] == [TestObject class]);

这个log会输出1
```
看起来有点奇怪，但是只要看一下源代码实现就能理解了。
```
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
```
object_getClass方法最终返回的是isa。所以TestObject调用class方法，返回的是自身；testObj调用class方法，返回的是isa指向的类，也是TestObject。所以上面结果相同就不奇怪了。

看到这里就顺便看一下可能会接触到isa的常用的几个方法：isMemberOfClass，isKindOfClass。废话不说，直接上源码：
```
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```
从源码来看这两个方法就一目了然了。有兴趣的可以写几个例子测试一下。