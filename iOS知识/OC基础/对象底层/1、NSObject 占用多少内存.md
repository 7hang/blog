### OC实现原理问题一：NSObject 占用多少内存

```c++
NSObject *obj = [[NSObject alloc] init];

// 获得NSObject实例对象的成员变量所占用的大小 >> 8
NSLog(@"%zd", class_getInstanceSize([NSObject class]));

// 获得obj指针所指向内存的大小 >> 16
NSLog(@"%zu", malloc_size((__bridge const void *)obj));

```



```c++
struct BFPerson_IMP{
	 Class isa;     // 8字节
	 int _age;      // 4字节
	 int _male;     // 4字节
	 double _height;   // 8字节
}
```



继承时，结构体内存是通过变量持有的，可以看做合并为一个结构体。

```c++
struct NSObject_IMPL {
    Class isa;
};

struct BFPerson_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    int _age;
    int _male;
    double _height;
};
```



#### class_getInstanceSize原理

```objective-c
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}

// Class's ivar size rounded up to a pointer-size boundary.
uint32_t alignedInstanceSize() const {
    return word_align(unalignedInstanceSize());  //内存对齐
}

// May be unaligned depending on class's ivars.
uint32_t unalignedInstanceSize() const {
    return data()->ro()->instanceSize;
}
```

可以看出，`class_getInstanceSize`最后返回的就是经过字节对齐，但是在`alloc`中`calloc`之前的大小。

所以我们看到其实际返回的是成员变量实际需要的空间大小。



#### alloc原理

allcoc是对象的实际内存通过`instanceSize`加上`extraBytes`

```c++
static ALWAYS_INLINE id _class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}

id _objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}
```

```c++
inline size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes))) {
        return cache.fastInstanceSize(extraBytes);
    }

    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}
```





[对象是如何初始化的](https://draveness.me/object-init/)

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

[深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)

[从源代码看 ObjC 中消息的发送](https://draveness.me/message/)

[你真的了解 load 方法么？](https://draveness.me/load/)

[懒惰的 initialize 方法](https://draveness.me/initialize/)

[如何实现 iOS 中的 Associated Object](https://draveness.me/retain-cycle3/)

[上古时代 Objective-C 中哈希表的实现](https://draveness.me/hashtable/)

