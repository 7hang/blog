# Block（五）__block变量

在前几篇，针对Block截获变量的类型已经作了全部说明——包括基本类型和对象类型。

下面，我们介绍Block中常用的一个修饰符`__block`。

# 一、`_block`的使用

在开发中，Block内部能直接修改全局变量或者static变量，而对auto变量，就不能。`__block`就用于在`Block`内部修改`auto`变量的值。

```objective-c
__block int age = 20;
__block BFPerson *person = [[BFPerson alloc] init];

void(^block)(void) = ^ {
    age = 30;
    person = [[BFPerson alloc] init];
    NSLog(@"age is %d", age);
    NSLog(@"person is %@", person);
};
block();
```

上面展示了，`__block`是如何修改`auto`变量的。

# 二、`__block`的底层结构

## 2.1 Block结构体

先观察`Block`对象的结构，以及转换后的`__block`变量结构体：

```objective-c
struct __Block_byref_age_0 {
    void *__isa;
    __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age;
};
struct __Block_byref_person_1 {
    void *__isa;
    __Block_byref_person_1 *__forwarding;
    int __flags;
    int __size;
    void (*__Block_byref_id_object_copy)(void*, void*);
    void (*__Block_byref_id_object_dispose)(void*);
    BFPerson *person;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_age_0 *age; // by ref
    __Block_byref_person_1 *person; // by ref
    .....
};
```

从上面的结构体中可以看出，`age`和`person`最后都转换成了一个新的结构体。而且这个新的结构体的第一个成员变量是`isa`指针，即这个是一个对象。所以得出第一个结论：

**结论1 ：`__block`变量转换为一个对象__Block_byref_\**；**

整个转换过程和结构如下：

![__block转换过程](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-121613.png)

`__Block_byref_***`对象其中各个元素如上图：

- `isa`指针
- `__forwarding`指针
- val是原变量
- `_size`及`_flags`不是很重要的元素，忽略不谈。

## 2.2 Block代码中调用

```c++
//以下代码
age = 30;
person = [[BFPerson alloc] init];
//转换为
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_age_0 *age = __cself->age; // bound by ref
    __Block_byref_person_1 *person = __cself->person; // bound by ref
    (age->__forwarding->age) = 30;
    (person->__forwarding->person) = 
        ((BFPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((BFPerson *(*)(id, SEL))(void *)objc_msgSend)
                                                       ((id)objc_getClass("BFPerson"), sel_registerName("alloc")), sel_registerName("init"));
}
```

以上代码，我们之前见过很多次，但是不同于之前的讨论过的情形：

- 全局变量，Block不会捕获，使用时直接访问；
- `auto`基本类型变量，Block捕获其变量，存储于Block内部，是值传递；
- `static`基本类型局部变量，Block捕获其变量指针，存储于Block内部，是指针传递；
- `auto`对象类型，Block捕获其变量及其**所有权修饰符**，即强引用、弱引用等修饰符；

然而在此处，Block针对`__block`变量，则是另一种处理方式，将其封装为为对象后，使用时访问该变量又略有不同。

> (age->__forwarding->age) = 30;
> (person->__forwarding->person = ….

我们整理了如下图：

![block捕获的变量类型](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-023247.png)

## 2.3 `__block`变量对象结构

### 2.3.1`isa`

`isa`指针作为对象的标记之一，不用多说。

### 2.3.2`__forwarding`

这是一个很奇怪的指针，从图中可以看出，它指向的是自己，如下图所示：

![__forwarding指针的作用](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-125428.png)

为什么会指向自己呢，原因是当栈中的Block复制到堆中的时候，在**栈中仍然能正确访问堆中的变量。**

下面我们就针对此，做一个小实验：

```objective-c
@autoreleasepool {
    __block int age = 20;
    __block BFPerson *person = [[BFPerson alloc] init];

    void(^block)(void) = ^ {
        age = 30;
        person = [[BFPerson alloc] init];
        NSLog(@"malloc age address: %p", &age);
        NSLog(@"malloc age is %d", age);
    };
    block();
    NSLog(@"stack age address: %p", &age);
    NSLog(@"stack age is %d", age);
}
```

上面打印的结果：

![_block变量在栈上与堆上的地址](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-09-125928.png)

可以看出，在栈中和堆中`age`的值只有一个，地址也是相同的。即均指向堆中的`age`值，也要特别注意，这个`age`变量，是__block修饰后，转换的结构体中的age变量，而不同于未加__block修饰，直接存在于栈中的age变量值。

### 2.3.3 val

我们在上面`Block`代码调用封装`__block`变量的值时，是通过`__forwarding`指针调用的，如下：

> (age->__forwarding->age) = 30;
> (person->__forwarding->person = ….

`val`就指封装成对应的`__Block_byref_***`后原来变量的值。比如`age`，就是

`__Block_byref_age_0->age`。

# 三、`__block`变量内存管理

上面在描述`__block`变量封装成对象后，一直没有讲述`Block`对象对应的`Desc`结果，根据之前的文章，我们了解到`Desc`就是描述如何管理内存的结构体。我们先回顾一下之前的一些捕获变量或对象是如何管理内存的。

**注：下面“干预”是指不用程序员手动管理，其实本质还是要系统管理内存的分配与释放。**

- `auto`局部基本类型变量，因为是值传递，内存是跟随Block，不用干预；
- `static`局部基本类型变量，指针传递，由于分配在静态区，故不用干预；
- 全局变量，存储在数据区，不用多说，不用干预；
- 局部对象变量，如果在栈上，不用干预。但`Block`在拷贝到堆的时候，对其`retain`，在`Block`对象销毁时，对其`release`；

在这里，`__block`变量呢？

很简单，看`Desc`，但也要注意一个点：

**注意点就是：`__block`变量在转换后封装成了一个新对象，内存管理会多出一层。**

## 3.1 基本类型的Desc

上述`age`是基本类型，其转换后的结构体为：

```c++
struct __Block_byref_age_0 {
    void *__isa;	
    __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age;
};
```

而`Block`中的`Desc`如下：

```c++
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

//下面两个方法是Block内: 内存管理相关函数
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _Block_object_assign((void*)&dst->age, (void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_assign((void*)&dst->person, (void*)src->person, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _Block_object_dispose((void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_dispose((void*)src->person, 8/*BLOCK_FIELD_IS_BYREF*/);
}
```

针对基本类型，以`age`类型为例：

- `__Block_byref_age_0`对象同样是在`Block对象`从栈上拷贝到堆上，进行`retain`；
- 当`Block对象`销毁时，对`__Block_byref_age_0`进行`release`；
- `__Block_byref_age_0`内`age`，由于是基本类型，是不用进行内存手动干预的。

## 3.2 对象类型的Desc

下面看`__block`对象类型的转换：

```c++
struct __Block_byref_person_1 {
    void *__isa;		//地址0---占用8字节
    __Block_byref_person_1 *__forwarding;	//地址8-16---占用8字节
    int __flags;		//地址16-20---占用4字节
    int __size;			//地址20-24---占用8字节
    void (*__Block_byref_id_object_copy)(void*, void*);	//地址24-32---占用8字节
    void (*__Block_byref_id_object_dispose)(void*);     //地址32-40---占用8字节
    BFPerson *person; 	//地址40-48---占用8字节
};
```

当然，针对`Desc`，在[基本类型的Desc](#3.1 基本类型的Desc)中已经贴出来了。但我们还观察到，因为捕获的本身是一个对象类型，所以该对象类型还需要进行内存是的干预。

这里有两个熟悉的函数，即用于管理对象auto变量时，我们见过，用于管理对象auto的内存：

```
void (*__Block_byref_id_object_copy)(void*, void*);
void (*__Block_byref_id_object_dispose)(void*);
```

那么这两个函数对应的实现，我们也找出来：

### 3.2.1 初始化__block对象

下面针对转换来转换去的细节做了删减，方便阅读：

```c++
struct __Block_byref_person_1 person = {
    0,	//isa
    &person,	//__forwarding
    33554432, 	//__flags
    sizeof(__Block_byref_person_1),		//__size
    __Block_byref_id_object_copy_131,  //(*__Block_byref_id_object_copy)
    __Block_byref_id_object_dispose_131,// (*__Block_byref_id_object_dispose)
    (objc_msgSend)((objc_msgSend)(objc_getClass("BFPerson"), sel_registerName("alloc")), sel_registerName("init"))
};

//注意此处+40字节
static void __Block_byref_id_object_copy_131(void *dst, void *src
{
	_Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) 
{
    _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```

 我们注意观察，在`__Block_byref_id_object_copy_131`和`__Block_byref_id_object_dispose_131`函数中，都会偏移40字节，我们再看`__block BFPerson`对象转换后的`__Block_byref_person_1`结构体发现，其40字节偏移处就是原本的`BFPerson *person`对象。

### 3.2.2 对象类型的内存管理

以`BFPerson *person`，在`__block`修饰后，转换为：`__Block_byref_person_1`对象：

- ```
  __Block_byref_person_1
  ```

  对象同样是在

  ```
  Block对象
  ```

  从栈上拷贝到堆上，进行

  ```
  retain
  ```

  ；

  - 当`__Block_byref_person_1`进行`retain`同时，会将`person`对象进行retain

- 当

  ```
  Block对象
  ```

  销毁时，对

  ```
  __Block_byref_person_1
  ```

  进行

  ```
  release
  ```

  - 当`__Block_byref_person_1`对象`release`时，会将`person`对象`release`

### 3.2.3 与auto对象变量的区别

![auto对象变量与_block变量对比](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-032731.png)

# 四、从栈到堆

Block从栈复制到堆时，__block变量产生的影响如下：

| __block变量的配置存储域 | Block从栈复制到堆的影响     |
| ----------------------- | --------------------------- |
| 栈                      | 从栈复制到堆，并被Block持有 |
| 堆                      | 被Block持有                 |

## 4.1 Block从栈拷贝到堆

![block从栈拷贝到堆](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-032134-20210521105853316.png)

当有多个Block对象，持有同一个__block变量。

- 当其中任何Block对象复制到堆上，__block变量就会复制到堆上。
- 后续，其他Block对象复制到堆上，__block对象引用计数会增加。
- Block复制到堆上的对象，持有__block对象。

## 4.2 Block销毁

![block的销毁](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-032345.png)

## 4.3 总结

![__block变量的内存管理规律](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-032552.png)

# 五、更多的细节

## 5.1 `__block`捕获变量存放在哪？

```
__block int age = 20;
__block BFPerson *person = [[BFPerson alloc] init];

void(^block)(void) = ^ {
    age = 30;
    person = [[BFPerson alloc] init];
    NSLog(@"malloc address: %p %p", &age, person);
    NSLog(@"malloc age is %d", age);
    NSLog(@"person is %@", person);
};
block();
NSLog(@"stack address: %p %p", &age, person);
NSLog(@"stack age is %d", age);
```

上面代码输出：

![__block变量的存储地址](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-151633.png)



可以看到，不管是`age`还是`person`，均在堆空间。

其实，本质上，将`Block`从栈拷贝到堆，也会将`__block`对象一并拷贝到堆，如下图：

![block从栈到堆细节](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-09-151848.png)

## 5.2. 对象与__block变量的区别

测试下面代码，[测试代码-__block与对象](https://github.com/wenghengcong/LearnCollection/tree/master/Block/__block与对象)

```
__block BFPerson *blockPerson = [[BFPerson alloc] init];
BFPerson *objectPerson = [[BFPerson alloc] init];
void(^block)(void) = ^ {
    NSLog(@"person is %@ %@", blockPerson, objectPerson);
};
```

转换后：

```
//Block对象
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
  	BFPerson *objectPerson;			//objectPerson对象，捕获
  	__Block_byref_blockPerson_0 *blockPerson; // blockPerson封装后的对象，内部捕获blockPerson
};

//__block blockPerson封装后的对象
struct __Block_byref_blockPerson_0 {
  	void *__isa;
	__Block_byref_blockPerson_0 *__forwarding;
 	void (*__Block_byref_id_object_copy)(void*, void*);
 	void (*__Block_byref_id_object_dispose)(void*);
 	BFPerson *blockPerson;
};

//两种对象不同的处理方式
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _Block_object_assign((void*)&dst->blockPerson, (void*)src->blockPerson, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_assign((void*)&dst->objectPerson, (void*)src->objectPerson, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

//__Block_byref_blockPerson_0内部对__block blockPerson的处理
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
	_Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
```

从上面可以得出：

![__block变量和对象的区别](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-10-024525.png)

