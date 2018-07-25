---
layout: post
title: OC源码 —— retain和release
category: OC源码阅读
tags: [OC]
---

retain/release两个关键字现在已经很少见了，但了解一下底层的实现还是能帮助我们更深刻的理解oc的内存管理。
***

## retain

通常情况下，当我们对一个对象调用retain方法时，调用的顺序是这样的：
```
[NSObject retain];

- (id)retain {
    return ((id)self)->rootRetain();
}

id objc_object::rootRetain()
{
    return rootRetain(false, false);
}

id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    if (isTaggedPointer()) return (id)this;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
            else return sidetable_retain();
        }
        if (slowpath(tryRetain && newisa.deallocating)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry); 

        if (slowpath(carry)) {
            if (!handleOverflow) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    if (slowpath(transcribeToSideTable)) {
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
```
最后这个方法：
```
id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
```
看起来有点复杂，但没关系，我会分成几种情况来分析。

在分析之前需要先回顾一下在：Runtime源码 —— 对象、类和isa 中介绍过isa的结构，其中有这么两个字段：
>// 对象的引用计数太大，无法存储
uintptr_t has_sidetable_rc : 1;

>// 对象的引用计数超过1，比如10，则此值为9
uintptr_t extra_rc : 8;

这两个字段在上面那个方法中也出现了，从注释来看，一个字段用来存储引用计数的数值，另一个标记引用计数是否溢出。下面就分溢出与否两种情况来讨论。

### 不溢出

最简单的情况当然是不溢出，这种情况下，rootRetain()方法可以简写如下：
```
id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry); 
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    return (id)this;
}
```
这里面的几个方法都需要解释一下，按照顺序来：
>这里的实现版本是x86_64

#### LoadExclusive
```
LoadExclusive(&isa.bits)

static uintptr_t LoadExclusive(uintptr_t *src)
{
    return *src;
}
```
第一个很简单，就是获取isa的内容。

#### addc
```
addc(newisa.bits, RC_ONE, 0, &carry)
#       define RC_ONE   (1ULL<<56)

static uintptr_t addc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
{
    return __builtin_addcl(lhs, rhs, carryin, carryout);
}
```
我搜不到__builtin_addcl方法的定义或者说明文档，我只能根据测试结果来做一些猜测。测试的过程是这样的：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_03_07/addcNotFlow.png" />
</p>

首先我在rootRetain()方法中添加了两个断点，一个在while内部，一个在外部，运行程序进入第一个断点，获取了一下这个时候isa的内容：
```
(lldb) p &isa
(isa_t *) $0 = 0x0000600000064f40
(lldb) p *$0
(isa_t) $1 = {
  cls = NSThread
  bits = 8444248074519609
   = {
    nonpointer = 1
    has_assoc = 0
    has_cxx_dtor = 0
    shiftcls = 17592032694407
    magic = 59
    weakly_referenced = 0
    deallocating = 0
    has_sidetable_rc = 0
    extra_rc = 0
  }
}
```
方法结束之后，看到carry的值是0。接着运行进入第二个断点，再获取一下isa的内容：
```
(lldb) p *$0
(isa_t) $2 = {
  cls = NSThread
  bits = 80501842112447545
   = {
    nonpointer = 1
    has_assoc = 0
    has_cxx_dtor = 0
    shiftcls = 17592032694407
    magic = 59
    weakly_referenced = 0
    deallocating = 0
    has_sidetable_rc = 0
    extra_rc = 1
  }
}
```
看到变化了没有，extra_rc的值增加了1。这个时候可以做一些猜测，首先根据isa_t结构体，extra_rc是在第57~64位，RC_ONE就是第57位为1，addc()方法结束之后，extra_rc从0->1，相当于两者相加了，没有溢出，返回的carry值为0。

再测试一下溢出的情况，需要调整一下断点的位置如图：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_03_07/addcFlow.png" />
</p>

进入第一个断点获取一下extra_rc的值，因为获取的时isa的内容，所以还是oldisa的值：
```
(lldb) p &isa
(isa_t *) $3 = 0x0000000100882a00
(lldb) p *$3
(isa_t) $4 = {
  cls = MTLIGAccelDevice
  bits = 18382989995916412357
   = {
    nonpointer = 1
    has_assoc = 0
    has_cxx_dtor = 1
    shiftcls = 553978040
    magic = 59
    weakly_referenced = 0
    deallocating = 0
    has_sidetable_rc = 0
    extra_rc = 255
  }
}
```
值为255，加1就会溢出，所以carry的值为1，继续运行进入第二个断点，再获取一下isa：
```
(lldb) p *$3
(isa_t) $5 = {
  cls = MTLIGAccelDevice
  bits = 9267704350118528453
   = {
    nonpointer = 1
    has_assoc = 0
    has_cxx_dtor = 1
    shiftcls = 553978040
    magic = 59
    weakly_referenced = 0
    deallocating = 0
    has_sidetable_rc = 1
    extra_rc = 128
  }
}
```
extra_rc从255->128，has_sidetable_rc从0->1，这里边的过程一会儿讲溢出的时候再说。

从这个结果又可以做一点猜测，如果addc的前两个参数加起来溢出了，carry的值就会变化，反正不等于0了，具体怎么变化，是不是存储溢出的值就不得而知了。这个方法就先这样了，大概的意思是懂了。

#### StoreExclusive
```
StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)

static bool StoreExclusive(uintptr_t *dst, uintptr_t oldvalue, uintptr_t value)
{
    return __sync_bool_compare_and_swap((void **)dst, (void *)oldvalue, (void *)value);
}
```
方法内部又调用了一个方法，在gcc的文档中这个方法是这样解释的：
```
bool __sync_bool_compare_and_swap (type *ptr, type oldval type newval, ...)
...
These builtins perform an atomic compare and swap. That is, if the current value of *ptr is oldval, then write newval into *ptr.
The “bool” version returns true if the comparison is successful and newval was written. 
```
应用到方法中去，就是如果&isa.bits和oldisa.bits相等，那么就把newisa.bits的值赋给&isa.bits，并且返回true。

在这里&isa.bits和oldisa.bits当然是相等的，所以while判断一次就结束了。

这3个方法讲清楚之后再回去看看简化版的rootRetain()方法就很简单了，其实就是给extra_rc+1，然后更新一下isa的内容。

### 有溢出

有溢出的时候，情况稍微复杂一点，rootRetain()方法在这个时候会变成这样：
```
id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    isa_t oldisa;
    isa_t newisa;

    oldisa = LoadExclusive(&isa.bits);
    newisa = oldisa;
    uintptr_t carry;
    newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry); 
    
    if (!handleOverflow) {
        ClearExclusive(&isa.bits);
        return rootRetain_overflow(tryRetain);
    }
}

id objc_object::rootRetain_overflow(bool tryRetain)
{
    return rootRetain(tryRetain, true);
}
```
方法的前半部分一模一样，但是因为溢出了，所以后面的路线变化了，直接调用了rootRetain_overflow()方法，这个方法内部又调用了rootRetain()方法。

不同的是，首次调用两个参数都是false，而这次调用，第二个参数为true。所以这个时候的rootRetain()方法又可以重新简化一下：
```
id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);

        newisa.extra_rc = RC_HALF;
        newisa.has_sidetable_rc = true;
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    sidetable_addExtraRC_nolock(RC_HALF);

    return (id)this;
}
#       define RC_HALF  (1ULL<<7)
```
奇怪的是，看看完整版的rootRetain()方法，第二个参数并没有其他用处，为什么需要绕个弯再调用一下自己，而不是直接继续执行后面的代码呢？

源代码中对此的注释只有：
```
// handleOverflow=false is the frameless fast path.
// handleOverflow=true is the framed slow path including overflow to side table
// The code is structured this way to prevent duplication.
```
但是就从源代码来看，直接将这个参数去掉，rootRetain()方法只留下第一个参数都是可以的。甚至还减少了函数调用的次数。

先不管这个疑问，这一次，更新了newisa的extra_rc和has_sidetable_rc字段，将extra_rc设置为了RC_HALF也就是128，has_sidetable_rc设为true，这就是为什么上面测试溢出时输出结果为：
>extra_rc = 128
has_sidetable_rc = 1

就是在此处设置的。

这里就有一个疑问了，讲道理这个时候extra_rc应该为256，这里值为128，还有128哪里去了？

这就是最后那个方法做的事情了：
```
sidetable_addExtraRC_nolock(RC_HALF);

bool objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage = SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}

#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  
#define SIDE_TABLE_RC_ONE            (1UL<<2)
#define SIDE_TABLE_RC_SHIFT 2
```
SideTable看起来就是用来存储引用计数的。看结构体SideTable的源码，应该和weak reference也有点关系，以后研究weak的时候再深入吧，目前我也不是很清楚。

现在只需要注意到在获取到oldRefcnt之后，紧跟着两个assert，从宏的定义来看，分别是第一和第二位，assert是为了确保oldRefcnt低两位为0，因为低两位是有作用的标志位。也就是说引用计数实际上是从第三位开始存放的。

再向后看，这就解释了addc()第二个参数左移了两位进行累加了。

这里再一次进行了溢出判断，如果溢出了，存放的引用计数被置为了SIDE_TABLE_RC_PINNED，因为(oldRefcnt & SIDE_TABLE_FLAG_MASK)结果一定是0，但是我不理解这里为什么要这么做。

同样不能理解的还有addc之前的if判断。是不是引用计数太大，就不处理了呢？毕竟有62位用来存，这要是再溢出了就无法想象了。

验证一下正常的结果：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_03_07/addExtraRCNotFlow.png" />
</p>

传入的参数是RC_HALF，也就是128，在addc之后，newRefcnt变为512 = 128 << 2也没问题，carry为0没有溢出，最后更新了一下存放的引用计数值。

这里稍微汇总一下，正常情况下，retain方法就是给extra_rc加1，当extra_rc溢出时，将一半的引用计数存放到SideTable中。

一个合理的疑问是，当extra_rc再次溢出的时候呢？很容易测试，只需要再进一次断点，如下图：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_03_07/addExtraRCFlow.png" />
</p>

这个时候oldRefcnt已经是512 = 128 << 2，也就是上一步存进去的结果，addc之后又加了128进去。说白了每次extra_rc溢出了，SideTable中就增加128。

到这里，关于retain方法的源码已经看的差不多了，但是还有疑问：
> 为什么每次只移动一半的引用计数（也就是128）到SideTable中？

我也不知道。

***

## release

在深入理解了retain之后，再看release的源码就很简单了。代码太长我就不贴了，总结一下release的流程：

1. 最普通的情况，直接将extra_rc减1
2. 如果extra_rc为0，判断has_sidetable_rc
3. has_sidetable_rc = false，说明对象已经没有引用计数了，直接dealloc释放内存
4. has_sidetable_rc = true，说明extra_rc有过溢出
5. 从SideTable中借位成功，每次取RC_HALF，也就是128，减1之后赋给extra_rc，回到步骤1
6. 从SideTable中借位失败，直接dealloc

***

## retainCount

最后再说一下和retain有关的一个方法，就是获取引用计数值retainCount，这个方法其实不看源码也能想象出来是怎么计算的：

retainCount = 1+ extra_rc + SideTable中存储的rc

源码贴一下作为结尾：
```
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}

inline uintptr_t 
objc_object::rootRetainCount()
{
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    return sidetable_retainCount();
}
```