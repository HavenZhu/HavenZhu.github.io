---
layout: post
title: Runtime源码 —— property和ivar
category: Runtime
tags: [Runtime]
---

我原本以为这两个东西没啥好写的，结果是property确实没啥好写的，但是ivar就不少了。

>本文不探讨何时该选择property，何时该选择ivar

我会把我研究这两东西的过程原原本本的展示出来。

## 试探期
在方法加载的过程中，我提到过两个结构体：
- class_ro_t
记录编译期就已经确定的信息
- class_rw_t
运行期拷贝class_ro_t中的部分信息存入此结构体中，并存放运行期添加的信息

细心的同学应该发现在ro和rw结构体中，method、protocol和property都是存在的，拷贝也就是拷贝的这部分信息，但是ro中还存在一个字段叫做：
```
const ivar_list_t * ivars;
```
这玩意儿就是本文研究的重点了。

### 例子
在写代码之前，我心里是这样想的：
ivar都应该存在ro的ivars字段中，property存在ro的baseProperties字段中，在运行期，将property拷贝到rw中。获取property的时候直接从rw中获取，获取ivar则从ro中获取。

写代码测试一下：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    NSInteger ivarInt;
    BOOL ivarBool;
}
@property (nonatomic, assign) NSInteger propertyInt;
@end

// ZNObjectFather.m
#import "ZNObjectFather.h"
@implementation ZNObjectFather
@end
```

还是通过lldb验证一下：
```
// 获取ZNObjectFather class的内存地址
2017-02-21 10:06:12.772188 TestOSX[6560:288465] 0x100002e10
(lldb) p (class_data_bits_t *)0x100002e30
(class_data_bits_t *) $0 = 0x0000000100002e30
(lldb) p $0->data()
(class_rw_t *) $1 = 0x000060800007e140
(lldb) p (*$1).ro
(const class_ro_t *) $2 = 0x0000000100002318
(lldb) p *$2
(const class_ro_t) $3 = {
  flags = 128
  instanceStart = 8
  instanceSize = 32
  reserved = 0
  ivarLayout = 0x0000000000000000 <no value available>
  name = 0x000000010000139a "ZNObjectFather"
  baseMethodList = 0x0000000100002260
  baseProtocols = 0x0000000000000000
  ivars = 0x0000000100002298
  weakIvarLayout = 0x0000000000000000 <no value available>
  baseProperties = 0x0000000100002300
}
// 获取ivars
(lldb) p $3.ivars
(const ivar_list_t *const) $4 = 0x0000000100002298
(lldb) p *$4
// $5的内容显示count = 3，但是实际只声明了两个ivar
(const ivar_list_t) $5 = {
  entsize_list_tt<ivar_t, ivar_list_t, 0> = {
    entsizeAndFlags = 32
    count = 3
    first = {
      offset = 0x0000000100002d88
      name = 0x0000000100001448 "ivarInt"
      type = 0x0000000100001aee "q"
      alignment_raw = 3
      size = 8
    }
  }
}
(lldb) p $5.get(1)
// ivar_t的结构体后面会分析
(ivar_t) $6 = {
  offset = 0x0000000100002d90
  name = 0x0000000100001450 "ivarBool"
  type = 0x0000000100001af0 "c"
  alignment_raw = 0
  size = 1
}
(lldb) p $5.get(2)
// 发现声明的属性自动生成了一个_propertyName的ivar
(ivar_t) $7 = {
  offset = 0x0000000100002d80
  name = 0x0000000100001459 "_propertyInt"
  type = 0x0000000100001aee "q"
  alignment_raw = 3
  size = 8
}
// 获取property
(lldb) p $3.baseProperties
(property_list_t *const) $8 = 0x0000000100002300
(lldb) p *$8
// 结果符合预期
(property_list_t) $9 = {
  entsize_list_tt<property_t, property_list_t, 0> = {
    entsizeAndFlags = 16
    count = 1
    first = (name = "propertyInt", attributes = "Tq,N,V_propertyInt")
  }
}
```
property的测试结果很正常，ivar不完全相同，如果直接声明的2个之外，属性也自动生成了一个ivar。

另外$3的baseMethodList也不为空，存的就是property自动生成的get/set方法，感兴趣自己打印一下。

看到这里也就不难理解为什么：
>property = ivar + get + set

但也并不总是这样，如果重写了属性的get/set方法，就不会生成_propertyName这样的ivar了，本文不做深入。


再看看$3里面的这么两个属性：
```
instanceStart = 8
instanceSize = 32
```
- instanceStart之所以等于8，是因为每个对象的isa占用了前8个字节。
- instanceSize = isa + 3个ivar，$6的size只有1，但是为了对齐，也占用了8，对齐是怎么计算的后面再讲。

到这里对ivar和property已经有一个大概的理解了，下面继续深入。

## 深入期

根据上半部分的分析，我们已经知道了
- ivars在编译期就已经确定了
- 属性会生成 _propertyName格式的ivar，也在编译期确定
- 对象的大小是由 isa + ivars决定的

但是这就引出了如下几个问题：
- 带有继承体系的对象是怎么表示的？
- 继承体系中对象的 instanceStart和 instanceSize是怎么计算的？
- ivar_t中的 alignment_raw和 offset是什么意思？
- class_ro_t中的 ivarLayout和 weakIvarLayout是什么意思？

现在我们都知道class_ro_t中的ivar，property，protocol和method都是在编译期就确定的，在运行期时，通过realizeClass()方法将部分信息拷贝到class_rw_t中。

在realizeClass()方法中有这么一段代码：
```
// Reconcile instance variable offsets / layout.
// This may reallocate class_ro_t, updating our ro variable.
if (supercls  &&  !isMeta) reconcileInstanceVariables(cls, supercls, ro);
```
注释里面讲了，这一步会调整ivar的offset值，并且更新ro的信息，看起来这一步就是关键，看看方法是怎么实现的：
```
static void reconcileInstanceVariables(Class cls, Class supercls, const class_ro_t*& ro) 
{
    class_rw_t *rw = cls->data();
    ...
    const class_ro_t *super_ro = supercls->data()->ro;
    ...// 省略了用于debug的相关代码
    if (ro->instanceStart >= super_ro->instanceSize) {
        // Superclass has not overgrown its space. We're done here.
        return;
    }
    if (ro->instanceStart < super_ro->instanceSize) {
        ...
        class_ro_t *ro_w = make_ro_writeable(rw);
        ro = rw->ro;
        moveIvars(ro_w, super_ro->instanceSize);
        gdb_objc_class_changed(cls, OBJC_CLASS_IVARS_CHANGED, ro->name);
    } 
}
```
只保留了最关键的代码，关注一下其中的if判断，比较的是当前类的instanceStart和父类的instanceSize，当start < size的时候调整了一下当前类ro的相关信息。

这给了我一个信息，也就是在这一步之前，ro中的instanceStart和instanceSize其实并不是最终值。

具体调整的过程在moveIvars(ro_w, super_ro->instanceSize)这个方法中完成：
```
static void moveIvars(class_ro_t *ro, uint32_t superSize)
{
    ...
    uint32_t diff;
    ...
    diff = superSize - ro->instanceStart;

    if (ro->ivars) {
        uint32_t maxAlignment = 1;
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t alignment = ivar.alignment();
            if (alignment > maxAlignment) maxAlignment = alignment;
        }

        uint32_t alignMask = maxAlignment - 1;
        diff = (diff + alignMask) & ~alignMask;

        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t oldOffset = (uint32_t)*ivar.offset;
            uint32_t newOffset = oldOffset + diff;
            *ivar.offset = newOffset;

            ...
        }
    }

    *(uint32_t *)&ro->instanceStart += diff;
    *(uint32_t *)&ro->instanceSize += diff;
}
```
这个方法做了这些事情：
- 更新当前类ivar中的offset字段
- 更新当前类ro的instanceStart和instanceSize

先按照源代码分析，最后写代码验证。

### part1
```
diff = superSize - ro->instanceStart;
```
获取了当前类的instanceStart和父类的instanceSize的偏移量，但这并不是最终的结果，因为存在对齐的问题。这就是后面这个if判断内部做的事情。

### part2
先看第一个for循环：
```
for (const auto& ivar : *ro->ivars) {
    if (!ivar.offset) continue;  // anonymous bitfield

    uint32_t alignment = ivar.alignment();
    if (alignment > maxAlignment) maxAlignment = alignment;
}
```
遍历了ivars，获取了最大得alignment。这个ivar.alignment()是ivar_t结构体中的方法：
```
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
}

#   define WORD_SHIFT 3UL
```
>备注：这里有这么一个字段：alignment_raw，这个字段据我的理解，应该是在编译期确定的，但是是按照什么规则确定的就不清楚了。根据测试的结果来看，一般都是0或者3。

通过ivar.alignment()得到的结果是1 << 3，也就是8。

### part3
```
uint32_t alignMask = maxAlignment - 1;
diff = (diff + alignMask) & ~alignMask;
```
这一步确定了diff的值，那个&运算的结果就是把diff按8对齐，比如本来diff = 9，这一步之后diff = 16。

### part4
```
for (const auto& ivar : *ro->ivars) {
    if (!ivar.offset) continue;  // anonymous bitfield

    uint32_t oldOffset = (uint32_t)*ivar.offset;
    uint32_t newOffset = oldOffset + diff;
    *ivar.offset = newOffset;
}
```
这一步调整ivar的offset字段，调整的过程就是用原来的offset加上上一步得到的diff。说白了就是当前类的ivar是在父类的ivar之后的。

### part5
```
*(uint32_t *)&ro->instanceStart += diff;
*(uint32_t *)&ro->instanceSize += diff;
```
最后更新了当前类的instanceStart和instanceSize，过程也是加上diff。其实就是把父类的instanceSize给空出来了。

到这里的时候，已经回答了这部分最开始提出的4个问题中的前3个。先来验证一下。

### 例子
为了验证前3个问题，需要给增加一个类：
```
// ZNObjectSon.h
#import "ZNObjectFather.h"
@interface ZNObjectSon : ZNObjectFather {
    NSInteger ivarIntSon;
    BOOL ivarBoolSon;
}
@property (nonatomic, assign) NSInteger propertyIntSon;

@end
// ZNObjectSon.m
#import "ZNObjectSon.h"
@implementation ZNObjectSon
@end
```
此类继承于ZNObjectFather，按照老套路，还是先获取一下类的地址：
```
2017-02-21 11:43:49.962750 TestOSX[6743:331148] father address: 0x100002e30
2017-02-21 11:43:49.962803 TestOSX[6743:331148] son address: 0x100002de0
```

接着在reconcileInstanceVariables()方法中添加一个条件断点，进入断点后，通过lldb获取一下相关值，请看图：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_21/ZNObjectFather.png" />
</p>

条件断点设置的是ZNObjectFather的地址，所以：
- ro的信息就是ZNObjectFather的ro

ZNObjectFather继承于NSObject，所以：
- super_ro是NSObject的ro

根据控制台打印的信息，这一步的if判断结果为true，所以直接return了，调整一下条件断点的内容，把地址设置为ZNObjectSon的地址再试一下：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_21/ZNObjectSon.png" />
</p>

可以看到father的start和size没有发生变化，因为上一步做过说明直接return了。

再来看看son的start值，说实话看到这个24我是无法理解的。在这之前我预期start = 8，这多出来的16是怎么回事？

我做了一个猜测：**instanceStart的值在编译期已经计算了父类直接声明的ivar，由property生成的没有计算。**

我做了一些验证，先把father类中的属性注释掉了：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    NSInteger ivarInt;
    BOOL ivarBool;
}
//@property (nonatomic, assign) NSInteger propertyInt;
@end
```
这时候打印出来的start和size如下：
```
// ZNObjectFather
instanceStart = 8
instanceSize = 24

// ZNObjectSon
instanceStart = 24
instanceSize = 48
```
没有问题，father的size少了8，son没有变化，这个时候son的start >= father的size，所以直接return。

如果把father中的一个ivar注释掉：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    NSInteger ivarInt;
//    BOOL ivarBool;
}
@property (nonatomic, assign) NSInteger propertyInt;
@end
```
这时候打印出来的start和size如下：
```
// ZNObjectFather
instanceStart = 8
instanceSize = 24

// ZNObjectSon
instanceStart = 16
instanceSize = 40
```
跟预期的一样，因为只有一个ivar，所以son的start只多了8，那是不是可以证明上面的猜测是对的呢？

回到上面的截图，这个时候那一步if判断是没法通过的，因为24 < 32，这个时候就进到了moveIvars()方法了，再进这个方法之前，先把son的ivars全打印出来，看看offset的原始值：
```
(lldb) p $2.ivars
(const ivar_list_t *const) $4 = 0x0000000100002170
(lldb) p *$4
(const ivar_list_t) $5 = {
  entsize_list_tt<ivar_t, ivar_list_t, 0> = {
    entsizeAndFlags = 32
    count = 3
    first = {
      offset = 0x0000000100002d90
      name = 0x00000001000013e5 "ivarIntSon"
      type = 0x0000000100001ace "q"
      alignment_raw = 3
      size = 8
    }
  }
}
(lldb) p $5.get(0)
(ivar_t) $6 = {
  offset = 0x0000000100002d90
  name = 0x00000001000013e5 "ivarIntSon"
  type = 0x0000000100001ace "q"
  alignment_raw = 3
  size = 8
}
(lldb) p $5.get(1)
(ivar_t) $7 = {
  offset = 0x0000000100002d98
  name = 0x00000001000013f0 "ivarBoolSon"
  type = 0x0000000100001ad0 "c"
  alignment_raw = 0
  size = 1
}
(lldb) p $5.get(2)
(ivar_t) $8 = {
  offset = 0x0000000100002d88
  name = 0x00000001000013fc "_propertyIntSon"
  type = 0x0000000100001ace "q"
  alignment_raw = 3
  size = 8
}
(lldb) p $6.offset
(int32_t *) $9 = 0x0000000100002d90
(lldb) p *$9
(int32_t) $10 = 24
(lldb) p $7.offset
(int32_t *) $11 = 0x0000000100002d98
(lldb) p *$11
(int32_t) $12 = 32
(lldb) p $8.offset
(int32_t *) $13 = 0x0000000100002d88
(lldb) p *$13
(int32_t) $14 = 40
```
$6和$7是直接声明的ivar，排在前2位，属性生成的$8排在后面，打印出各自的offset，第一个ivar的offset即$10就是instanceStart，最后一个offset即$14加上ivar的size就是instanceSize，结果很清晰。

moveIvars()方法前面已经分析过源码了，这里不再赘述，直接看看方法结束之后的结果，在moveIvars()方法之后加一个断点：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_21/moveIvars.png" />
</p>

这个时候ro的start和size已经是这样的了：
```
// ZNObjectSon
instanceStart = 32
instanceSize = 56
```
调整结果符合预期，继续打印出ivar的offset也是没问题的，这里就不截图了。

到这里，前3个问题基本验证完毕了，还剩最后一个问题：
>class_ro_t中的 ivarLayout和 weakIvarLayout是什么意思？

这个问题之所以单独讲，是因为在寻找答案的过程中，出现了一些有趣的结果，怎么个有趣法，一起来看看。

首先依然是一个猜测，weakIvarLayout名字中有个weak，是不是统计weak类型的ivar用的。又因为ivar默认类型是strong，所以ivarLayout是不是用于统计strong类型的ivar呢？
>当然这里默认strong是不针对基本类型的

这时候又要修改一下测试的代码了，son类已经不需要了，只用一个father类就可以了：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    NSInteger ivarInt;
    BOOL ivarBool;
    __strong NSArray *ivarArray;
}
@end
// .m文件就不写了，因为什么也没有
```
runtime也提供了方法用于获取 ivarLayout和 weakIvarLayout
```
const uint8_t *
class_getIvarLayout(Class cls)
{
    if (cls) return cls->data()->ro->ivarLayout;
    else return nil;
}

const uint8_t *
class_getWeakIvarLayout(Class cls)
{
    if (cls) return cls->data()->ro->weakIvarLayout;
    else return nil;
}
```
其实就是返回ro的那两个值，直接用这两个方法就不需要用lldb慢慢打印了，测试的代码是这样的：
```
const uint8_t *ivarLayout = class_getIvarLayout([ZNObjectFather class]);
const uint8_t *weakIvarLayout = class_getWeakIvarLayout([ZNObjectFather class]);
```
使用上面修改之后的father代码测试一下，有趣的事情就发生了：
```
ivarLayout = "!"
weakIvarLayout = NULL
```
说实话，看到这个结果的时候，我的第一反应是: **卧槽，这个!是什么鬼**

第二行为空我装作可以理解，因为没有weak类型的ivar。

我在想，是不是因为在strong之前有两个基本类型，去掉那两个基本类型再试试：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    __strong NSArray *ivarArray;
}
@end

结果:
ivarLayout = "\x01"
weakIvarLayout = NULL
```
这个结果看起来还像点样子，那个01中的1应该就表示有一个strong类型的ivar吧，接着做测试：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    __strong NSArray *ivarArray;
}
@property (nonatomic, weak) NSArray *propertyArrayWeak;
@end

结果:
ivarLayout = "\x01"
weakIvarLayout = "\x11"
```
看到这里我又不能理解了，这个"\x11"怎么解释呢？

没办法，只能搜索一下，发现了[Objective-C Class Ivar Layout 探索](http://blog.sunnyxx.com/2015/09/13/class-ivar-layout/)

这篇文章里面的结果输出并不完全正确，可能作者并没有真正写代码测试吧，但是关于layout编码的规则猜测看起来是没问题的：
> layout 就是一系列的字符，每两个一组，比如 \xmn，每一组 ivarLayout 中第一位表示有 m 个非强属性，第二位表示接下来有 n 个强属性。

再回过去看之前的结果：
- ivarLayout = "\x01"，表示在先有0个弱属性，接着有1个连续的强属性。若之后没有强属性了，则忽略后面的弱属性，对weakIvarLayout也是同理。
- weakIvarLayout = "\x11"，表示先有1个强属性，然后才有1个连续的弱属性。

但是文章中并没有出现过那个神奇的"!"，我继续做测试。
>中间过程比较艰辛，省略无数次结果

直到发现下面这两次结果：
```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    __weak NSArray *ivarArrayWeak;
    __weak NSArray *ivarArrayWeak2;
    __strong NSArray *ivarArray;
}

结果:
ivarLayout = "!"
weakIvarLayout = "\x02"
```
这个感叹号又来了，这个时候根据上面的规则，ivarLayout = "\x21" 才对。

```
// ZNObjectFather.h
@interface ZNObjectFather : NSObject {
    __weak NSArray *ivarArrayWeak;
    __weak NSArray *ivarArrayWeak2;
    __strong NSArray *ivarArray;
    __strong NSArray *ivarArray2;
}

结果:
ivarLayout = "\""
weakIvarLayout = "\x02"
```
居然输出了一个引号(")，结果难道不应该是：ivarLayout = "\x22" 吗？

这个时候我灵光一闪！
>当然在闪之前已经搜索了好久，但没有找到答案，不过这个时候真的是一闪！

我去搜索了ASCII码表，结果真让我猜中了：

<p align="center">
    <img src="http://betterzn.com/assets/images/2017_02_21/ascii.png" />
</p>

所以结果其实是正确的，只是被转成了ASCII码，至于xcode为什么要这么做，我就不得而知了...

## 总结
原本以为很简单的property和ivar，其实一点也不简单，特别是ivar，真的是花了很多时间。顺便把class_ro_t中几个之前没有分析的属性也一并理解了一下，还是很不错的。
- property在编译期会生成_propertyName的ivar，和相应的get/set属性
- ivars在编译期确定，但不完全确定，offset属性在运行时会修改
- 对象的大小是由ivars决定的，当有继承体系时，父类的ivars永远放在子类之前
- class_ro_t的instanceStart和instanceSize会在运行时调整
- class_ro_t的ivarLayout和weakIvarLayout存放的是强ivar和弱ivar的存储规则


