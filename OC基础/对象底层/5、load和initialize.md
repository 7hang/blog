# 五、load和initialize

# 一、常用知识点

在讲述之前，我们先把该两个方法常用到的一些知识点先列出。

![image-20181211200612904](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-11-120613.png)

# 二、+load

## 2.1 源码导读

根据下列顺序，阅读[objc源码](https://opensource.apple.com/tarballs/objc4/)即可。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-01-055844.png" alt="image-20181201135843918" style="zoom:50%;" />

## 2.2 测试用例

### 2.2.1 类与父类含`+load`

- 先调用父类的`+load`方法
- 再调用类的`+load`方法

```objective-c
@implementation BFPerson
+ (void)load
{
    NSLog(@"+[BFPerson load]");
}
@end

@implementation BFBoy
+ (void)load
{
    NSLog(@"+[BFBoy load]");
}
@end
```

输出日志：

> **14:21:44.542004+0800 Category[4483:5050573] +[BFPerson load]**
>
> **14:21:44.542545+0800 Category[4483:5050573] +[BFBoy load]**

### 2.2.2 分类有`load`方法

- 先调用类的`+load`方法

- 再调用分类的

  ```c++
  +load
  ```

  方法

  - 分类的`+load`方法调用顺序与编译顺序一致

```objective-c
@implementation BFBoy
+ (void)load
{    NSLog(@"+[BFBoy load]");}
@end

@implementation BFBoy (Handsome)
+ (void)load
{  NSLog(@"+[BFBoy load]--Handsome Cat");}
@end


@implementation BFBoy (Tall)
+ (void)load
{	NSLog(@"+[BFBoy load]--Tall Cat"); }
@end
```

输出日志：

> **14:24:09.679907+0800 Category[4565:5054709] +[BFPerson load]**
>
> **14:24:09.680555+0800 Category[4565:5054709] +[BFBoy load]**
>
> **14:24:09.680632+0800 Category[4565:5054709] +[BFPerson load]–Wrok Cat**
>
> **14:24:09.680749+0800 Category[4565:5054709] +[BFBoy load]–Tall Cat**
>
> **14:24:09.680845+0800 Category[4565:5054709] +[BFBoy load]–Handsome Cat**

## 2.3 核心流程

### 2.3.1 抽取+load方法

- （1）抽取类的+load方法
  - a.先抽取父类的+load方法
  - b.再抽取子类的+load方法
- （2）抽取分类的+load方法

```objective-c
void prepare_load_methods(const headerType *mhdr)
{
    //1.获取非懒加载的类列表
    classref_t *classlist = _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        //1. 先将类及其父类的load ---> loadable_classes
        schedule_class_load(remapClass(classlist[i]));
    }

    //2.获取分类列表
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        // 2. 将分类的load ---> loadable_categories
        add_category_to_loadable_list(cat);
    }
}
```

### 2.3.2 调用+load方法

调用load方法，即将上一步抽取出来的方法列表`loadable_classes`、`loadable_categories`，逐一调用即可。

- （1）先调用类（包含父类）的

  ```
  +load
  ```

  方法

  - a.先调用父类`+load`方法
  - b.再调用子类`+load`方法

- （2）调用分类的`+load`方法

```objective-c
void call_load_methods(void)
{
    do {
        // 1. 会先迭代调用完所有的类的load方法
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        // 2. 依次调用所有分类的load方法
        more_categories = call_category_loads();
    } while (loadable_classes_used > 0  ||  more_categories);
}
```

## 2.4. 源码截取

### 2.4.1 抽取类+load 方法

```objective-c
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();  //获取类中load的IMP
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    /*
      1. loadable_class ：Class load方法的结构体
            1.1 class与load一一对应
            1.2 struct loadable_class {
                    Class cls;  // may be nil
                    IMP method;
                };
      2. loadable_classes 存放loadable_class列表
      3. loadable_classes_used 当前load存放在loadable_class序号
      4. loadable_classes 每次分配的内存为 loadable_classes_allocated*2 + 16;
          4.1 realloc void *realloc(void *ptr, size_t size)
              重新调整之前调用 malloc 或 calloc 所分配的 ptr 所指向的内存块的大小。
     */

    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

### 2.4.2 分类方法抽取

```objective-c
void add_category_to_loadable_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = _category_getLoadMethod(cat);

    // Don't bother if cat has no +load method
    if (!method) return;

    if (PrintLoading) {
        _objc_inform("LOAD: category '%s(%s)' scheduled for +load", 
                     _category_getClassName(cat), _category_getName(cat));
    }
    /*
        1. loadable_category ：存放Category load方法的结构体
            1.1 class与load一一对应
            1.2 struct loadable_category {
                    Category cat;  // may be nil
                    IMP method;
              };
        2. loadable_categories 存放loadable_category列表
        3. loadable_categories_used 当前load存放在loadable_category序号
        4. loadable_categories 分配的内存
            4.1 realloc void *realloc(void *ptr, size_t size)
                重新调整之前调用 malloc 或 calloc 所分配的 ptr 所指向的内存块的大小。
            4.2 每次分配为 loadable_categories_allocated*2 + 16;
                即上次分配后，一直能使用到（上次的2倍+16）次，才会再次分配

     */
    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }

    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
```

### 2.4.3 +load方法调用

```
(*load_method)(cls, SEL_load);
```

# 三、+initialize

## 3.1 源码导读

由于`+initialize`是在类第一次接收到消息是调用，我们调用**[BFPerson alloc];**时，就会调用。

我们根据汇编debug得到了下面流程：

```apl
> 1. objc_msgSend
> 2. objc_msgSend_uncached
> 3._class_lookupMethodAndLoadCache3
> 4. lookUpImpOrForward
> 5. _class_initialize
	5.1>> _setThisThreadIsInitializingClass(objc_class*)
		5.1.1>>> _fetchInitializingClassList
    5.2>> CALLING_SOME+initialize_METHOD:
		5.2.1>>> objc_msgSend(cls, SEL_initialize) 
		//这一步会打印 +[BFPerson initialize]--Study Cat
	5.3>> lockAndFinishInitializing
```

然后，我们去[objc4](https://opensource.apple.com/tarballs/objc4/)源码挖矿。

我们发现，`1`、`2`在源码中都是汇编，`3`是可读的C代码。就从`3`开始。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-30-234217.png" alt="image-20181201074217670" style="zoom:50%;" />

## 3.2 测试用例

### 3.2.1 子类重写**+initialize**

- 优先调用父类`initialize`

```objective-c
@implementation BFPerson
+ (void)initialize
{
    NSLog(@"+[BFPerson initialize]");
}
@end

@implementation BFBoy
+ (void)initialize
{
    NSLog(@"+[BFBoy initialize]");
}
@end
```

调用

```
[BFBoy alloc];
```

此时打印如下：

> **13:44:35.437131+0800 Category[3527:4997267]** **+[BFPerson initialize]**
>
> **13:44:35.437240+0800 Category[3527:4997267]** **+[BFBoy initialize]**

### 3.2.2 分类重写**+initialize**

- 会调用分类`+initialize`，不调用类本身的`+initialize`

```objective-c
@implementation BFPerson
+ (void)initialize
{
    NSLog(@"+[BFPerson initialize]");
}
@end

@implementation BFPerson (Work)
+ (void)initialize
{
    NSLog(@"+[BFBoy initialize]");
}
@end
```

调用：

```
[BFBoy alloc];
```

打印输出

> **13:45:35.437121+0800 Category[3527:4997267] +[BFPerson initialize]–Wrok Cat**

### 3.2.3 多次调用`initialize`

多个子类均未实现，但父类实现`+initialize`方法，会多次调用`+initialize`。

```objective-c
@implementation BFPerson
+ (void)initialize
{
    NSLog(@"+[BFPerson initialize]");
}
@end

@implementation BFBoy
@end

@implementation BFGirl
@end
```

调用：

```
[BFBoy alloc];
[BFGirl alloc];
```

打印输出

> **13:52:44.968194+0800 Category[3786:5010745] +[BFPerson initialize]**
>
> **13:52:44.968327+0800 Category[3786:5010745] +[BFPerson initialize]**
>
> **13:52:44.968427+0800 Category[3786:5010745] +[BFPerson initialize]**

## 3.3. 核心流程

类的初始化加载，会先调用父类`+initialize`方法，再调用类本身的`+initialize`。

```objective-c
//伪代码
BOOL personInitialized = NO;
BOOL boyInitialized = NO;

if (!boyInitialized) {
    //1. 父类调用
    if (!personInitialized) {
        objc_msgSend([BFPerson class], @selector(initialize));
        personInitialized = YES;
    }
    //2. 类本身调用
	objc_msgSend([BFBoy class], @selector(initialize));
	boyInitialized = YES;
}
```

## 3.4. 源码截取

### 3.4.1 寻找方法

```objective-c
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    ....
	//需要初始化，且类未进行初始化
    if (initialize  &&  !cls->isInitialized()) {
        //initialize
        _class_initialize (_class_getNonMetaClass(cls, inst));
    }
    ....
}
```

### 3.4.2 类的`+initialize`调用

```objective-c
void _class_initialize(Class cls)
{
    // 1. 先调用父类 initialization
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    ....
    //2. 给类发送initialize消息
    callInitialize(cls);
    ....
    //3. 完成类的初始化
    lockAndFinishInitializing(cls, supercls);
    ...
}
```