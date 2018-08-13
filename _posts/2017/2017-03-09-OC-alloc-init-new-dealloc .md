---
layout: post
title: OC源码 —— alloc,init,new和dealloc
category: OC
tags: [OC]
---

上一篇最后讲release的时候说到，在release的最后，当引用计数减为0的时候就进入了dealloc的过程。这一篇就来讲讲dealloc和相关的一些方法。先从dealloc的对头alloc说起。

***

## alloc

关于alloc，最常见的用法应该算是：
```
[[XXClass alloc] init];
```
方法alloc的调用栈是这样的：
```
+ (id)alloc {
    return _objc_rootAlloc(self);
}

id _objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

static id callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    return class_createInstance(cls, 0);
}

id  class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

static id _class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    size_t size = cls->instanceSize(extraBytes);
    id obj = (id)calloc(1, size);
    obj->initInstanceIsa(cls, hasCxxDtor);
    return obj;
}
```
>在深入alloc方法的时候，我发现alloc方法的调用栈很深，但是做的事情其实不多，中间很多方法看起来一大段，其实真正执行的只有一小部分，所以我在展示源代码的时候只保留了最常用的代码。

下面就来分析一下alloc方法最深处的_class_createInstanceFromZone()方法。

- size_t size = cls->instanceSize(extraBytes);

```
size_t instanceSize(size_t extraBytes) {
    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}

uint32_t alignedInstanceSize() {
  return word_align(unalignedInstanceSize());
}

uint32_t unalignedInstanceSize() {
    return data()->ro->instanceSize;
}

static inline uint32_t word_align(uint32_t x) {
    return (x + WORD_MASK) & ~WORD_MASK;
}
#   define WORD_MASK 7UL
```
第一步获取对象的大小，注释已经讲的很清楚，对象大小最小为16。并且word_align()方法确保大小是8的倍数，其实就是按字节对齐。

- id obj = (id)calloc(1, size);
懂一点c的人都明白，这是分配一块大小为size的内存，并返回指向起始地址的指针。

- obj->initInstanceIsa(cls, hasCxxDtor);
初始化isa，这一部分的源代码在：Runtime源码 —— 对象、类和isa中已经讲过了，这里就不说了。

- return obj;
最后返回初始化好的obj。

总的来说alloc方法还是很简单的。

***

## init

init方法更简单：
```
- (id)init {
    return _objc_rootInit(self);
}

id _objc_rootInit(id obj)
{
    return obj;
}
```
简单到什么都没做。

看到这里，我就奇怪了，这init什么都不做，还要调用了干什么，只要用一个alloc不就够了么，我测试了一下：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_03_09/alloc.png" />
</p>

只调用alloc，可以设置属性，可以调用方法，看起来我们习惯的写法是远古遗留的产物。

***

## new

现在其实这么写的已经不多了：
```
[[XXClass alloc] init];
```
大多数都用一个new代替了：
```
[XXClass new];
```
这个方法也很简单：
```
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```
其实就是把alloc和init结合起来了，不过现在init什么事都不做，new和alloc方法其实是一个意思了。

***

## dealloc

自从进入ARC时代，dealloc方法已经不常见了，除非有CF对象需要释放。

本文开始的时候也提到过，在release方法最后，当引用计数降为0的时候，会调用dealloc方法释放内存。看看dealloc方法究竟做了什么事。
```
- (void)dealloc {
    _objc_rootDealloc(self);
}

void _objc_rootDealloc(id obj)
{
    obj->rootDealloc();
}

inline void objc_object::rootDealloc()
{
    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```
在最后的rootDealloc()方法中，有个if判断，条件分为5个部分：
1. isa.nonpointer
是否是nonpointer，基本都是
2. !isa.weakly_referenced
是否被弱引用
3. !isa.has_assoc 
是否有关联对象
4. !isa.has_cxx_dtor
是否有c++析构器
5. !isa.has_sidetable_rc
引用计数是否曾经溢出过

如果以上判断条件都为真，那么可以快速释放对象，这也是为什么在：Runtime源码 —— 对象、类和isa 这篇文章中介绍isa结构体相关字段时提到过的可以快速释放对象，就是在这里知道的。

那么大多数情况是什么样的呢？大多数情况这个判断都是false，因为第4点，只需要类中声明过属性或者实例变量，那么就需要c++析构函数来释放这些ivars。所以想要快速释放，要求还挺高。

那就看看慢速释放的过程吧：
```
object_dispose((id)this);

id object_dispose(id obj)
{
    objc_destructInstance(obj);    
    free(obj);
    return nil;
}

void *objc_destructInstance(id obj) 
{
    if (obj) {
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
```
释放对象的时机是在objc_destructInstance()方法调用之后，这个方法做的事情是销毁这个对象存在的证据，分成3个部分销毁：
- object_cxxDestruct(obj);
- _object_remove_assocations(obj);
- obj->clearDeallocating();

### part1
```
object_cxxDestruct(obj);

void object_cxxDestruct(id obj)
{
    object_cxxDestructFromClass(obj, obj->ISA());
}

static void object_cxxDestructFromClass(id obj, Class cls)
{
    void (*dtor)(id);

    // Call cls's dtor first, then superclasses's dtors.
    for ( ; cls; cls = cls->superclass) {
        if (!cls->hasCxxDtor()) return; 
        dtor = (void(*)(id))
            lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
        if (dtor != (void(*)(id))_objc_msgForward_impcache) {
            (*dtor)(obj);
        }
    }
}
```
学过c++的都知道析构函数从子类开始，逐层向上调用。这个方法其实就做了这个事情，有一点奇怪的就是，在我们的类中，并没有声明过任何析构函数，那么查找到的析构函数究竟是什么呢？

用代码测试一下，先在类中增加一个属性，这样就能进入这个方法了：
```
// ZNObject.h
@interface ZNObject : NSObject
@property (nonatomic, copy) NSString *name;
@end

// ZNObject.m
@implementation ZNObject
- (void)dealloc {
    NSLog(@"znobject dealloc");
}
```
接着添加这些代码用来测试dealloc的过程：
```
// ViewController.m
@interface ViewController() 
@property (nonatomic, strong) ZNObject *obj;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.obj = [ZNObject new];
}

- (void)viewDidAppear {
    self.obj = nil;
}
```
>挺愚蠢的，其实加个大括号就可以了。

在self.obj = nil;这行先加个断点，进入断点之后，在object_cxxDestructFromClass()方法中添加一个断点，运行进入断点如下图：


<p align="center">
    <img src="http://betterzn.com/assets/images/2017_03_09/dealloc.png" />
</p>

先看左侧的调用栈，当我将self.obj设为nil的时候，就进入了release的过程。这里引用计数为0，所以继续进入了dealloc的过程。先调用了ZNObject类中的dealloc，打印出了log，然后就进入了本节dealloc的流程。

看看dtor的内容，析构函数是一个叫做.cxx_destruct的方法，但是这个方法找不到实现的代码，怎么才能看到这个方法做了什么呢？

换个思路试试看，c++的析构函数是在对象有ivar的时候才会被调用，调用的目的就是为了释放这些ivar，那么我们只需要观察ivar的变化情况就可以了。

调整一下测试的代码，先给属性赋个值，再运行：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_03_09/name.png" />
</p>

添加一个watchpoint，当name变化的时候，会自动进入断点，直接运行程序：


<p align="center">
    <img src="http://betterzn.com/assets/images/2017_03_09/destruct.png" />
</p>

可以看到name的销毁是在objc_storeStrong方法中进行的，这个方法被.cxx_destroy调用。遗憾的是，依然不能窥探到.cxx_destroy究竟做了些什么事。

讲了这么多，其实就说明了ivar释放的过程，下面进入第二步。

### part2

```
_object_remove_assocations(obj);

void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```
关于关联对象，最常见的应用应该就是在category中声明属性。这一块这里就不展开了，后面谈Associated Object的时候再说吧。

这里只要知道这一块做的事情就是移除对象的associated object，更直接一点就是如果有这样的代码：
```
objc_setAssociatedObject(self.obj, @selector(obj), self.obj2, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```
在self.obj对象dealloc的时候，就会进入_object_remove_assocations(obj)方法。

### part3

```
obj->clearDeallocating();

inline void objc_object::clearDeallocating()
{
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}
```
在这一节最开始提到了一共5个判断条件，part1对应的是4和part2对应的是3，其他3个条件就都在这个方法里啦。

第一个if判断对应1，因为基本都是nonpointer，所以直接看else中的内容。else中对应了2和5两个条件，看看clearDeallocating_slow()是怎么做的：
```
void objc_object::clearDeallocating_slow()
{
    SideTable& table = SideTables()[this];
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
}
```
这里再次出现了SideTables，在上一篇讲release的时候有提到过，SideTable存储了溢出的引用计数，也与弱引用相关，这里其实就是对这两部分进行处理。关于SideTable的具体内容就不展开了。这里只需要知道：
- 如果有对此对象的弱引用，那么把所有的弱引用都置为nil。weak对象设为nil原来就是在这里进行的。
- 如果引用计数曾经溢出过，那么SideTable中就存储过相关信息，当然在这个时间点，引用计数的值肯定是为0的，但是即使是0也不放过，还要把曾经存在的痕迹抹除掉。感觉好残忍呀。。。

到这里就完成了dealloc的全部过程。