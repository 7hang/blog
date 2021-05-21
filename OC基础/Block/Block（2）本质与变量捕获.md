# Block（二）本质与变量捕获

# 一、本质

如果用一句话概括，**Block是一个将函数及其执行上下文封装起来的对象**。

## 1.1 [clang](https://clang.llvm.org/docs/Block-ABI-Apple.html)说Block是个对象

- 是对象：其内部第一个成员为isa指针；
- 封装了函数调用：Block内代码块，封装了函数调用，调用Block，就是调用该封装的函数；
- 执行上下文：Block还有一个描述Desc，该描述对象包含了Block的信息以及捕获变量的内存相关函数，及Block所在的环境上下文；

基于此，绘制了下图：

![block本质](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-06-092414.png)



## 1.2 验证

有以下的Block：

```objective-c
@implementation BlockStruct
- (void)test
{
    void (^helloBlock)(void) = ^() {
        NSLog(@"Hello world");
    };
    helloBlock();
}
@end
```

重写`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc BlockStruct.m`，我们得到C++源码：

### 1.2.1 test函数

test函数将转换为为下面代码：

```c++
static void _I_BlockStruct_test(BlockStruct * self, SEL _cmd) {
    void (*helloBlock)(void) = ((void (*)())
     	&__BlockStruct__test_block_impl_0(
        	(void *)__BlockStruct__test_block_func_0,
        	&__BlockStruct__test_block_desc_0_DATA)
      	);
    ((void (*)(__block_impl *))((__block_impl *)helloBlock)->FuncPtr)((__block_impl *)helloBlock);
}
```

`test`函数转换为`_I_BlockStruct_test`，在该函数内部，block转换的结构体为：`__BlockStruct__test_block_impl_0`。

### 1.2.2 block结构

block结构体`__BlockStruct__test_block_impl_0`有两个成员变量：

- `__block_impl`：block结构体
- `__BlockStruct__test_block_desc_0`：block描述的结构体

```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __BlockStruct__test_block_impl_0 {
  struct __block_impl impl;
  struct __BlockStruct__test_block_desc_0* Desc;
    //构造函数
  __BlockStruct__test_block_impl_0(void *fp, struct __BlockStruct__test_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

根据构造函数，再回过来看，如何初始化`helloBlock`的：

```c++
//helloBlock初始化
void (*helloBlock)(void) = (  (void (*)())
            &__BlockStruct__test_block_impl_0(
                (void *)__BlockStruct__test_block_func_0,
                &__BlockStruct__test_block_desc_0_DATA
            )
        );
```

将`__BlockStruct__test_block_func_0`赋值给`impl`，`__BlockStruct__test_block_desc_0_DATA`赋值给`Desc`。

### 1.2.3 `__block_impl`结构

上面说的block内部代码，此处为 `NSLog(@"Hello world");`，封装成了`__block_impl`类型，传入block结构体并初始化。

下面就是block内部封装的函数实现，后文我们将其称为Block的**Func**：

```c++
static void __BlockStruct__test_block_func_0(struct __BlockStruct__test_block_impl_0 *__cself) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_BlockStruct_e24876_mi_0);
}
```

### 1.2.4 block Desc结构

Block Desc描述的block的信息，包括Block大小和保留字。

后文将其称为Block的**Desc**。

```objective-c
static struct __BlockStruct__test_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __BlockStruct__test_block_desc_0_DATA = { 0, sizeof(struct __BlockStruct__test_block_impl_0)};
```

### 1.2.5 调用

在分析了`- (void)test` 函数的结构，以及block对应的结构组成，我们再看下是如何调用block的：

```c++
// helloBlock(); 转换为
((void (*)(__block_impl *)) ((__block_impl *)helloBlock)->FuncPtr)((__block_impl *)helloBlock);
```

- helloBlock结构的第一个成员变量为`__block_impl`，所以helloBlock首地址，就是`__block_impl impl`的首地址，即可以直接转换为`__block_impl`类型
- **(void (\*)(__block_impl \*))** 是`__block_impl` 中Func的类型
- **((__block_impl \*)helloBlock)->FuncPtr()** 调用函数
- **((__block_impl \*)helloBlock)** 函数参数

# 二、变量截获

Block本质是一个对象，那么在Block中访问全局变量以及局部变量，这个对象又是怎么处理这些变量的呢。

我们将变量分为以下几种类型，在以下表格中的存储区域：

| 类型     | 局部变量                                                     | 全局变量                                | 成员变量                                     |
| -------- | ------------------------------------------------------------ | --------------------------------------- | -------------------------------------------- |
| 定义     | ☞ 先定义再初始化， ☞ 定义同时初始化                          | ☞ 先定义再初始化 ☞ 定义同时初始化       | 不能在定义的同时进行初始化                   |
| 访问     | 函数（方法）或者大括号内部访问                               | 文件内直接访问                          | 通过对象来访问                               |
| 存储     | **栈** **全局存储区(static)** **text(const)**                | **全局存储区(static)** **text (const)** | **堆**                                       |
| 内存管理 | ☞ 栈中数据系统管理，会自动释放。 ☞ 全局存储区和Text在程序运行中一直保留 | ☞ 全局存储区和Text在程序运行中一直保留  | ☞ ARC会自动管理对象的内存 ☞ MRC下手动管理    |
| 其他     |                                                              |                                         | 成员变量不能离开类，离开类之后就不是成员变量 |

在此，需要特别指出`static`修饰的静态变量：

- static修饰变量，都存放在全局存储区，根据是否初始化，分别存储在data段或bss中，在程序运行中一直保留；
  - static修饰局部变量，其作用域为**函数或方法内**；
  - static修饰全局变量，其作用域为**该文件内**。

另外，上面指出的区域如下：

- .bss：存放未初始化数据的全局变量，以及 static 修饰的变量；
- .data：存放已经初始化的的全局变量，以及 static 修饰的变量；
- .text：存放代码，以及 const 等常量，这种常量包括 const 修饰符所修饰的，以及常量字符串。

我们在本文讨论Block对象**捕获基本类型**的情形。**另外需要提醒的是**，在捕获成员变量，即访问对象中属性或直接访问对象中成员变量的情形下，Block对象会做另外的处理。我们在后面篇章[Block（四）对象类型的auto变量](http://wenghengcong.com/posts/2acb8878/)中讨论。

## 2.1 各种变量如何被捕获

我们下面见根据变量修饰符，来探查Block如何捕获不同修饰符的类型变量。

- auto：自动变量修饰符
- static：静态修饰符
- const：常量修饰符

在这三种修饰符，我们又细分为**全局变量和局部变量**。

我们针对变量的类型，重写成C++结构如下：

![block对象的变量捕获过程](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-07-102645.png)

并得到如下结论：

![block变量捕获规律](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-17-020635.png)

在Block对象中捕获变量的类型基于变量类型，注意在**局部变量中的异数：static变量**。

- `auto`变量捕获后，`Block`中变量的类型和变量原类型一致；

  - 如变量是`int`类型，那么`Block`中：

    ```c++
    struct __CaptureAutoBlock__test_block_impl_0 {
      struct __block_impl impl;
      struct __CaptureAutoBlock__test_block_desc_0* Desc;
      int weight; 	//捕获外部变量
    };
    ```

- `static`变量捕获后，`Block`对应的变量是对应变量的指针类型；

  - 如变量是`int`类型，那么`Block`中：

    ```c++
    struct __CaptureAutoBlock__test_block_impl_0 {
      struct __block_impl impl;
      struct __CaptureAutoBlock__test_block_desc_0* Desc;
      int *weight; 	//捕获外部变量
    };
    ```

## 2.2 `auto`

`auto`变量，其实就是我们平时默认在方法内部定义的变量，在此，我们定义了一个：

- 全局变量`height`；
- 局部变量`weight`；

```objective-c
int height = 170;
- (void)test
{
    auto int weight = 66;
    void (^personInfoBlock)(void) = ^() {
        NSLog(@"height is %d, weight is %d", height, weight);
    };
    void (^bmiBlock)(int, int) = ^(int height, int weight) {
        NSLog(@"height is %d, weight is %d", height, weight);
    };
    height = 180;
    weight = 60;
    personInfoBlock();
    bmiBlock(height, weight);
}
```

再看重新编译C++后**personInfoBlock**的结构体

```c++
int height = 170; 	//全局变量
struct __CaptureAutoBlock__test_block_impl_0 {
  struct __block_impl impl;
  struct __CaptureAutoBlock__test_block_desc_0* Desc;
  int weight; 	//捕获外部变量
  __CaptureAutoBlock__test_block_impl_0(void *fp, struct __CaptureAutoBlock__test_block_desc_0 *desc, int _weight, int flags=0) : weight(_weight) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

我们可以看出，auto变量被捕获到Block的结构体中。

看下**personInfoBlock**内部代码封装的函数，及Block Desc描述信息：

```objective-c
static void __CaptureAutoBlock__test_block_func_0(struct __CaptureAutoBlock__test_block_impl_0 *__cself) {
  	int weight = __cself->weight; // bound by copy
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_CaptureAutoBlock_2618d5_mi_1, height, weight);
    }

static struct __CaptureAutoBlock__test_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __CaptureAutoBlock__test_block_desc_0_DATA = { 0, sizeof(struct __CaptureAutoBlock__test_block_impl_0)};
```

可以看出：

- 描述Block的`Desc`结构体未发生变化

- `Block Func`结构也未发生变化

  - 其中`Func`内部使用到的变量`weight`，用了在初始化`Block`时传入的变量，即**值传递**

    > **auto** **int** weight = 66;

  - `height`变量为全局变量，**直接访问**；

作为对比，我们将再看看**bmiBlock**有参数的情况：

```objective-c
struct __CaptureAutoBlock__test_block_impl_1 {
  struct __block_impl impl;
  struct __CaptureAutoBlock__test_block_desc_1* Desc;
  __CaptureAutoBlock__test_block_impl_1(void *fp, struct __CaptureAutoBlock__test_block_desc_1 *desc, int flags=0) {
  	.....
  }
};
static void __CaptureAutoBlock__test_block_func_1(struct __CaptureAutoBlock__test_block_impl_1 *__cself, int height, int weight) {

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_CaptureAutoBlock_68f446_mi_2, height, weight);
    }
```

可以看出，**bmiBlock**这种带有参数的情况下，Block并不会捕获变量，而是在使用时候，即时将新参数传入完成调用，Block最终转换的结构体和**无参数且无变量的Block**是非常相似。

## 2.3 `static`

下面我们定义了三个变量：

- 全局
  - 变量：vision
- 局部
  - 常量：height
  - 变量：weight

```
static int vision = 5;
- (void)test
{
    static const int height = 170;
    static int weight = 60;
    void (^personInfoBlock)(void) = ^() {
        weight = 70;
        vision = 4;
        NSLog(@"vision is %d, height is %d, weight is %d", vision, height, weight);
    };
    weight = 80;
    vision = 3;
    personInfoBlock();
}
```

经过测试：

1. 上述代码，结果输出为：**vision is 4, height is 170, weight is 70**
2. 注释第7、8行代码，输出：**vision is 3, height is 170, weight is 80**
3. 接着，注释11、12行，很明显输出：**vision is 5, height is 170, weight is 60**

从以上测试结果我们可以得出：

- Block对象能获取最新的静态全局变量和静态局部变量；
- 静态局部常量由于值不会更改，故没有变化；

我们来看一下，发生了什么。

```c
static int vision = 5;		//全局静态变量
struct __CaptureStaticBlock__test_block_impl_0 {
  struct __block_impl impl;
  struct __CaptureStaticBlock__test_block_desc_0* Desc;
  int *weight;			//捕获变量，获取变量地址
  const int *height;	//捕获变量，获取变量地址
   ...
};
static void __CaptureStaticBlock__test_block_func_0(struct __CaptureStaticBlock__test_block_impl_0 *__cself) {
        //2.通过Block对象获取到weight和height的指针
  int *weight = __cself->weight; // bound by copy
  const int *height = __cself->height; // bound by copy
    	//3.通过weight指针，更改weight指向的值
        (*weight) = 70;
        vision = 4;
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_CaptureStaticBlock_4f7cf7_mi_1, vision, (*height), (*weight));
    }

static void _I_CaptureStaticBlock_test(CaptureStaticBlock * self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_CaptureStaticBlock_4f7cf7_mi_0);
    static const int height = 170;
    static int weight = 60;
    //1.传入&weight, &height地址进行Blcok对象的初始化
    void (*personInfoBlock)(void) = ((&__CaptureStaticBlock__test_block_impl_0((void *)__CaptureStaticBlock__test_block_func_0, &__CaptureStaticBlock__test_block_desc_0_DATA, &weight, &height));

    weight = 80;
    vision = 3;
    (personInfoBlock)->FuncPtr)(personInfoBlock);
}
```

为什么能获取`static`变量最新的值？从上面的转换中得出：

1. `static`修饰的，其作用区域不管是全局还是局部，不管是常量还是变量，**均存储在全局存储区中**，存在全局存储区，该地址在程序运行过程中一直不会改变，所以能访问最新值。
2. `static`修饰后：

- 全局变量，**直接访问**。

- 局部变量，

  指针访问

  。其中在局部变量中，又有局部静态常量，即被const修饰的。

  - `const`存放在text段中，即使被`static`同时修饰，也存放text中的常量区；

## 2.4 `const`

如下定义：

- `const`全局变量：vision
- `const`局部变量：height

```objective-c
const int vision = 5;
- (void)test
{
    NSLog(@"captrue const variable in block");
    const int height = 170;
    void (^personInfoBlock)(void) = ^() {
        NSLog(@"height is %d, vision is %d", height, vision);
    };
    
    personInfoBlock();
}
```

转换后：

```c++
const int vision = 5;

struct __CaptureConstBlcok__test_block_impl_0 {
  struct __block_impl impl;
  struct __CaptureConstBlcok__test_block_desc_0* Desc;
  const int height;
  ....
};
static void __CaptureConstBlcok__test_block_func_0(struct __CaptureConstBlcok__test_block_impl_0 *__cself) {
  const int height = __cself->height; // bound by copy
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_vx_b9xvt9pn7rnfbdlljj6tyqc40000gn_T_CaptureConstBlcok_b33b47_mi_1, height, vision);
    }
```

从上面看出：

- `const`全局变量**直接访问**；
- `const`局部变量，其实仍然是`auto`修饰，**值传递**；