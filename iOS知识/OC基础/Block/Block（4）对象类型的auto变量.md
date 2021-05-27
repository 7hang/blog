# Block（四）对象类型的auto变量

我们在针对`Block`的剖析在进一步加深，了解了`Block`如何截获基本类型，了解Block的类型和`copy`操作，下面我们开始进入Block的**内存管理世界**。

Block的内存管理，主要针对捕获外部对象类型的`auto`变量。

```
BFPerson *person = [[BFPerson alloc] init];
person.age = 28;
void (^block)(void) = ^{
    NSLog(@"age %d", person.age);
};
block();
```

针对对象，**仍然适用**。

- `auto`变量捕获后，`Block`中变量的类型和变量原类型一致；
- `static`变量捕获后，`Block`对应的变量是对应变量的指针类型；

那么，`auto`对象与基本类型在Block内部有什么区别呢。

我们将从两方面讨论：

1. `Block`是如何捕获对象类型的？
2. `Block`内部是如何管理对象类型的？

# 一、捕获对象类型

## 1. `Block`对象结构体

将下面代码重写：

```objective-c
- (void)testBlockCaptrueAutoObject
{
    BFPerson *person = [[BFPerson alloc] init];
    person.age = 28;
    void (^block)(void) = ^{
        NSLog(@"age %d", person.age);
    };
    block();
}
```

以上`Block`对象重写后的结构体如下：

```c++
struct __Test__test_block_impl_0 {
	struct __block_impl impl;
	struct __Test__test_block_desc_0* Desc;
	BFPerson *person;
	.....
};
```

可以看到：
`Block`对象仍然捕获`auto`变量后，保留了`person`对象的类型。

## 2.`Desc`及`Func`

进一步观察`Block`对象中的`Desc`结构以及`Func`：

```c++
//Func
static void __Test__test_block_func_0(struct __Test__test_block_impl_0 *__cself) {
    BFPerson *person = __cself->person; // bound by copy
    .....
}

//Desc
static struct __Test__test_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __Test__test_block_impl_0*, struct __Test__test_block_impl_0*);
  void (*dispose)(struct __Test__test_block_impl_0*);
} __Test__test_block_desc_0_DATA = { 0, sizeof(struct __Test__test_block_impl_0), __Test__test_block_copy_0, __Test__test_block_dispose_0};
```

那么与基本类型的捕获区别在哪里呢？

我们观察与之前基本类型的区别，`Func`基本没有区别，`Desc`有区别，多了两个函数指针：

- void (*copy)
- void (*dispose)

那么继续查看这两个函数：

```c++
//copy函数
static void __Test__test_block_copy_0(struct __Test__test_block_impl_0*dst, struct __Test__test_block_impl_0*src) 
{
    _Block_object_assign((void*)&dst->person, (void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

//dispose函数
static void __Test__test_block_dispose_0(struct __Test__test_block_impl_0*src) 
{
    _Block_object_dispose((void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

针对这两个函数，它们的作用就是：

| 函数                         | 作用                                                         | 调用时机              |
| ---------------------------- | ------------------------------------------------------------ | --------------------- |
| __Test__test_block_copy_0    | 调用 _Block_object_assign，相当于**retain**，将对象赋值在对象类型的结构体变量 __Test__test_block_impl_0中。 | 栈上的Block复制到堆时 |
| __Test__test_block_dispose_0 | 调用 _Block_object_dispose，相当于**release**，释放赋值在对象类型的结构体变量中的对象。 | 堆上Block被废弃时     |

# 二、内存管理

在我们观察了`Block`对象捕获对象类型的内部结构之后，我们基本就能了解`Block`内部是如何管理的：

- 在栈上的Block对象复制到堆上，对`person`进行`retain`；
- 在堆上Block被废弃时，对`person`进行废弃；

以上操作在`ARC`环境下，由系统帮助我们完成，但在`MRC`下，我们仍然要自己管理。

下面，我们就一步一步探索验证不同情境下的对象类型内存管理。

## 2.1 Block在栈上

我们将项目调成`MRC`环境，并在类`BFPerson`重写`dealloc`

```objective-c
@implementation BFPerson
- (void)dealloc
{
    [super dealloc];        //MRC下打开，ARC下注释
    NSLog(@"BFPerson delloc");
}
@end
```

测试如下代码：

![image-20181209111540551](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-031541.png)



可以看到，在`[person release]`后，`Block`内部再次访问直接崩溃，说明`Block`内并没有对`person`对象进行强引用，使得`person`在内存中释放。

总结：

**在栈空间，Block不会对auto变量进行强引用。**

## 2.2 Block在堆上

### 2.1 Block强引用对象类型

我们将2.1，即上节中的代码，在ARC环境下再次试验，打印结果如下：

> **Block捕获对象类型[94670:4794884] begin**
>
> **Block捕获对象类型[94670:4794884] class: __NSMallocBlock__**
>
> **Block捕获对象类型[94670:4794884] age 28**
>
> **Block捕获对象类型[94670:4794884] end**
>
> **Block捕获对象类型[94670:4794884] BFPerson delloc**

针对以上结果：

1. `ARC`情况下，在`Block`赋值给`__strong`指针时，栈上的`Block`自动拷贝到堆；
2. 拷贝的同时会将`person`对象进行`retain`操作，在这里相当于强引用；
3. 调用`block`时，即使`person`出了大括号，系统会`release`一次，但由于`block`内部仍然强引用`person`，所以不会销毁；
4. 在打印”end”后，`block`销毁，那么对`person`将进行一次`release`操作，所以`person`对象销毁。

### 2.2 Block弱引用对象类型

我们将`block`内部引用的auto变量改为`__weak`，我们知道`__weak`表示的是弱引用指针。

```objective-c
BFBlock block;
NSLog(@"begin");
{
    BFPerson *person = [[BFPerson alloc] init];
    person.age = 28;
    __weak BFPerson *weakSelf = person;
    block = ^{
        NSLog(@"age %d", weakSelf.age);
    };
    NSLog(@"class: %@", [block class]);
}
block();
NSLog(@"end");
```

上面代码输出的测试结果如下：

> **Block捕获对象类型[94923:4823185] begin**
>
> **Block捕获对象类型[94923:4823185] class: __NSMallocBlock__**
>
> **Block捕获对象类型[94923:4823185] BFPerson delloc**
>
> **Block捕获对象类型[94923:4823185] age 0**
>
> **Block捕获对象类型[94923:4823185] end**

从输出结果我们很明显的看出，只要出了大括号，`person`对象就被销毁了，此后`block`调用，获取到的结果是0，这其实并不是`person`对象的`age`值。

我们将上面代码重写为C++代码，看看Block内部到底发生了什么？

```
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m
```

此时发生错误：

![block代码转C++缺少运行时错误](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-063549.png)

针对上面问题，我们制定`ARC`下的运行时系统版本即可：

```
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m
```

得到的Block对象结构体：

```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  BFPerson *__weak weakSelf;		//weak变量
  ....	
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  BFPerson *__weak weakSelf = __cself->weakSelf; // bound by copy
  ....
}

static void __main_block_copy_0(
    struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
    _Block_object_assign((void*)&dst->weakSelf, (void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) 
{
    _Block_object_dispose((void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```

其实上面大体和强引用对象类似，只是其对应`copy`和`dispose`函数针对强引用和弱引用有所不同。

# 三、 总结

![对象类型的auto变量](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-074827.png)