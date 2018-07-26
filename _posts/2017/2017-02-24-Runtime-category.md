---
layout: post
title: Runtime源码 —— 关于category的一个问题
category: Runtime
tags: [Runtime]
---

关于category的文章太多了，有介绍用法的，也有介绍源码的。流传较广的应该算是美团那篇[深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)。

原本我已经不打算写了，但是在我做验证测试的时候发现了一个问题。这个问题在我看过的其他文章中从来没人提起过，包括美团那篇。我花了不少时间来研究，但因为一篇佐证的文章都找不到，所以依然不敢肯定自己的结论。

现在我只能认为是由于runtime的更新导致了不一致。这篇文章，只为记录下这个问题。欢迎对此有更深入理解的小伙伴给我留言，不胜感激。

本文不会对category做系统的分析，如果对category不太了解的请先阅读美团那篇文章。

## 结论
首先看看其他文章中给出的一个结论：
- **分类是在运行时被附加到相关类上的**

我测试下来的结论是：
- **对系统提供的类做分类，分类是在运行时附加到相关类上的**
- **对自己创建的类做分类，分类是在编译期附加到相关类上的**

在其他文章中，根本没有这样的区分，分类全部都是在运行时附加上去的，附加时的调用栈也很清晰：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/attachCategory.png" />
</p>

## 过程

最开始我对其他文章给出的结论是深信不疑的，因为所有的文章都是这么说的。我只是想写代码做一下验证。

为了验证那个结论，我创建了这些类：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/testClass.png" />
</p>

1. 对NSObject和NSViewController创建的分类
2. 对ZNObject创建的分类，ZNObject继承于NSObject
3. 对ZNViewController创建的分类，ZNViewController继承于NSViewController

我在attachCategories()方法里面添加了一点代码，用于打印出class和category，代码如下：
>**我不会分析这个方法，只需要注意我添加了(*)的那一行**

```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);
    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);

（*）    printf("class name is: %s, category name is: %s\n", cls->mangledName(), (*(entry.cat)).name);
        
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
    rw->properties.attachLists(proplists, propcount);
    free(proplists);
    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

运行之前，我预期上面写的4个分类都可以打印出来，格式应该是这样的：
```
预期结果，这4行应该穿插在结果当中：
...
class name is: NSObject, category name is: NSObjAddition
...
class name is: NSViewController, category name is: NSVCAddition
...
class name is: ZNObject, category name is: MyObjAddition
...
class name is: ZNViewController, category name is: MyVCAddition
...
```
结果当然不是这样的，不然我也不会写这篇文章了：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/objAddition.png" />
</p>

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/vcAddition.png" />
</p>

前2个分类都顺利找到了，但是后两个分类却搜索不到。

为了更清晰一点，我把那行printf修改了一下，这样可以过滤掉无关的信息：
```
if (!strcmp((*(entry.cat)).name, "NSObjAddition")
    || !strcmp((*(entry.cat)).name, "MyObjAddition")
    || !strcmp((*(entry.cat)).name, "NSVCAddition")
    || !strcmp((*(entry.cat)).name, "MyVCAddition")) {
    printf("class name is: %s, category name is: %s\n", cls->mangledName(), (*(entry.cat)).name);
}
```

结果当然还是一样的：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/same.png" />
</p>

一共只有两行，而不是预期的四行。

我一度以为是在类首次被调用时，通过realizeClass()方法调用methodizeClass()再调用attachCategories()进行的，然而即使我主动调用了category中的方法，还是不会有信息打印出来。

所以问题就来了，上图中2和3两块的category究竟是什么时候被附加到类上的呢？

猜测：**既然在运行期找不到附加的过程，那是不是编译期就已经做完这个操作了呢？**

为了验证这个猜测，我在之前创建的ZNObject+MyObjAddition分类中添加了一个方法：
```
// ZNObject+MyObjAddition.h
#import "ZNObject.h"
@interface ZNObject (MyObjAddition)
- (void)printMyObjAddition;
@end

// ZNObject+MyObjAddition.m
#import "ZNObject+MyObjAddition.h"
@implementation ZNObject (MyObjAddition)
- (void)printMyObjAddition {
    NSLog(@"MyObjAddition");
}
@end
```
接着在_objc_init()方法的最开始添加了一个断点，然后获取了一下ZNObject类的方法列表：

<p align="center">
    <img src="http://47.100.168.106/assets/images/2017_02_24/znobject.png" />
</p>

可以看到分类的方法这个时候已经在baseMethodList中了，所以说在编译期结束的时候分类就已经附加完成了。

我又对NSObject做了相同的测试，结果是NSObject+NSObjAddition分类的方法在这个时候并没有被添加进来。确实就是在attachCategories()方法中被添加的。

所以才有了文章最开始我放出的那个结论，这里再列一下作为本文的结尾吧：
- **对系统提供的类做分类，分类是在运行时附加到相关类上的**
- **对自己创建的类做分类，分类是在编译期附加到相关类上的**

这就是我测试下来的结论，这个结论看起来实在是太奇怪了。也完全找不到相关文章佐证，所以。。。

如果有同学对此有更深入的研究，请一定留言，不甚感激！

ps:这个过程也告诉我一个信息，即使几乎所有人都那么说，结果也可能是错误的，纸上得来终觉浅呀~