---
layout: post
title: OC源码 —— autoreleasepool
category: OC
tags: [OC]
---

因为现在普遍使用ARC，所以项目中几乎看不到release这样的字眼了，但是在一个不起眼的地方 —— main.m，有一个@autoreleasepool，本文就是要研究一下这货啦。

```
// iOS项目的main.m
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

## 一个疑问
在我对autoreleasepool有一点理解之后，我就发现了一个奇怪的问题：**既然main函数并不会真的return，那么这个autoreleasepool究竟有什么用呢？**

作为对比，在macOS的项目中，main函数中的return就没有被autoreleasepool包围，是这个样子的：
```
// macOS项目的main.m
int main(int argc, const char * argv[]) {
    return NSApplicationMain(argc, argv);
}
```

难不成iOS项目的@autoreleasepool真的无关紧要？

## autoreleasepool

还是老老实实看一看@autoreleasepool究竟是什么吧。

新建一个iOS项目，用clang重写一下main.m(当然在macOS项目中手写一个@autoreleasepool也没问题)。在main.cpp文件的最后，看到main函数变成了这样：
```
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```
原先的@autoreleasepool没有了，看起来\__AtAutoreleasePool才是其真实面目，搜索一下\__AtAutoreleasePool，它的定义是这样的：
```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```
就一个构造和一个析构函数，分别调用了这两个方法：
```
objc_autoreleasePoolPush()
objc_autoreleasePoolPop(obj)
```
看到这个方法名，首先想到的是栈，难道autoreleasepool的数据结构就是栈吗？

先看看push方法的实现：
```
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```
内部调用了一个c++类的类方法，看看这个类是怎么定义的：
```
class AutoreleasePoolPage 
{
    ...
    static size_t const SIZE = PAGE_MAX_SIZE;
    ...
    magic_t const magic;                   // 16字节
    id *next;                              // 8字节
    pthread_t const thread;                // 8字节
    AutoreleasePoolPage * const parent;    // 8字节 
    AutoreleasePoolPage *child;            // 8字节
    uint32_t const depth;                  // 4字节
    uint32_t hiwat;                        // 4字节
    ...
}

#define PAGE_MAX_SIZE           PAGE_SIZE
#define PAGE_SIZE		       I386_PGBYTES
#define I386_PGBYTES		    4096
```
每个page大小是4096字节，固定字段占用了56字节。剩余的部分都可以用来存放加入到page中的对象地址。

注意这两个字段：
```
AutoreleasePoolPage * const parent;
AutoreleasePoolPage *child;
```
这两个字段说明了pool并不是栈结构，而是一个双向链表。

想要了解这些字段的意思，必须看看这个类的构造函数：
```
AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
    : magic(), next(begin()), thread(pthread_self()),
      parent(newParent), child(nil), 
      depth(parent ? 1+parent->depth : 0), 
      hiwat(parent ? parent->hiwat : 0)
{ 
    if (parent) {
        parent->check();
        assert(!parent->child);
        parent->unprotect();
        parent->child = this;
        parent->protect();
     }
     protect();
}
```
除了parent和child以外，其他字段的含义如下：
- magic
用作校验
- next
通过begin()方法初始化：
```
id * begin() {
    return (id *) ((uint8_t *)this+sizeof(*this));
}
```
返回的是当前page空闲的首位置，即可以用于存放对象地址的第一个位置，即56个字节之后开始的地方。
- thread
当前pool所处的线程
- depth
page的深度，首次为0，以后每次初始化一个page都加1。
- hiwat
这个字段是high water的缩写，这个字段用来计算pool中最多存放的对象个数。在每次执行pop()的时候，会更新一下这个字段。
```
// Check and propagate high water mark
// Ignore high water marks under 256 to suppress noise.
AutoreleasePoolPage *p = hotPage();
uint32_t mark = p->depth*COUNT + (uint32_t)(p->next - p->begin());
if (mark > p->hiwat  &&  mark > 256) {
    for( ; p; p = p->parent) {
        p->unprotect();
        p->hiwat = mark;
        p->protect();
    }
    ...
}
```

## push方法
类的结构已经清楚了，回过去看看push()方法的实现：
```
static inline void *push() 
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}

#   define POOL_BOUNDARY nil
```
因为目的是了解autoreleasepool的真实表现，所以我们只需要关心autoreleaseFast()这个方法：
```
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```
这个方法是autorelease的关键，分成4个部分来研究一下：
- AutoreleasePoolPage *page = hotPage();
- page->add(obj);
- autoreleaseFullPage(obj, page);
- autoreleaseNoPage(obj);

### part1
```
AutoreleasePoolPage *page = hotPage();

static inline AutoreleasePoolPage *hotPage() 
{
    AutoreleasePoolPage *result = (AutoreleasePoolPage *) tls_get_direct(key);
    if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
    if (result) result->fastcheck();
    return result;
}
```
看看hotPage()方法的源码，这里使用了tls的方法来存取当前的page。

#### TLS
TLS是线程局部存储(Thread Local Storage)的缩写，在oc中通过这两个方法来使用：
```
static inline void *tls_get_direct(tls_key_t k) 
{ 
    assert(is_valid_direct_key(k));

    if (_pthread_has_direct_tsd()) {
        return _pthread_getspecific_direct(k);
    } else {
        return pthread_getspecific(k);
    }
}

static inline void tls_set_direct(tls_key_t k, void *value) 
{ 
    assert(is_valid_direct_key(k));

    if (_pthread_has_direct_tsd()) {
        _pthread_setspecific_direct(k, value);
    } else {
        pthread_setspecific(k, value);
    }
}
```
其实就是通过键值对来存取信息，但是很快我发现了一个问题：这个key是定义在AutoreleasePoolPage类中的：
```
static pthread_key_t const key = AUTORELEASE_POOL_KEY;
#define AUTORELEASE_POOL_KEY  ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY3)
#define __PTK_FRAMEWORK_OBJC_KEY3	43
```
但是如果key是全局的，那么对于不同的线程来说，通过相同的key不是会取到相同的page吗，那么不同线程的autoreleasepool之间不是乱套了吗？

当然这个担心是多余的，我用c++测试了一下，即使key是全局的，不同线程通过这个key操作的也是不同的内存区域，不会互相影响。

回过去看hotPage()，就可以理解成当前使用的这个page，第一次调用push的时候，当然还没有page，所以通过hotPage()是获取不到的。

所以我们先看一看最后那个部分。

### part2 
```
autoreleaseNoPage(obj)

static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    ...
    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        ...
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        return setEmptyPoolPlaceholder();
    }
    
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
    
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }
    
    return page->add(obj);
}

static inline bool haveEmptyPoolPlaceholder()
{
    id *tls = (id *)tls_get_direct(key);
    return (tls == EMPTY_POOL_PLACEHOLDER);
}
```
第一次调用这个方法时，haveEmptyPoolPlaceholder()返回的是false，所以会进入最后一个if判断中，调用setEmptyPoolPlaceholder()：
```
static inline id* setEmptyPoolPlaceholder()
{
    assert(tls_get_direct(key) == nil);
    tls_set_direct(key, (void *)EMPTY_POOL_PLACEHOLDER);
    return EMPTY_POOL_PLACEHOLDER;
}
```
这个方法调用完之后，再次进入autoreleaseNoPage()时，就会进入第一个if判断中了。接着就会走到这个方法的最后那部分，创建一个新page，设置为hotpage。给page中添加一个边界标记，最后把将要autorelease的对象添加进来。

关于整个调用流程，最后会做一下汇总。

### part3
```
page->add(obj);

id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```
这个方法很简单，就是给把obj添加到page中，然后把next指针向后移动一步。

### part4
```
autoreleaseFullPage(obj, page);

static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full. 
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
如果一个page满了，那么就需要新建一个页，然后把链表的链条接上。最后同样把新建的页设为hotpage，并把obj添加进来。

## autorelease方法
```
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```
这个方法也很简单，就调用了autoreleaseFast()，只不过这个时候参数是调用这个方法的对象，而不是push方法中的边界标识。

所以autorelease的过程并不是对象真正释放的过程，这也是autorelease的作用，说白了就是延迟释放的时机。真正释放的时机是在@autoreleasepool块结束的时候，那个时候会调用pop()方法。

## pop方法
```
static inline void pop(void *token) 
{
    AutoreleasePoolPage *page;
    id *stop;
(*******part1*******)
    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        if (hotPage()) {
            pop(coldPage()->begin());
        } else {
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);

(*******part2*******)
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
        } else {
            return badPop(token);
        }
    }

(*******part3*******)
    ...
    page->releaseUntil(stop);
    ...
    if (page->child) {
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```
这个方法大致可以分为3个部分，
- part1部分，判断token是否是EMPTY_POOL_PLACEHOLDER，这个东西是autoreleasepool首次push的时候返回的，也就是最顶层的pop会执行这一部分
- part2部分，这部分不太理解，主要是不清楚什么时候token会不等于POOL_BOUNDARY，先略去。
>经[@KylinRoc](http://www.jianshu.com/u/5222ca681583)提示，在非ARC情况下，在新创建的线程中不使用autoreleasepool，直接调用autorelease方法时会出现这个情况。此时没有pool，直接进行autorelease。详细情况请参考评论。
- part3部分，多数情况下，都会进入到这一部分。重点说一下这个部分。

我们可以先把autoreleasepool运行的过程大概表示一下：
```
token1 = push()
...
    token2 = push()
    ...
        token3 = push()
        ...
        pop(token3)
    ...
    pop(token2)
...
pop(token1)
```
每次pop，实际上都会把最近一次push之后添加进去的对象全部release掉，所以autoreleasepool是nest结构完全没问题。

在part3开始前，先通过token获取了一下当前的page：
```
page = pageForPointer(token);

static AutoreleasePoolPage *pageForPointer(const void *p) 
{
    return pageForPointer((uintptr_t)p);
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    assert(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}
```
这个方法可以缩写为以下几步：
```
uintptr_t offset = p % SIZE;
result = (AutoreleasePoolPage *)(p - offset);

例如：
p = 0x100623bc2
offset = p % 4096 = 0xbc2
result = p - offset = 0x100623000
```
获取到page的地址之后，就可以调用release方法了：
```
page->releaseUntil(stop);

void releaseUntil(id *stop) 
{
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();

        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}
```
从next指针开始，一个一个向前调用release方法，直到碰到push时压入的token为止。

最后就是kill child的过程：
```
if (page->child) {
    if (page->lessThanHalfFull()) {
        page->child->kill();
    }
    else if (page->child->child) {
        page->child->child->kill();
    }
}
```
如果当前page小于一半满，则把当前页的所有孩子都杀掉，否则，留下一个孩子，从孙子开始杀。正是因为这一步，在autoreleaseFullPage()方法中才会有这两步：
```
if (page->child) page = page->child;
else page = new AutoreleasePoolPage(page);
```
而不是直接new一个新的page出来。
>这么做是不是出于性能的考虑？

## 总结

汇总一下autoreleasepool的调用流程：
1. 使用@autoreleasepool标记，调用push()方法；
2. 没有hotpage，调用autoreleaseNoPage()方法，设置EMPTY_POOL_PLACEHOLDER；
3. 在@autoreleasepool块内部，第一次有对象调用autorelease，进而调用autoreleaseFast()，此时依然没有hotpage，进入autoreleaseNoPage()；
>这么说是不准确的，通常在我们的代码第一次调用autorelease的时候，page已经被创建出来了，但没关系，意思还是这么个意思。
4. 因为在2设置了EMPTY_POOL_PLACEHOLDER，所以会设置本页为hotpage，添加边界标记POOL_BOUNDARY，最后添加obj；
5. 继续有对象调用autorelease，此时已经有了page，调用page->add(obj)；
6. 如果page满了，调用autoreleaseFullPage()创建新page，重复步骤5；
7. 到达autoreleasepool边界，调用pop方法，通常情况下会释放掉POOL_BOUNDARY之后的所有对象。

autoreleasepool和runloop关系挺大，后面写runloop的时候再分析用法吧。