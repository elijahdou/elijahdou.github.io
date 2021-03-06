---
layout: post
title: "weak变量的生命周期及具体实现方法"
description: ""
category: 
tags: [weak, variable, life cycle]
---

#### 本文转自[weak的生命周期：具体实现方法](http://www.cocoachina.com/ios/20150605/11990.html?utm_source=tuicool)，看着不错，有助于理解OC就收藏了。

##### 我们都知道`weak`表示的是一个弱引用，这个引用不会增加对象的引用计数，并且在所指向的对象被释放之后，`weak`指针会被设置的为`nil`。`weak`引用通常是用于处理循环引用的问题，如代理及`block`的使用中，相对会较多的使用到`weak`。

#### 之前对`weak`的实现略有了解，知道它的一个基本的生命周期，但具体是怎么实现的，了解得不是太清晰。今天又翻了翻***《Objective-C高级编程》***关于`__weak`的讲解，在此做个笔记。

##### 我们以下面这行代码为例：

```objc
{
    id __weak obj1 = obj;
}
```

#### 当我们初始化一个`weak`变量时，`runtime`会调用`objc_initWeak`函数。这个函数在`Clang`中的声明如下：
```objc
id objc_initWeak(id *object, id value);
```

##### 其具体实现如下：
```objc
id objc_initWeak(id *object, id value)
{
    *object = 0;
    return objc_storeWeak(object, value);
}
```

##### 示例代码轮换成编译器的模拟代码如下：
```objc
id obj1;
objc_initWeak(&obj1, obj);
```

#### 因此，这里所做的事是先将obj1初始化为0(nil)，然后将obj1的地址及obj作为参数传递给objc_storeWeak函数。

`objc_initWeak`函数有一个前提条件：就是object必须是一个没有被注册为`__weak`对象的有效指针。而`value`则可以是`null`，或者指向一个有效的对象。

如果`value`是一个空指针或者其指向的对象已经被释放了，则`object`是`zero-initialized`的。否则，`object`将被注册为一个指向`value`的`__weak`对象。而这事应该是`objc_storeWeak`函数干的。`objc_storeWeak`的函数声明如下：
```objc
id objc_storeWeak(id *location, id value);
```

##### 其具体实现如下：
```objc
id objc_storeWeak(id *location, id newObj)
{
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;
    ......
    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    oldObj = *location;
    oldTable = SideTable::tableForPointer(oldObj);
    newTable = SideTable::tableForPointer(newObj);
    ......
    if (*location != oldObj) {
        OSSpinLockUnlock(lock1);
#if SIDE_TABLE_STRIPE > 1
        if (lock1 != lock2) OSSpinLockUnlock(lock2);
#endif
        goto retry;
    }
    if (oldObj) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
    if (newObj) {
        newObj = weak_register_no_lock(&newTable->weak_table, newObj,location);
        // weak_register_no_lock returns NULL if weak store should be rejected
    }
    // Do not set *location anywhere else. That would introduce a race.
    *location = newObj;
    ......
    return newObj;
}
```

#### 我们撇开源码中各种锁操作，来看看这段代码都做了些什么。在此之前，我们先来了解下`weak`表和`SideTable`。

#### `weak`表是一个弱引用表，实现为一个`weak_table_t`结构体，存储了某个对象相关的的所有的弱引用信息。其定义如下(具体定义在objc-weak.h中)：
```objc
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    ......
};
```

#### 其中`weak_entry_t`是存储在弱引用表中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用hash表。其定义如下：
```objc
struct weak_entry_t {
    DisguisedPtr referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line : 1;
            ......
        };
        struct {
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```

#### 其中referent是被引用的对象，即示例代码中的obj对象。下面的`union`即存储了所有指向该对象的弱引用。由注释可以看到，当`out_of_line`等于0时，hash表被一个数组所代替。另外，所有的弱引用对象的地址都是存储在`weak_referrer_t`指针的地址中。其定义如下：
```objc
typedef objc_object ** weak_referrer_t;
```

#### SideTable是一个用C++实现的类，它的具体定义在NSObject.mm中，我们来看看它的一些成员变量的定义：
```objc
class SideTable {
private:
    static uint8_t table_buf[SIDE_TABLE_STRIPE * SIDE_TABLE_SIZE];
public:
    RefcountMap refcnts;
    weak_table_t weak_table;
    ......
}
```

#### `RefcountMap refcnts`，大家应该能猜到这个做什么用的吧？看着像是引用计数什么的。哈哈，貌似就是啊，这东东存储了一个对象的引用计数的信息。当然，我们在这里不去探究它，我们关注的是`weak_table`。这个成员变量指向的就是一个对象的`weak`表。

#### 了解了`weak`表和`SideTable`，让我们再回过头来看看`objc_storeWeak`。首先是根据`weak`指针找到其指向的老的对象：
```objc
oldObj = *location;
```

#### 然后获取到与新旧对象相关的SideTable对象：
```objc
oldTable = SideTable::tableForPointer(oldObj);
newTable = SideTable::tableForPointer(newObj);
```

#### 下面要做的就是在老对象的`weak`表中移除指向信息，而在新对象的`weak`表中建立关联信息：
```objc
if (oldObj) {
    weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
}
if (newObj) {
    newObj = weak_register_no_lock(&newTable->weak_table, newObj,location);
    // weak_register_no_lock returns NULL if weak store should be rejected
}
```

#### 接下来让弱引用指针指向新的对象：
```objc
*location = newObj;
```

#### 最后会返回这个新对象：
```objc
return newObj;
```

#### `objc_storeWeak`的基本实现就是这样。当然，在`objc_initWeak`中调用`objc_storeWeak`时，老对象是空的，所有不会执行`weak_unregister_no_lock`操作。

#### 而当`weak`引用指向的对象被释放时，又是如何去处理`weak`指针的呢？当释放对象时，其基本流程如下：

- 调用`objc_release`
- 因为对象的引用计数为0，所以执行`dealloc`
- 在`dealloc`中，调用了`_objc_rootDealloc`函数
- 在`_objc_rootDealloc`中，调用了`object_dispose`函数
- 调用`objc_destructInstance`
- 最后调用`objc_clear_deallocating`

#### 我们重点关注一下最后一步，`objc_clear_deallocating`的具体实现如下：
```objc
void objc_clear_deallocating(id obj) 
{
    ......
    SideTable *table = SideTable::tableForPointer(obj);
    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    OSSpinLockLock(&table->slock);
    if (seen_weak_refs) {
        arr_clear_deallocating(&table->weak_table, obj);
    }
    ......
}
```

#### 我们可以看到，在这个函数中，首先取出对象对应的`SideTable`实例，如果这个对象有关联的弱引用，则调用`arr_clear_deallocating`来清除对象的弱引用信息。我们来看看`arr_clear_deallocating`具体实现：
```objc
PRIVATE_EXTERN void arr_clear_deallocating(weak_table_t *weak_table, id referent) {
    {
        weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
        if (entry == NULL) {
            ......
            return;
        }
        // zero out references
        for (int i = 0; i < entry->referrers.num_allocated; ++i) {
            id *referrer = entry->referrers.refs[i].referrer;
            if (referrer) {
                if (*referrer == referent) {
                    *referrer = nil;
                }
                else if (*referrer) {
                    _objc_inform("__weak variable @ %p holds %p instead of %p\n", referrer, *referrer, referent);
                }
            }
        }
        weak_entry_remove_no_lock(weak_table, entry);
        weak_table->num_weak_refs--;
    }
}
```

#### 这个函数首先是找出对象对应的`weak_entry_t`链表，然后挨个将弱引用置为`nil`。最后清理对象的记录。

#### 通过上面的描述，我们基本能了解一个`weak`引用从生到死的过程。从这个流程可以看出，一个`weak`引用的处理涉及各种查表、添加与删除操作，还是有一定消耗的。所以如果大量使用`__weak`变量的话，会对性能造成一定的影响。那么，我们应该在什么时候去使用`weak`呢？《Objective-C高级编程》给我们的建议是只在避免循环引用的时候使用`__weak`修饰符。

#### 另外，在`clang`中，还提供了不少关于`weak`引用的处理函数。如`objc_loadWeak, objc_destroyWeak`, `objc_moveWeak`等，我们可以在苹果的开源代码中找到相关的实现。等有时间，我再好好研究研究。

> 参考

>《Objective-C高级编程》1.4: __weak修饰符

> Clang 3.7 documentation – Objective-C Automatic Reference Counting (ARC)

> apple opensource – NSObject.mm
