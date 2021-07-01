# Runtime（一）isa指针

# 一、基础

在开始探讨isa指针之前，我们要准备一些基础知识，包括位域、联合体以及内存分配的相关知识。

## 1.1 位域

```c
//定义位段
struct mybitfields  
{  
    unsigned short a : 4;  
    unsigned short b : 5;  
    unsigned short c : 7;  
} test;  
  
int main( void );  
{  
    test.a = 2;  
    test.b = 31;  
    test.c = 0;  
}  

//最后如下显示
00000001 11110010  
cccccccb bbbbaaaa
```

位域表示的是，在一个结构体中，用位来存储数据。

关于位域的内存分配，有几个值得注意的点：

- 位域的最小内存不能小于位域中最大的修饰符，上面实例都是`short`，即最小不能小于`short`的大小；
- 位域的最小内存不能小于所有位域字段加起来的大小，并且根据内存对齐，可能会分配多余。假设所有位域字段加起来为18位，那么就会分配32位，即4字节。
- 位域中如果有无名位域，那么无名位域会强制下一位域对齐到其下一type位域的边界。（[C/C++位域结构深入解析](https://jocent.me/2017/07/24/bit-field-detail.html)）

## 1.2 联合体

```c
#include <stdio.h>
union data{
    int n;
    char ch;
    short m;
};
int main(){
    union data a;
    printf("%d, %d\n", sizeof(a), sizeof(union data) );
    a.n = 0x40;
    printf("%X, %c, %hX\n", a.n, a.ch, a.m);
    a.ch = '9';
    printf("%X, %c, %hX\n", a.n, a.ch, a.m);
    a.m = 0x2059;
    printf("%X, %c, %hX\n", a.n, a.ch, a.m);
    a.n = 0x3E25AD54;
    printf("%X, %c, %hX\n", a.n, a.ch, a.m);  
    return 0;
}
```

运行结果：

```
4, 4
40, @, 40
39, 9, 39
2059, Y, 2059
3E25AD54, T, AD54
```

# 二、isa

`isa` 在arm64架构前，是一个指针，指向类对象或元类对象。

但是在arm64之后，是个联合体，里面包含了更丰富的实例对象的信息，存储着更多运行期使用的信息。

**需要提醒的是，本系列中讨论的isa，都只针对iOS，且只针对64位。**

## 2.1 `isa`

`isa`指针的探索：

![image-20181123102401283](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-113830.png)

并且，我们通过实践得到通过isa如何得到类地址：

![image-20181123102411439](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-23-022412-20210519104814811.png)现在，我们从[objc](https://opensource.apple.com/tarballs/objc4/)的源码里，需要重新认识`isa`是什么了！

![image-20181212194136472](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-114137.png)2.2 `isa`里其他字段

通过上面我们看到联合体还包含了其他有用的字段，简单罗列如下：

![image-20181212195446353](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-115447.png)

看这些字段，我们验证了如下代码，代码放在[这里](https://github.com/wenghengcong/LearnCollection/tree/master/Runtime/isa指针)：

```objective-c
NSLog(@"1----%p", [BFPerson class]);

//0：nonpointer, 是否指针优化过，优化过即1，表示存储更多信息
BFPerson *person1 = [[BFPerson alloc] init];
//此时person1->isa 0x000001a100d88e39

//has_assoc, 添加关联对象
objc_setAssociatedObject(person1, @"test", @"good", OBJC_ASSOCIATION_RETAIN);
//person1->isa	0x000001a100d88e3b

//shiftclass，isa地址
BFPerson *person2 = person1;
//person1->isa	0x000021a100d88e3b

// weakly_referenced，是否被弱引用
__weak BFPerson *person3 = person1;
//person1->isa	0x000025a100d88e3b
```

其中一次验证的结果：

![image-20181212200341187](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-120341.png)

# 三、一个晦涩的实例

现在我们对`isa`已经了解的七七八八了，可以来看一个有趣的例子，源自sunny的一道面试题。

我作了细微改动，代码如下，

```objective-c
@interface BFPerson : NSObject
@property (nonatomic, copy) NSString * name;
- (void)print;
@end

@implementation BFPerson
- (void)print
{
    NSLog(@"my name is %@", self.name);
}
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    id cls = [BFPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
@end
```

看到上面，有两个疑问？

1. 这代码能不能跑起来？
2. 跑起来，最终结果又是什么呢？

 答案是，可以跑起来，输出如下：

> **my name is <ViewController: 0x7fc1c6e15df0>**

 那么疑问就来了，这`name`怎么来的。

 要解释这个问题，我们还是要了解`OC/C`的内存布局以及函数调用栈帧：

 看第一个问题：能不能跑起来？

## 3.1 能不能运行？

先看看平时是如何调用方法的：

```objective-c
BFPerson *person = [[BFPerson alloc] init];
 [person print];
```

再看看，这个面试题中又是如何调用的：

```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    
    id cls = [BFPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

现在，我们将两中调用的方式作了如下的对比：

![image-20181213071551847](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-231552.png)

 从上图可以看到：

 两者在内存中的方式极其类似。我们再还原一下，一个实例对象是如何调用实例方法的过程：

1. 根据实例对象，找到其`isa`指针；
2. 根据`isa`找到类对象，类对象存储了实例方法，找到方法列表，找出实例方法；
3. 调用实例方法；

 不管是`[obj print]`还是`[person print]`，我们目的是找到类对象，此处，`person`是指向实例对象的指针，当然能找到。

 那么！

 `obj`能找到吗？

 可以的，`obj`指向了`cls`，而`cls`直接指向了类对象，那么通过`cls`，这个相当于`isa`指针的指针，`obj`是能找到类对象的。

 那么，既然能找到类对象，调用其实例方法也就顺理成章。

我们的目标就是：**找到`cls`后第一个指针变量。**

## 3.2 运行结果的探讨

下面是我们最终的输出函数，要输出实例对象的`name`：

```objective-c
- (void)print
{
    NSLog(@"my name is %@", self.name);
}
```

根据上面的探讨，我们知道`cls`和实例对象`person`中的`isa`非常类似，**在本质上，其实就是isa指针的变体**。

最终`person`是找到其`name`属性就是直接在`isa`指针后的第一个成员变量值，而且**必须是指针变量**，因为我们知道`name`是一个指针变量。

那么，我们只要找到`cls`后面第一个**指针变量**即可。

现在，重新回到函数：

```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    
    id cls = [BFPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

`viewDidLoad`是一个函数，根据函数栈帧，局部变量分布在栈空间。

现在，将以上代码转为C++代码。

```objective-c
//[super viewDidLoad];
(
    (__rw_objc_super){
        self,
        class_getSuperclass(objc_getClass("ViewController"))
    }, sel_registerName("viewDidLoad")
 );

id cls = (objc_msgSend)(objc_getClass("BFPerson"), sel_registerName("class"));
void *obj = &cls;
(objc_msgSend)(obj, sel_registerName("print"));
```

焦点放在第一行，在OC中，调用`super`方法，均会将`super`转换为一个结构体，这个在后面会讲到：

```objective-c
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class class;
};
```

其他几行，我们非常熟悉。

现在来看一下该函数内部的局部变量有哪些：

- `objc_super`临时变量
- `cls`
- `obj`

它们内存空间分布如下：

![image-20181213074055539](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-12-234055.png)

现在我们**找到了`cls`的第一个指针变量`self`**，这也就是结果，在开始给出的输出：

> **my name is <ViewController: 0x7fc1c6e15df0>**

我们还可以验证一下：

看是不是`cls`后的第一个变量就行：

```objective-c
NSString *test = @"good";
id cls = [BFPerson class];
void *obj = &cls;
[(__bridge id)obj print];
```

上面的就会直接输出`test`的值。

另外，我们再验证下，假如不是**指针变量**是不是可以。

```objective-c
NSString *test = @"good";
int age = 18;
id cls = [BFPerson class];
void *obj = &cls;
[(__bridge id)obj print];
```

最终结果如下：

![image-20181213093129593](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-013129.png)