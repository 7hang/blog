# 四、Category

# 一、概述

## 1.1 category简介

category是Objective-C 2.0之后添加的语言特性，分类、类别其实都是指的category。category的主要作用是为已经存在的类添加方法。

可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处。

- 把不同的功能组织到不同的category里，减少单个文件的体积，且易于维护；
- 可以由多个开发者共同完成一个类；
- 可以按需加载想要的category；
- 声明私有方法；

不过除了apple推荐的使用场景，还衍生出了category的其他几个使用场景：

1. 模拟多继承（另外可以模拟多继承的还有protocol）
2. 把framework的私有方法公开

## 1.2 extension

 extension被开发者称之为扩展、延展、匿名分类。extension看起来很像一个匿名的category，但是extension和category几乎完全是两个东西。

 和category不同的是extension不但可以声明方法，还可以声明属性、成员变量。extension一般用于声明私有方法，私有属性，私有成员变量。

 使用 extension 必须有原有类的源码。extension 声明的方法、属性和成员变量必须在类的主 `@implementation` 区间内实现，可以避免使用有名称的 category 带来的多个不必要的 implementation 段。

 extension 很常见的用法，是用来给类添加私有的变量和方法，用于在类的内部使用。例如在 interface 中定义为 readonly 类型的属性，在实现中添加 extension，将其重新定义为 readwrite，这样我们在类的内部就可以直接修改它的值，然而外部依然不能调用 setter 方法来修改。

## 1.3 category与extension的区别

category和extension的区别来看，

- extension可以添加实例变量，而category是无法添加实例变量的。
- extension在编译期决议，是类的一部分，category则在运行期决议。extension在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，extension伴随类的产生而产生，亦随之一起消亡。
- extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension，除非创建子类再添加extension。而category不需要有类的源码，我们可以给系统提供的类添加category。
- extension和category都可以添加属性，但是category的属性不能生成成员变量和getter、setter方法的实现。

# 二、category编译期

首先，我们编写下面这些类，源码在：**[01-principle](https://github.com/wenghengcong/LearnCollection/tree/master/OCObject/category/01-principle)**

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-27-123306.png" style="zoom:50%;" />

重编译为C++代码：

```shell
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc BFPerson+Work.m
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc BFPerson+Study.m
```

以**Study**分类为例，看看C++中的结构体：

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-28-033532.png" style="zoom:50%;" />

我们看到在编译期中针对分类中的，实例方法、类方法以及属性（假如有协议同样会生成，示例代码中未继承协议）都生成了对应的结构体。

最后形成的结构体就是上图中的`_category_t`：

```c++
struct _category_t {
	const char *name;		//分类名
	struct _class_t *cls; 	//类
	const struct _method_list_t *instance_methods;	//实例方法列表	
	const struct _method_list_t *class_methods;		//类方法列表
	const struct _protocol_list_t *protocols;		//协议列表
	const struct _prop_list_t *properties;			//属性列表
};
```

# 三、category运行期

## 3.1 分类在运行期做了什么

- runtime会加载某个类的所有category数据；
- 将所有category的方法（对象方法、类方法）、属性、协议数据，合并到一个大数组中
  - 后面参与编译的category数据，会在合并数组的前面；
- 将合并后的分类数据（方法、属性、协议），插入到类原来数据的前面；

## 3.2 category_t结构体

```c++
//from objc-runtime-new.h
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

## 3.3 源码导读

下面源码，我们以分类中的**对象方法**为例。**其类方法、协议或属性，和对象方法处理流程类似，遵循同样的规律。**

### 3.3.1 导读图

源码执行流程如下：

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-30-024455.png" style="zoom:50%;" />



下面是分类方法合并到原类中的流程图：

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-30-063206.png" style="zoom:50%;" />



### 3.3.2 核心流程

我们读取最核心重布局对象方法的环节：

#### a. 确定分类编译顺序

项目编译中，Compile Sources会决定分类编译顺序，下图中**Study**分类在前，**Work**分类在后。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-30-030038.png"  style="zoom:50%;" />

#### b. 获取分类列表`cats`

得到列表，Study在前，Work在后。

> cats = [
>
>  category_t (BFPerson+Study),
>
>  category_t (BFPerson+Work)
>
> ]

#### c. 将`cats`方法合并到`mlists`

将分类`cats` **倒序**抽取出每个分类的方法，放入**二维数组**`mlists`中，此时Work对象方法在前：

> mlists = [
> ​ [**method_t** work, **method_t** test],
> ​ [**method_t** study, **method_t** test],
> ]

#### d. `mlists`插入`rw`

将`mlists`方法插入`rw`对象方法列表`method_array_t`中

1. 将rw原有方法列表`[method_t test]`移动到`method_array_t`最后；
2. 将分类`mlists`方法拷贝到`method_array_t`头部。

#### e. 对象方法结果

最后，我们获得的`rw`中`method_array_t`为：

> method_array_t = [
>
>  [method_t study, method_t test], *<–>BFPerson+Work*
>
>  [method_t work, method_t test], *<–>BFPerson+Study*
>
>  [ method_t test], *<–>BFPerson*
>
> ]

### 3.3.3 核心源码截取

下面是抽取的部分源码：

#### 3.3.3.1 分类方法处理

```c++
// cls - BFPerson
// *cats = [category_t (BFPerson+Study), category_t (BFPerson+Work) ]
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    // 分配内存：针对未进行重布局的分类列表
    /*mlists 方法二维数组*/
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    int i = cats->count;
    
    while (i--) {
        // i-- 在category list最后面的，先添加到mlists
        // 取出分类列表中的分类
        // 最后参与编译的，先取出category_t BFPerson+Work
        // 编译顺序：在编译log中查看，在Compile Sources中更改
        auto& entry = cats->list[i];
        // 取出分类的对象方法列表
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            // 将对应分类的方法列表添加到mlists
            /*
             以 *cats = [category_t (BFPerson+Study), category_t (BFPerson+Work)]
             执行完while 循环后，mlists为
             [
                [method_t work, method_t test],
                [method_t study, method_t test],
             ]
             */
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    
    // 将重新布局的category list添加到 rw中
    // 将所有分类的对象方法，附加到类对象的方法列表中
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    // 刷新方法缓存列表
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
}
```

#### 3.3.3.2 方法列表处理

然后开始重新编排`rw`中的方法列表：

```c++
/**
 @param addedLists 分类方法列表————二维数组
     [
        [method_t work, method_t test],
        [method_t study, method_t test]
     ]
 @param addedCount 分类列表长度————即分类的数目
 */
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;

    if (hasArray()) {
        // many lists -> many lists
        uint32_t oldCount = array()->count;
        uint32_t newCount = oldCount + addedCount;
        //重新分配实例对象数组列表内存
        setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
        array()->count = newCount;

        // array()->lists 原来的方法列表
        // *memmove(void *__dst, const void *__src, size_t __len);
        // 将原方法列表array()->lists开始，移动oldCount长度，移动到array()->lists + addedCount（分类个数）
        // 此处，假如原单位占用1个，分类此处有两个（Study、Work）
        // 【1】【】【】-> 【】【】【1】
        memmove(array()->lists + addedCount, array()->lists, 
                oldCount * sizeof(array()->lists[0]));
        // *memcpy(void *__dst, const void *__src, size_t __n);
        // 将addedLists开始的addedCount长度，拷贝到array()->lists位置
        // 【1】【】【】-> 【】【】【1】-> 【2-Work】【3-Study】【1】
        memcpy(array()->lists, addedLists, 
               addedCount * sizeof(array()->lists[0]));
    }
}
```