# 内存管理（四）引用计数与weak

在前面的文章中，已经对引用计数以及其在开发中的使用做了初步了解。在本篇中，我们将会深入阐述苹果对引用计数这个技术的底层实现。

# 一、引用计数的存储

我们知道，对于纯量类型的变量，是没有引用计数的，因为不是对象，其申请的变量存储在栈上，由系统来负责管理。

另外一种类型，是前面讲过的`Tagged Pointer`小对象，包括`NSNumber`、`NSString`、`NSDate`等几个类的变量。它们也是存储在栈上，当然也没有引用计数。

整理了下面表格：

![image-20190106224511986](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-144512.png)

关于对象类型的引用计数，其存在的地方有两个：

![image-20190106224908024](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-144908.png)

## 1.1 isa指针里的引用计数

首先，我们来观察对象类型中存放在优化后的`isa指针`，下面是出现过多次的`isa指针`布局：

![image-20181212195446353](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-115447-20210520165535041.png)

## 1.2 Side Table里的引用计数

Side Table在系统中的结构如下：

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/20190710072617.png)而每一张Side Table中针对引用计数的部分如下：

![image-20190106225435341](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-145436.png)

# 二、引用计数管理

在了解了引用计数存储的地方之后，对引用计数管理的理解就更方便了。

## 2.1 管理引用计数的方法

![image-20190107004403705](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-164404-20210520165812017.png)

下面，针对上面的方法，根据源码，进行一定的追踪。

## 2.2 retainCount

后续引用计数相关方法，只看优化过的指针，即只针对64位进行描述。

根据源码，其调用轨迹如下：

> NSObject.mm
>
> ━retainCount()
>
>  ┗━ rootRetainCount()
>
>  ┗━ rootRetainCount()

```objective-c
inline uintptr_t objc_object::rootRetainCount()
{
    // 1. 如果是Tagged Pointer，直接返回
    if (isTaggedPointer()) return (uintptr_t)this;
    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) {  //优化的isa指针
        //2. 取出isa中的retainCount+1
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            //3. 如果Side Table还有存储，则取出累加返回
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
```

### retainCount总结

![image-20190107004642477](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-164643.png)

其中，针对后两种情况：

![image-20190107004705066](http://blog-1251606168.file.myqcloud.com/blog_2018/2019-01-06-164705.png)

对于存在于Side Table中的引用计数，**需要注意**，其最低2位被占用，所以取出来，要右移2位。

## 2.3 retain

源码调用路径如下：

> NSObject.mm
>
> ━retain()
>
>  ┗━objc_retain()
>
>  ┗━obj->retain()
>
>  ┗━rootRetain()

```objective-c
ALWAYS_INLINE id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    // 1. Tagged Pointer对象直接返回
    if (isTaggedPointer()) return (id)this;
    bool sideTableLocked = false;
    // transcribeToSideTable用于表示extra_rc是否溢出，默认为false
    bool transcribeToSideTable = false;
    isa_t oldisa;
    isa_t newisa;
    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);   // 将isa_t提取出来
        newisa = oldisa;
        // 如果对象正在析构，则直接返回nil
        // don't check newisa.fast_rr; we already called any RR overrides
        if (slowpath(tryRetain && newisa.deallocating)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        uintptr_t carry;
        // 2. isa中指针extra_rc+1
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (slowpath(carry)) {
            // 有进位，表示溢出，extra_rc已经不能存储在isa指针了
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) {
                // 如果不处理溢出情况，则在这里会递归调用一次，再进来的时候
                // handleOverflow会被rootRetain_overflow设置为true，从而直接进入到下面的溢出处理流程
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }

            /*
               溢出处理：
               1. 先将extra_rc中引用计数减半，继续存储在isa中。
               2. 同时把has_sidetable_rc设置为true，表明借用了Side Table存储。
               3. 将另一半的引用计数，存放到Side Table中.
             */
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits))); // 将oldisa 替换为 newisa，并赋值给isa.bits(更新isa_t), 如果不成功，do while再试一遍

    if (slowpath(transcribeToSideTable)) {
        // 3. isa的extra_rc溢出，将一半的refer count值放到Side Table中(RC_HALF = 18，extra_rc一共19位)
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }
    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
```

### retain总结：

**Tagged Pointer对象，没有retain。**

**isa中extra_rc若未溢出，则累加1。如果溢出，则将isa和side table对半存储。**

![image-20190107010929482](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-170929.png)

## 2.4 release

源码调用路径：

> NSObject.mm
>
> ━release()
>
>  ┗━ objc_release()
>
>  ┗━ obj->release()
>
>  ┗━ rootRelease()

下面是`release`方法的主要执行流程：

```objective-c
ALWAYS_INLINE bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    // 1. 如果是Tagged Pointer，直接返回false，不需要dealloc
    if (isTaggedPointer()) return false;
    isa_t oldisa;
    isa_t newisa;
 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        // 2. 将当前isa指针中存储的引用计数减1，如果未溢出，返回false，不需要dealloc
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // 3. 减1导致下溢出，则需要从Side Table借位，跳转到下溢出处理
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));

    return false;
 underflow:
    newisa = oldisa;
    if (slowpath(newisa.has_sidetable_rc)) {
        // Side Table存在引用计数
        if (!handleUnderflow) {
            // 如果不处理溢出，此处只会调用一次，handleUnderflow会被rootRelease_underflow设置为ture,
            // 再次进入，则会直接进入溢出处理流程
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc);
        }
        // 3.1 尝试从Side table取出 当前isa能存储引用计数最大值的一半 的引用计数
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF);
        // 如果取出引用计数大于0
        if (borrowed > 0) {
            // 3.2 将取出的引用计数-1，并保存到isa中，保存成功返回false（未被销毁）
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits, 
                                                oldisa.bits, newisa.bits);
            if (!stored) {
                // 3.2.1 保存到isa中失败，再次尝试保存
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }
            if (!stored) {
                // 3.2.2 如果还是保存失败，则将借出来的一半retain count，重新返回给Side Table
                // 并且重新从 2 开始尝试
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }
            return false;
        } else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }
    // 4. 以上流程执行完了，则需要销毁对象，调用dealloc
    if (slowpath(newisa.deallocating)) {
        // 对象正在销毁
        ClearExclusive(&isa.bits);
        if (sideTableLocked) sidetable_unlock();
        return overrelease_error();
        // does not actually return
    }
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
    __sync_synchronize();
    // 销毁对象
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}


// Equivalent to [this autorelease], with shortcuts if there is no override
inline id 
objc_object::autorelease()
{
    if (isTaggedPointer()) return (id)this;
    if (fastpath(!ISA()->hasCustomRR())) return rootAutorelease();

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_autorelease);
}
```

### release总结：

![image-20190107020428943](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-180429.png)

# 三、对象的销毁

## 3.1 dealloc方法

![image-20190107020803294](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-180804.png)

## 3.2 dealloc重写

![image-20190107020819452](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-180819.png)

## 3.3 dealloc源码

源码的执行流程如下：

> ━ dealloc
>
> ┗━_objc_rootDealloc —— <NSObject.mm>
>
> ┗━ rootDealloc —— <objc-object.h>
>
>  ┗━ object_dispose —— <objc-runtime-new.mm>
>
>  ┗━ objc_destructInstance、free —— <objc-runtime-new.mm>

我们跟踪方法`rootDealloc`：

```objective-c
inline void objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?
    // 假如是非指针的isa，即优化的isa
    // 且对应的没有关联对象、weak应用、以及c++析构函数和side_table的引用计数，就直接free释放掉。
    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        // 否则，进入销毁流程
        object_dispose((id)this);
    }
}
```

下面是具体销毁对象的流程：

```objective-c
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);   // 清除成员变量
        if (assoc) _object_remove_assocations(obj); // 移除关联对象
        obj->clearDeallocating();   // 将指向当前的弱指针置为nil
    }

    return obj;
}
```

是的，下一步就是进入`clearDeallocating()`方法。

# 四、weak

weak有一个特性，在对象销毁的时候，指向该对象所有的weak指针都会置为nil，那么这个特性是如何在`dealloc`方法里体现的。

## 4.1 `weak`指针存储对象结构

要了解weak指针的处理，先要了解其对象结构。

weak指针存储在一个个`Side Table`对象中的`weak_table`，这是一张`hash`表。

之后，在`weak_table`中，存储着一个个对象的`entry`表，这也是个`hash`表，每个`entry`就存储着要销毁对象的所有弱引用地址。

Side Table在系统有一组，可根据对象地址获取其对应的SideTable。

如下图所示：

![image-20190107021140377](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-181141.png)

## 4.2 `weak`指针存储的hash表

上面说道`weak`指针的对象结构，如下图，展示的是`weak`指针如何通过`hash`表串联起来，进行存储的。

![image-20190107021614627](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-181615.png)

## 4.3 源码

下面，我们就继续跟踪上面说到的`clearDeallocating()`方法，根据优化过的isa指针，其中调用的：

> ━ clearDeallocating() —— <objc-object.h>
>
>  ┗━ clearDeallocating_slow() —— <NSObject.mm>

```objective-c
NEVER_INLINE void objc_object::clearDeallocating_slow()
{
    assert(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        // 对象有weak指针指向，将会全部置为nil
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        // 引用计数存储在Side Table中，对引用计数进行擦除
        table.refcnts.erase(this);
    }
    table.unlock();
}
```

更进一步，看看是如何清空`weak`指针的。

```objective-c
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
{
    //1. 拿到被销毁对象的指针
    objc_object *referent = (objc_object *)referent_id;
    //2. 通过hash查找对象指针在weak_table中查找出对应的entry
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        return;
    }
    // 3. 将所有的引用设置成nil
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        // 3.1. 如果弱引用超过4个则将referrers数组内的弱引用都置成nil。
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } else {
        // 3.2. 不超过4个则将inline_referrers数组内的弱引用都置成nil
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    // 循环设置所有的引用为nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
        }
    }
    // 4. 从weak_table中移除referent_id对应对象的entry
    weak_entry_remove(weak_table, entry);
}
```

### weak指针处理总结：

![image-20190107022245529](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-182310.png)

# 五、图

## 5.1 对象结构图

![image-20190107022553946](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-182554.png)

## 5.2 存储图

![image-20190107022707773](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-182708.png)







引用计数为零的时候,内存会直接释放吗.

是的会直接释放,[维基百科](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)。MRC下,引用计数为零会直接触发delloc。

