---
layout: post
title: Runtime源码 —— Associated Object
category: Runtime源码阅读
tags: [Runtime]
---

这玩意儿已经在前面的文章里多次提到，但一直没深入，这一篇就来研究研究。

runtime提供的和associated object有关的接口有3个：
```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object) ;
```
选第一个作为切入点，详细分析一下，其他两个方法稍微说一说。

***

## objc_setAssociatedObject
```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}

void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
```
想要理解associated object的存取过程，就必须要对这个方法中提到的几个类有足够的了解，按照层次依次是：
- AssociationsManager
```
// class AssociationsManager manages a lock / hash table singleton pair.
// Allocating an instance acquires the lock, and calling its assocations() method
// lazily allocates it.
class AssociationsManager {
    static spinlock_t _lock;
    static AssociationsHashMap *_map;               // associative references:  object pointer -> PtrPtrHashMap.
public:
    AssociationsManager()   { _lock.lock(); }
    ~AssociationsManager()  { _lock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
```
注释写了，AssociationsManager管理了一个自旋锁到哈希表的单例映射。通过associations()方法可以取到管理的AssociationsHashMap单例。
- AssociationsHashMap
```
class AssociationsHashMap : public unordered_map
<disguised_ptr_t, 
ObjectAssociationMap *, 
DisguisedPointerHash, 
DisguisedPointerEqual, 
AssociationsHashMapAllocator> {
public:
    void *operator new(size_t n) { return ::malloc(n); }
    void operator delete(void *ptr) { ::free(ptr); }
};
```
这是一个无序哈希表，存储的是对象地址(**set方法的第一个参数取反**)到ObjectAssociationMap的映射。
- ObjectAssociationMap
```
class ObjectAssociationMap : public std::map
<void *, 
ObjcAssociation, 
ObjectPointerLess, 
ObjectAssociationMapAllocator> {
public:
    void *operator new(size_t n) { return ::malloc(n); }
    void operator delete(void *ptr) { ::free(ptr); }
};
```
此表存储了key(**set方法的第二个参数**)到被关联对象ObjcAssociation的映射。
- ObjcAssociation
```
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}

    uintptr_t policy() const { return _policy; }
    id value() const { return _value; }
    
    bool hasValue() { return _value != nil; }
};
```
存储关联对象的信息，**_value对应第三个参数**，**_policy对应第四个参数**。

至此set方法的四个参数都用上了，再回过去把set方法分解一哈就很简单了：

1.
```
ObjcAssociation old_association(0, nil);
```
先声明一个对象用来存放可能存在的旧的value，即如果你对一个对象调用了两次set方法，那么在第二次set的时候，需要把第一个set进去的value释放掉。这个对象就是存储前一次的value用于释放。

2.
``` 
id new_value = value ? acquireValue(value, policy) : nil;

static id acquireValue(id value, uintptr_t policy) {
    switch (policy & 0xFF) {
    case OBJC_ASSOCIATION_SETTER_RETAIN:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
    case OBJC_ASSOCIATION_SETTER_COPY:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_copy);
    }
    return value;
}

typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,    
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, 
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  
    OBJC_ASSOCIATION_RETAIN = 01401,      
    OBJC_ASSOCIATION_COPY = 01403          
};
```
根据调用set方法传的policy调用对应的方法。比如retain或者copy。

3.
```
AssociationsManager manager;
AssociationsHashMap &associations(manager.associations());
disguised_ptr_t disguised_object = DISGUISE(object);

inline disguised_ptr_t DISGUISE(id value) { return ~ uintptr_t(value); }
typedef uintptr_t disguised_ptr_t;
```
先获取AssociationsManager，在构造函数中会进行加锁。接着通过associations()方法获取AssociationsHashMap单例，如果前两步看起来有些不习惯，重写一下就清楚了：
```
AssociationsManager manager = AssociationsManager();
AssociationsHashMap &associations = manager.associations();
```
最后一行就是把object的地址取反，后面会用作AssociationsHashMap的键。

4.
```
if (new_value) {
    AssociationsHashMap::iterator i = associations.find(disguised_object);
(1)  if (i != associations.end()) {
        ObjectAssociationMap *refs = i->second;
        ObjectAssociationMap::iterator j = refs->find(key);
(2)     if (j != refs->end()) {
            old_association = j->second;
            j->second = ObjcAssociation(policy, new_value);
        } else {
            (*refs)[key] = ObjcAssociation(policy, new_value);
        }
    } else {
        ObjectAssociationMap *refs = new ObjectAssociationMap;
        associations[disguised_object] = refs;
        (*refs)[key] = ObjcAssociation(policy, new_value);
        object->setHasAssociatedObjects();
    }
}
```
先判断是否传入了新的value，如果传入新的value，就进入了设置关联对象的流程。

首先通过上一步处理好的对象地址进行查找：

- (1) == true
根据AssociationsHashMap的定义，直接获取ObjectAssociationMap。

- (2) == true
代表曾经设置过关联对象，把原先的值存到old_association中，留作后面释放，再把新值通过ObjcAssociation(policy, new_value)构造函数存放进去。

- (2) == false
通过key找不到关联对象，直接构造一个新的ObjcAssociation对象作为key的value。

- (1) == false
即通过对象地址就查找不到，代表从未设置过关联对象，那么该创建的创建，该关联的关联。最后通过：
```
object->setHasAssociatedObjects()
```
将isa的has_assoc字段设为true。
>随着源代码的阅读，最开始那篇:Runtime源码 —— 对象、类和isa 中对isa有些不熟悉的字段也就一个一个见到了。有种融会贯通的感觉，哈哈。

5.
```
else {
    // setting the association to nil breaks the association.
    AssociationsHashMap::iterator i = associations.find(disguised_object);
    if (i !=  associations.end()) {
        ObjectAssociationMap *refs = i->second;
        ObjectAssociationMap::iterator j = refs->find(key);
        if (j != refs->end()) {
            old_association = j->second;
            refs->erase(j);
        }
    }
}
```
如果调用set方法没有传入新的值，那么就把关联对象从ObjectAssociationMap中删除掉。就好像把对象设为nil要release一样。

6.
```
// release the old value (outside of the lock).
if (old_association.hasValue()) ReleaseValue()(old_association);

struct ReleaseValue {
    void operator() (ObjcAssociation &association) {
        releaseValue(association.value(), association.policy());
    }
};

static void releaseValue(id value, uintptr_t policy) {
    if (policy & OBJC_ASSOCIATION_SETTER_RETAIN) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_release);
    }
}
```
最后一步，把原先的值释放掉。当然只有policy是retain的才需要释放，assign和copy的的就不需要了。

这就是set方法的全部过程。因为嵌套了两层map，看起来有点绕。

***

## objc_getAssociatedObject

理解了set的过程，再通过get方法的参数：
```
id _object_get_associative_reference(id object, void *key)
```
猜测一下get的过程应该是这样的：
- 先获取AssociationsManager单例，进而获取AssociationsHashMap
- 通过object获取ObjectAssociationMap
- 通过key获取ObjcAssociation
- 取出ObjcAssociation中的value并返回

看起来合情合理，再看看源码：
```
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
    }
    return value;
}
```
跟预测的流程几乎一样，只是从ObjcAssociation中获取value的时候，需要根据policy进行相应的retain，autorelease。

***

## objc_removeAssociatedObjects

这个方法用于删除对象的全部关联对象，参数就一个，就是想要清理的对象：
```
void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}

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
先判断是否存在关联对象，如果存在，才需要进行清除。

清除的过程分为几个部分：
- 存储ObjectAssociationMap中存在的所有的ObjcAssociation
- 释放ObjectAssociationMap内存并从AssociationsHashMap中删除
- 释放所有的ObjcAssociation，最后那个ReleaseValue()方法就是对有需要的对象调用release

全部完成之后AssociationsHashMap中就不存在此对象的任何关联对象了。

***

## 总结

- 被关联的对象与关联对象在数据结构上并没有什么关系，关联对象是由AssociationsManager统一管理
- 通常情况下不需要主动调用objc_removeAssociatedObjects(...)方法，移除关联对象会在对象release -> dealloc的时候自动调用，如果想要移除某个关联对象，调用objc_setAssociatedObject(...)方法，并设置value为nil。
- 在category中通过关联对象实现的get/set接口只是让属性看起来像是属性，category本身还是无法添加实例变量。