---
layout: post
title: Runtime源码 —— 方法调用的过程
category: Runtime
tags: [Runtime]
---

在写这篇文章之前，我关于方法调用的知识是比较零散的，甚至一度以为消息转发就是方法调用的过程。

现有的文章大多根据苹果的官方文档[Runtime Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1)进行分析，一般包含这些内容：
- 方法的调用会被转换成objc_msgSend()
- 如果找不到方法的实现，会开始执行动态方法解析
- 如果动态方法解析失败了，会启动消息转发

所以消息转发应该只是方法调用中的一个步骤。这中间似乎缺了点什么，那就是：
- 在启动消息转发之前，objc_msgSend()做了什么？

这也就是本文将要解答的：方法究竟是如何被调用的？

## 方法的调用栈
在上一篇讲方法加载的过程时，用过这么一张图来讲realizeClass()的调用栈：
<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/realizeClass.png" />
</p>

当时调用的是类的class方法，在调用栈里有这么一个关键的方法：
```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
```
方法名字就是查找实现或者转发，看起来这就是我们要找的方法了。

沿用之前的TestObject类，再修改一下main函数的内容，现在看起来是这个样子的：
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

// main.m
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        TestObject *testObj = [TestObject new];
        [testObj hello];
    }
    return 0;
}
```
在[testObj hello]这一行添加一个断点，运行程序进入断点，这时候在lookUpImpOrForward()方法中添加断点，继续运行进入此方法：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/hello.png" />
</p>
左侧的调用栈里面供包含了3层，按照调用的顺序依次是：
- _objc_msgSend_uncached
- _class_lookupMethodAndLoadCache3(id, SEL, Class)
- lookUpImpOrForward(Class, SEL, id, bool, bool, bool)

一步步来看：
- _objc_msgSend_uncached
不对啊，官方文档中说的是调用objc_msgSend，这个uncached是怎么回事。看看objc_msgSend:
```
        ...
        ENTRY _objc_msgSend
        UNWIND _objc_msgSend, NoFrame
        MESSENGER_START

        NilTest	NORMAL

        GetIsaFast NORMAL		// r10 = self->isa
        CacheLookup NORMAL, CALL	// calls IMP on success

        NilTestReturnZero NORMAL

        GetIsaSupport NORMAL
// cache miss: go search the method lists
LCacheMiss:
        // isa still in r10
        MESSENGER_END_SLOW
        jmp	__objc_msgSend_uncached

        END_ENTRY _objc_msgSend
        ...
```
源码是汇编，说实话我是不太懂的，但没关系，关注一下这一行：
jmp	__objc_msgSend_uncached。
从注释可以看到当cache miss的时候，会跳转到uncached方法中，到底是不是这样呢？重新运行程序，加个断点测试一下：

（注意，这里也需要先运行进入main函数中[testObj hello]这一行之后再激活断点）

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/msgSend.png" />
</p>
没有问题，调用栈显示先进入了objc_msgSend，单步调试的图我就不放了，感兴趣的同学可以自己试一下，下面是过程：
1. 先进入：CacheLookup NORMAL, CALL
2. cache miss，跳到这里：jmp	__objc_msgSend_uncached
3. 进入：__objc_msgSend_uncached

这个时候调用栈的objc_msgSend已经看不到了，取而代之的就是__objc_msgSend_uncached：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/msgSend_uncached.png" />
</p>

所以之前调用栈中的结果就可以理解了，这里也告诉了我们一个很重要的信息：**在objc_msgSend最开始的地方就已经通过cache进行过一次查找。**

-  _class_lookupMethodAndLoadCache3(id, SEL, Class)

现在断点所在的行是这么一个方法：MethodTableLookup。看起来像是在方法列表里进行查找。沿着断点继续走，就会走到现在这个方面里面，这个方法的实现非常简单：
```
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```
就是完善了一下lookUpImpOrForward()的参数。话不多说，看看最关键的一步。

- lookUpImpOrForward(Class, SEL, id, bool, bool, bool)

这个方法的实现有点长，我就不一起展示了，一步一步来分析：
### part1
```
    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    if (!cls->isRealized()) {
        rwlock_writer_t lock(runtimeLock);
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
    }
```
还记得前面说到的关键信息吗，之所以传入cache=NO就是因为在objc_msgSend()初期就已经查找过cache了，不需要在这里再查找一次。这部分代码主要做的是初始化的相关工作，这里不做扩展。接着往下：
### part2
```
retry:
    runtimeLock.read();

    // Try this class's cache.
    imp = cache_getImp(cls, sel);
    if (imp) goto done;
```
>加锁这一部分只有一行简单的代码，其主要目的保证方法查找以及缓存填充（cache-fill）的原子性，保证在运行以下代码时不会有新方法添加导致缓存被冲洗（flush）。

这里又一次使用cache进行查找。这里我是有点疑问的，在这个时候cache有可能会命中吗？或者说在什么情况下才能在这里命中cache？

在上一篇方法加载的过程中提到，在realizeClass()方法深处会拷贝编译期确定的方法同时添加category中的方法，难道这个过程改变了cache的内容，所以需要在这里查一下cache？先不深究，等研究category的时候看看能不能有所进展。

cache_getImp()方法同样是用汇编实现的：
```

	STATIC_ENTRY _cache_getImp

// do lookup
	movq	%a1, %r10		// move class to r10 for CacheLookup
	CacheLookup NORMAL, GETIMP	// returns IMP on success

LCacheMiss:
// cache miss, return nil
	xorl	%eax, %eax
	ret

	END_ENTRY _cache_getImp
```
CacheLookup应该就是用来查找cache的，这里是首次调用hello()方法，所以肯定不会命中，继续向下。
### part3
```
    // Try this class's method lists.
    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }
```
在当前类的方法列表中查找，因为hello()就是当前类的方法，所以在这一步会命中，命中时候的调用栈是这样的：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/current_func_shot.png" />
</p>

中间的方法都比较简单，我就不把源代码一一贴上来了，稍微说一下每个方法做了些什么：
- getMethodNoSuper_nolock(Class cls, SEL sel)
遍历class的methods列表，依次调用下一个方法
- search_method_list(const method_list_t *mlist, SEL sel)
如果是无序列表，直接匹配名字，成功则返回
如果是有序列表，调用下一个方法
- findMethodInSortedMethodList(SEL key, const method_list_t *list)
匹配方法名，成功就直接返回

这些做完之后，会调用log_and_fill_cache()把方法加入缓存，这个方法的调用栈是这样的：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_16/log_and_fill_cache.png" />
</p>

在cache_fill_nolock()方法中把当前调用的方法加入到cache中：
```
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
    cacheUpdateLock.assertLocked();

    if (!cls->isInitialized()) return;
    if (cache_getImp(cls, sel)) return;

    cache_t *cache = getCache(cls);
    cache_key_t key = getKey(sel);

    // Use the cache as-is if it is less than 3/4 full
    mask_t newOccupied = cache->occupied() + 1;
    mask_t capacity = cache->capacity();
    if (cache->isConstantEmptyCache()) {
        // Cache is read-only. Replace it.
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
    }
    else if (newOccupied <= capacity / 4 * 3) {
        // Cache is less than 3/4 full. Use it as-is.
    }
    else {
        // Cache is too full. Expand it.
        cache->expand();
    }

    bucket_t *bucket = cache->find(key, receiver);
    if (bucket->key() == 0) cache->incrementOccupied();
    bucket->set(key, imp);
}
```
注释还是很清楚的，在cache已经3/4满的时候，就会调用expand()方法扩充，这样可以保证cache一直都是有空位的：
```
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}
```
中间的if判断是对溢出情况的处理。正常情况下，expand方法会将容量翻倍，通过调用reallocate方法给cache重新分配内存，但出于性能考虑不会将老cache中的内容拷贝到新cache中。

这里插一点题外话，如果对swift没兴趣就跳过吧。这里的操作让我想起了swift中map的实现：
```
public func map<T>(
    _ transform: (Iterator.Element) throws -> T
  ) rethrows -> [T] {
    let initialCapacity = underestimatedCount
    var result = ContiguousArray<T>()
    result.reserveCapacity(initialCapacity)

    var iterator = self.makeIterator()

    // Add elements up to the initial capacity without checking for regrowth.
    for _ in 0..<initialCapacity {
      result.append(try transform(iterator.next()!))
    }
    // Add remaining elements, if any.
    while let element = iterator.next() {
      result.append(try transform(element))
    }
    return Array(result)
  }
```
里面有这么一行：
```
result.reserveCapacity(initialCapacity)
```
就是先直接申请了一段空间用来存放结果，满了之后才需要检查是否需要扩充，所以result.append()操作才会分成两部分来做，应该也是出于性能的考虑。

### part4
因为hello()方法已经在上一步找到了，所以走不到下面的代码了，但还是可以看一看：
```
    // Try superclass caches and method lists.
    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, imp, sel, inst, curClass);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
            imp = meth->imp;
            goto done;
        }
    }
```
这一块还是很好理解的，就是在父类的缓存和方法列表中查找，逻辑跟前面两步基本一样，就不再细说了。只需要注意一点，在父类中找到的方法，也会被添加到当前类的cache中。

### part5
```
    // No implementation found. Try method resolver once.
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }
```
如果当前类和父类都找不到方法的实现，就进入了动态方法解析。这里面调用了_class_resolveMethod()方法，看看是怎么实现的：
```
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    if (! cls->isMetaClass()) {
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
```
还是很清楚的，如果类不是元类，调用_class_resolveInstanceMethod()，是元类则调用_class_resolveClassMethod()。这两个方法很类似，就以第一个为例，注意看我添加的注释：
```
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    // 查找类是否实现了+ (BOOL)resolveInstanceMethod:(SEL)sel方法
    // 如果没有实现就直接返回
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (__typeof__(msg))objc_msgSend;
    // 调用类里面实现的+ (BOOL)resolveInstanceMethod:(SEL)sel
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    ...(略去了一些代码，主要是验证是否添加成功)
}
```
关于+ (BOOL)resolveInstanceMethod:(SEL)sel方法，这里就不细说了，有非常多的文章讲解了这个方法该怎么写，如果曾经看过，就会知道在这个方面里面通常都会调用:
```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
```
通过这个方法来给某个方法添加新的实现。在这个方法内部，有这么一行：
```
cls->data()->methods.attachLists(&newlist, 1);
```
将新的方法实现添加到了方法列表里面。这就完成了整个动态方法解析的过程。

这个时候回到part5最开始的地方，在调用完_class_resolveMethod()方法之后，有一步goto retry，就是回到part2重新开始，只不过这个时候在类的方法列表里面就可以找到这个方法了。

### part6
```
    // No implementation found, and method resolver didn't help. 
    // Use forwarding.
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);
```
如果上一步依然没有解决问题，还有最后一个办法：消息转发。这个过程实在是太复杂，简单一点来说，如果你的类实现了这个方法：
```
- (void)forwardInvocation:(NSInvocation *)anInvocation
```
这个时候就会进到这个方法里面，在这里可以转发给其他对象进行处理。如果消息转发也失败了，那么这次方法的调用就失败了。

如果想要对消息转发的全部过程有更深刻的理解，可以参考这篇文章，讲的很详细：
>[forwarding 中路漫漫的消息转发](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/#forwarding-中路漫漫的消息转发)

## 缓存命中

上面讲了那么多，前提是objc_msgSend汇编代码中的的缓存没有命中，如果在最开始缓存就命中了，会怎么样呢？

想要测试命中缓存很简单，把方法连续调用两次就可以了，第二次调用的时候上面那些方法都不会被调用到，直接就把hello()方法的log打印出来了。

## 总结

最后汇总一下正常方法调用的过程，总的来看还是很合情合理的：
- 查找当前类的缓存和方法列表
- 查找父类的缓存和方法列表
- 动态方法解析
- 消息转发

## 参考资料

[[从源代码看 ObjC 中消息的发送](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/%E4%BB%8E%E6%BA%90%E4%BB%A3%E7%A0%81%E7%9C%8B%20ObjC%20%E4%B8%AD%E6%B6%88%E6%81%AF%E7%9A%84%E5%8F%91%E9%80%81.md) ](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/%E4%BB%8E%E6%BA%90%E4%BB%A3%E7%A0%81%E7%9C%8B%20ObjC%20%E4%B8%AD%E6%B6%88%E6%81%AF%E7%9A%84%E5%8F%91%E9%80%81.md)