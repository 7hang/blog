## 前言

在文章之前，先抛出如下问题。

- block的原理是怎样的？本质是什么？
- `__block`的作用是什么？有什么使用注意点？
- block的属性修饰词为什么是copy？使用block有哪些使用注意？
- block一旦没有进行copy操作，就不会在堆上
- block在修改NSMutableArray，需不需要添加__block？

如果现在不是很熟悉，希望看完这篇文章，能有个新的认识。

### 导读

本文主要从如下几个方面讲解block

- block的基本使用
- block在内存中的布局
- block对变量的捕获分析
- MRC和ARC的对比
- `__block`的分析
- block中内存管理问题
- block导致的循环引用问题

### 什么是block

先介绍一下什么是闭包。在 wikipedia 上，[闭包](https://en.wikipedia.org/wiki/Closure_(computer_science):)的定义是

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

翻译过来表达就是

> 闭包是一个函数（或指向函数的指针），再加上该函数执行的外部的上下文变量(有时候也称作自由变量）。

- block 实际上就是 Objective-C 语言对于闭包的实现。

## block的基本使用

- block本质上也是一个OC对象，它内部也有个isa指针
- block是封装了函数调用以及函数调用环境的OC对象
- block的底层结构如下图



![block的底层结构](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09240311d9088.png)



### 无参无返回值的定义和使用

```
//无参无返回值 定义 和使用
void (^MyBlockOne)(void) = ^{
      NSLog(@"无参无返回值");
};
    
// 调用
MyBlockOne();

复制代码
```

### 无参有返回值的定义和使用

```
// 无参有返回值
int (^MyBlockTwo)(void) = ^{
    NSLog(@"无参有返回值");
    return 2;
};
// 调用
int res = MyBlockTwo();
复制代码
```

### 有参无返回值的定义和使用

```
//有参无返回值 定义
void (^MyBlockThree)(int a) = ^(int a){
    NSLog(@"有参无返回值 a = %d",a);
};
    
// 调用
MyBlockThree(10);
复制代码
```

### 有参有返回值的定义和使用

```
//有参有返回值
int (^MyBlockFour)(int a) = ^(int a){
    NSLog(@"有参有返回值 a = %d",a);
    return a * 2;
};
MyBlockFour(4);
复制代码
```

### typedef 定义Block

实际开发中，经常需要把block作为一个属性，我们可以定义一个block

eg:定义一个有参有返回值的block

```
typedef int (^MyBlock)(int a, int b);
复制代码
```

定义属性的时候,如下即可持有这个block

```
@property (nonatomic,copy) MyBlock myBlockOne;
复制代码
```

block实现

```
self.myBlockOne = ^int(int a, int b) {
      return a + b;
};
复制代码
```

调用

```
self.myBlockOne(2, 5);
复制代码
```

## block 类型和数据结构

### block 数据结构分析

#### 生成cpp文件

如下代码

```
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
 };
        
block();

复制代码
```

- 打开终端，cd到当前目录下

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`

#### block 结构分析

```
int age = 20;

// block的定义
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
// block的调用
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

复制代码
```

上面的代码删除掉一些强制转换的代码就就剩下如下所示

```
int age = 20;
void (*block)(void) = &__main_block_impl_0(
						__main_block_func_0, 
						&__main_block_desc_0_DATA, 
						age
						);
// block的调用
block->FuncPtr(block);
复制代码
```

看出block的本质就是一个结构体对象，结构体`__main_block_impl_0`代码如下

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
    //构造函数(类似于OC中的init方法) _age是外面传入的
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    //isa指向_NSConcreteStackBlock 说明这个block就是_NSConcreteStackBlock类型的
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
复制代码
```

结构体中第一个是`struct __block_impl impl;`

```
struct __block_impl {
      void *isa;
      int Flags;
      int Reserved;
      void *FuncPtr;
};       
复制代码
```

结构体中第二个是`__main_block_desc_0;`

```
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size; // 结构体__main_block_impl_0 占用的内存大小
}
复制代码
```

结构体中第三个是`age`

也就是捕获的局部变量 `age`

```
__main_block_func_0
//封装了block执行逻辑的函数
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int age = __cself->age; // bound by copy
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_7f3f1b_mi_0,age);
}
复制代码
```

用一幅图来表示

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09240319606d9.png)



### 变量捕获

其实上面的代码我们已经看得出来变量捕获了，这里继续详细分析一下

| 变量类型        | 捕获到block内部 | 访问方式 |
| --------------- | --------------- | -------- |
| 局部变量 auto   | √               | 值传递   |
| 局部变量 static | √               | 指针传递 |
| 全局变量        | ×               | 直接访问 |

#### 局部变量auto(自动变量)

- 我们平时写的局部变量，默认就有 auto(自动变量，离开作用域就销毁)

##### 运行代码

例如下面的代码

```
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
};
age = 25;
       
block();
复制代码
```

等同于

```
auto int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
};
age = 25;
       
block();
复制代码
```

输出

> 20

##### 分析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`

如图所示

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c0924031be5084.png)



```
int age = 20;
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
age = 25;

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

struct __main_block_impl_0 *blockStruct = (__bridge struct __main_block_impl_0 *)block;

NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_d36452_mi_5);
复制代码
```

可以知道，直接把age的值 20传到了结构体`__main_block_impl_0`中，后面再修改`age = 25`并不能改变block里面的值

#### 局部变量 static

static修饰的局部变量，不会被销毁

##### 运行代码

eg

```
static int height  = 30;
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d height = %d",age,height);
};
age = 25;
height = 35;
block();
        
复制代码
```

执行结果为

```
age is 20 height = 35

复制代码
```

可以看得出来，block外部修改height的值，依然能影响block内部的值

##### 分析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c0924031b78e23.png)



```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age; // bound by copy
  int *height = __cself->height; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_3146e1_mi_4,age,(*height));
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 



        static int height = 30;
        int age = 20;
        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age, &height));
        age = 25;
        height = 35;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
复制代码
```

如图所示，`age`是直接值传递，`height`传递的是`*height` 也就是说直接把内存地址传进去进行修改了。

#### 全局变量

##### 运行代码

```
int age1 = 11;
static int height1 = 22;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) =  ^{
            NSLog(@"age1 is %d height1 = %d",age1,height1);
        };
        age1 = 25;
        height1 = 35;
        block();

    }
    return 0;
}
复制代码
```

输出结果为

```
age1 is 25 height1 = 35


复制代码
```

##### 分析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c0924031a1a372.png)



```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_4e8c40_mi_4,age1,height1);
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        age1 = 25;
        height1 = 35;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
        
    }
    return 0;
}
复制代码
```

从cpp文件可以看出来，并没有捕获全局变量age1和height1,访问的时候，是直接去访问的，根本不需要捕获

#### 小结

| 变量类型        | 捕获到block内部 | 访问方式 |
| --------------- | --------------- | -------- |
| 局部变量 auto   | √               | 值传递   |
| 局部变量 static | √               | 指针传递 |
| 全局变量        | ×               | 直接访问 |

- auto修饰的局部变量，是值传递
- static修饰的局部变量，是指针传递

其实也很好理解，因为auto修饰的局部变量，离开作用域就销毁了。那如果是指针传递的话，可能导致访问的时候，该变量已经销毁了。程序就会出问题。而全局变量本来就是在哪里都可以访问的，所以无需捕获。

### block类型

#### block也是一个OC对象

在进行分析block类型之前，先明确一个概念，那就是block中有isa指针的，block是一个OC对象，例如下面的代码

```
void (^block)(void) =  ^{
      NSLog(@"123");
};

NSLog(@"block.class = %@",[block class]);
NSLog(@"block.class.superclass = %@",[[block class] superclass]);
NSLog(@"block.class.superclass.superclass = %@",[[[block class] superclass] superclass]);
NSLog(@"block.class.superclass.superclass.superclass = %@",[[[[block class] superclass] superclass] superclass]);
复制代码
```

输出结果为

```
iOS-block[18429:234959] block.class = __NSGlobalBlock__
iOS-block[18429:234959] block.class.superclass = __NSGlobalBlock
iOS-block[18429:234959] block.class.superclass.superclass = NSBlock
iOS-block[18429:234959] block.class.superclass.superclass.superclass = NSObject

复制代码
```

说明了上面代码中的block的类型是`__NSGlobalBlock`，继承关系可以表示为`__NSGlobalBlock__ : __NSGlobalBlock : NSBlock : NSObject`

#### block有3种类型

block有3种类型，可以通过调用class方法或者isa指针查看具体类型，最终都是继承自NSBlock类型

- `__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）`
- `__NSStackBlock__ （ _NSConcreteStackBlock ）`
- `__NSMallocBlock__ （ _NSConcreteMallocBlock ）`

其中三种不同的类型和环境对应如下

| block类型           | 环境                         |
| ------------------- | ---------------------------- |
| `__NSGlobalBlock__` | 没有访问auto变量             |
| `__NSStackBlock__`  | 访问了auto变量               |
| `__NSMallocBlock__` | `__NSStackBlock__`调用了copy |

其在内存中的分配如下对应



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c0924036237a71.png)



#### 运行代码查看

##### MRC下

**注意，以下代码在MRC下测试**

**注意，以下代码在MRC下测试**

**注意，以下代码在MRC下测试**

因为ARC的时候，编译器做了很多的优化，往往看不到本质,

- 改为MRC方法： `Build Settings` 里面的`Automatic Reference Counting`改为NO

如下图所示



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09240e346ed71.png)



用代码来表示

```
void (^block)(void) =  ^{
       NSLog(@"123");
};

NSLog(@"没有访问auto block.class = %@",[block class]);
        
        
auto int a = 10;
void (^block1)(void) =  ^{
      NSLog(@"a = %d",a);
};
        
NSLog(@"访问了auto block1.class = %@",[block1 class]);
               
NSLog(@"访问量auto 并且copy block1-copy.class = %@",[[block1 class] copy]);
复制代码
```

输出为

```
OS-block[23542:349513] 没有访问auto block.class = __NSGlobalBlock__
iOS-block[23542:349513] 访问了auto block1.class = __NSStackBlock__
iOS-block[23542:349513] 访问量auto 并且copy block1-copy.class = __NSStackBlock__
复制代码
```

可以看出和上面说的

| block类型           | 环境                         |
| ------------------- | ---------------------------- |
| `__NSGlobalBlock__` | 没有访问auto变量             |
| `__NSStackBlock__`  | 访问了auto变量               |
| `__NSMallocBlock__` | `__NSStackBlock__`调用了copy |

是一致的

##### ARC下

在ARC下，上面的代码输出结果为下面所示，因为编译器做了copy

```
iOS-block[24197:358752] 没有访问auto block.class = __NSGlobalBlock__
iOS-block[24197:358752] 访问了auto block1.class = __NSMallocBlock__
iOS-block[24197:358752] 访问量auto 并且copy block1-copy.class = __NSMallocBlock__
复制代码
```

### block的copy

前面说了在ARC环境下，编译器会根据情况自动将栈上的block复制到堆上，具体来说比如以下情况

#### copy的情况

- block作为函数返回值时
- 将block赋值给__strong指针时
- block作为Cocoa API中方法名含有usingBlock的方法参数时
- block作为GCD API的方法参数时

##### block作为函数返回值时



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092417f50345c.png)



```
// 定义Block
typedef void (^YZBlock)(void);

// 返回值为Block的函数
YZBlock myblock()
{
    int a = 6;
    return ^{
        NSLog(@"--------- %d",a);
    };
}

YZBlock Block = myblock();
Block();
NSLog(@" [Block class] = %@", [Block class]);
复制代码
```

输出为

```
iOS-block[25857:385868] --------- 6
iOS-block[25857:385868]  [Block class] = __NSMallocBlock__
复制代码
```

上述代码如果再MRC下输出`__NSStackBlock__`，在ARC下，自动copy，所以是`__NSMallocBlock__`

##### 将block赋值给`__strong`指针时

```
// 定义Block
typedef void (^YZBlock)(void);

int b = 20;
YZBlock Block2 = ^{
    NSLog(@"abc %d",b);
};
NSLog(@" [Block2 class] = %@", [Block2 class]);

复制代码
```

输出为

```
iOS-block[26072:389164]  [Block2 class] = __NSMallocBlock__
复制代码
```

上述代码如果再MRC下输出`__NSStackBlock__`，在ARC下，自动copy，所以是`__NSMallocBlock__`

##### block作为Cocoa API中方法名含有usingBlock的方法参数时

eg:

```
NSArray *array = @[@1,@4,@5];
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            // code
}];
复制代码
```

##### block作为GCD API的方法参数时

eg

```
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
            
});    
       
        
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        //code to be executed after a specified delay
});
复制代码
```

#### MRC下block属性的建议写法

- @property (copy, nonatomic) void (^block)(void);

#### ARC下block属性的建议写法

- @property (strong, nonatomic) void (^block)(void);
- @property (copy, nonatomic) void (^block)(void);

## 对象类型的auto变量

### 例子一

首先看一个简单的例子 定义一个类 `YZPerson`,里面只有一个`dealloc`方法

```
@interface YZPerson : NSObject
@property (nonatomic ,assign) int age;
@end


@implementation YZPerson

- (void)dealloc
{
    NSLog(@"%s",__func__);
}

@end
复制代码
```

如下代码使用

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        {
            YZPerson *person = [[YZPerson alloc]init];
            person.age = 10;
        }
        NSLog(@"-----");
    }
    return 0;
}
复制代码
```

想必大家都能知道会输出什么,没错，就是person先销毁，然后打印`-----` 因为person是在大括号内，当大括号执行完之后，person 就销毁了。

```
iOS-block[1376:15527] -[YZPerson dealloc]
iOS-block[1376:15527] -----
复制代码
```

### 例子二

上面的例子，是不是挺简单，那下面这个呢，

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        YZBlock block;
        
        {
            YZPerson *person = [[YZPerson alloc]init];
            person.age = 10;
            
            block = ^{
                NSLog(@"---------%d", person.age);
            };
            
             NSLog(@"block.class = %@",[block class]);
        }
        NSLog(@"block销毁");

    }
    return 0;
}


复制代码
```

如下结果，输出可知当 block为`__NSMallocBlock__`类型时候，block可以保住person的命的，因为person离开大括号之后没有销毁，当block销毁，person才销毁

```
iOS-block[3186:35811] block.class = __NSMallocBlock__
iOS-block[3186:35811] block销毁
iOS-block[3186:35811] -[YZPerson dealloc]
复制代码
```

#### 分析

终端执行这行指令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`把`main.m`生成`main.cpp` 可以 看到如下代码

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  YZPerson *person;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, YZPerson *_person, int flags=0) : person(_person) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
复制代码
```

很明显就是这个block里面包含 `YZPerson *person`。

### MRC下 block引用实例对象

上面的例子，是不是挺简单，那如果是MRC下呢

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        YZBlock block;
        
        {
            YZPerson *person = [[YZPerson alloc]init];
            person.age = 10;
            
            block = ^{
                NSLog(@"---------%d", person.age);
            };
            
            NSLog(@"block.class = %@",[block class]);

            // MRC下，需要手动释放
            [person release];
        }
        NSLog(@"block销毁");
        // MRC下，需要手动释放
		 [block release];
    }
    return 0;
}
复制代码
```

输出结果为

```
iOS-block[3114:34894] block.class = __NSStackBlock__
iOS-block[3114:34894] -[YZPerson dealloc]
iOS-block[3114:34894] block销毁
复制代码
```

和上面的对比，区别就是，还没有执行`NSLog(@"block销毁");`的时候，`[YZPerson dealloc]`已经执行了。也就是说，person 离开大括号，就销毁了。

输出可知当 block为`__NSStackBlock__`类型时候，block不可以保住person的命的

### MRC下 [block copy]引用实例对象

在MRC下，对block执行了copy操作

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        YZBlock block;
        
        {
            YZPerson *person = [[YZPerson alloc]init];
            person.age = 10;
            
            block = [^{
                NSLog(@"---------%d", person.age);
            } copy];
            
			NSLog(@"block.class = %@",[block class]);
            // MRC下，需要手动释放
            [person release];
        }
       
        NSLog(@"block销毁");
        [block release];
    }
    return 0;

复制代码
```

输出结果为,可知当 block为`__NSMallocBlock__`类型时候，block是可以保住person的命的

```
iOS-block[3056:34126] block.class = __NSMallocBlock__
iOS-block[3056:34126] block销毁
iOS-block[3056:34126] -[YZPerson dealloc]
复制代码
```

### `__weak`修饰

- 如下代码

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        YZBlock block;
        
        {
            YZPerson *person = [[YZPerson alloc]init];
            person.age = 10;

            __weak YZPerson *weakPerson = person;
            
            block = ^{
                NSLog(@"---------%d", weakPerson.age);
            };
            
             NSLog(@"block.class = %@",[block class]);
        }
       
        NSLog(@"block销毁");
    }
    return 0;
}
复制代码
```

- 输出为

```
iOS-block[3687:42147] block.class = __NSMallocBlock__
iOS-block[3687:42147] -[YZPerson dealloc]
iOS-block[3687:42147] block销毁
复制代码
```

- 生成cpp文件
- 

注意：

- 在使用clang转换OC为C++代码时，可能会遇到以下问题 `cannot create __weak reference in file using manual reference`
- 解决方案：支持ARC、指定运行时系统版本，比如 `xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m`

生成之后，可以看到，如下代码，MRC情况下，生成的代码明显多了,这是因为ARC自动进行了copy操作

```
 //copy 函数
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  
  //dispose函数
  void (*dispose)(struct __main_block_impl_0*);
复制代码
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  //weak修饰
  YZPerson *__weak weakPerson;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, YZPerson *__weak _weakPerson, int flags=0) : weakPerson(_weakPerson) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};


static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  //copy 函数
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  
  //dispose函数
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {
 0, 
 sizeof(struct __main_block_impl_0),
  __main_block_copy_0,
   __main_block_dispose_0
};


//copy函数内部会调用_Block_object_assign函数
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {

//asssgin会对对象进行强引用或者弱引用
_Block_object_assign((void*)&dst->person, 
(void*)src->person, 
3/*BLOCK_FIELD_IS_OBJECT*/);
}

//dispose函数内部会调用_Block_object_dispose函数
static void __main_block_dispose_0(struct __main_block_impl_0*src) {
_Block_object_dispose((void*)src->person, 
3/*BLOCK_FIELD_IS_OBJECT*/);
}

复制代码
```

### 小结

无论是MAC还是ARC

- 当block为`__NSStackBlock__`类型时候，是在栈空间，无论对外面使用的是strong 还是weak 都不会对外面的对象进行强引用
- 当block为`__NSMallocBlock__`类型时候，是在堆空间，block是内部的`_Block_object_assign`函数会根据`strong`或者 `weak`对外界的对象进行强引用或者弱引用。

其实也很好理解，因为block本身就在栈上，自己都随时可能消失，怎么能保住别人的命呢？

- 当block内部访问了对象类型的auto变量时
- 如果block是在栈上，将不会对auto变量产生强引用
- 如果block被拷贝到堆上
  - 会调用block内部的copy函数
  - copy函数内部会调用`_Block_object_assign`函数
  - `_Block_object_assign`函数会根据auto变量的修饰符`（__strong、__weak、__unsafe_unretained）`做出相应的操作，形成强引用（retain）或者弱引用
- 如果block从堆上移除
  - 会调用block内部的dispose函数
  - dispose函数内部会调用`_Block_object_dispose`函数
  - `_Block_object_dispose`函数会自动释放引用的auto变量（release）

| 函数        | 调用时机              |
| ----------- | --------------------- |
| copy函数    | 栈上的Block复制到堆上 |
| dispose函数 | 堆上的block被废弃时   |

## `__block`

先从一个简单的例子说起，请看下面的代码

```
// 定义block
typedef void (^YZBlock)(void);

int age = 10;
YZBlock block = ^{
    NSLog(@"age = %d", age);
};
block();
复制代码
```

代码很简单，运行之后，输出

> age = 10

上面的例子在block中访问外部局部变量，那么问题来了，如果想在block内修改外部局部的值，怎么做呢？

### 修改局部变量的三种方法

#### 写成全局变量

我们把a定义为全局变量，那么在哪里都可以访问，

```
// 定义block
typedef void (^YZBlock)(void);
 int age = 10;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        YZBlock block = ^{
            age = 20;
            NSLog(@"block内部修改之后age = %d", age);
        };
        
        block();
        NSLog(@"block调用完 age = %d", age);
    }
    return 0;
}
复制代码
```

这个很简单，输出结果为

```
block内部修改之后age = 20
block调用完 age = 20
复制代码
```

对于输出就结果也没什么问题，因为全局变量，是所有地方都可访问的，在block内部可以直接操作age的内存地址的。调用完block之后，全局变量age指向的地址的值已经被更改为20，所以是上面的打印结果

#### static修改局部变量

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
       static int age = 10;
        YZBlock block = ^{
            age = 20;
            NSLog(@"block内部修改之后age = %d", age);
        };
        
        block();
        NSLog(@"block调用完 age = %d", age);
    }
    return 0;
}
复制代码
```

上面的代码输出结果为

```
block内部修改之后age = 20
block调用完 age = 20
复制代码
```

终端执行这行指令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`把`main.m`生成`main.cpp` 可以 看到如下代码

```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int *age;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_age, int flags=0) : age(_age) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *age = __cself->age; // bound by copy
    
    (*age) = 20;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_5dbaa1_mi_0, (*age));
}

复制代码
```

可以看出，当局部变量用static修饰之后，这个block内部会有个成员是`int *age`，也就是说把age的地址捕获了。这样的话，当然在block内部可以修改局部变量age了。

- 以上两种方法，虽然可以达到在block内部修改局部变量的目的，但是，这样做，会导致内存无法释放。无论是全局变量，还是用static修饰，都无法及时销毁，会一直存在内存中。很多时候，我们只是需要临时用一下，当不用的时候，能销毁掉，那么第三种，也就是今天的主角 `__block`隆重登场

#### `__block`来修饰

代码如下

```
// 定义block
typedef void (^YZBlock)(void);


int main(int argc, const char * argv[]) {
    @autoreleasepool {
       __block int age = 10;
        YZBlock block = ^{
            age = 20;
            NSLog(@"block内部修改之后age = %d",age);
        };
        
        block();
        NSLog(@"block调用完 age = %d",age);
    }
    return 0;
}
复制代码
```

输出结果和上面两种一样

```
block内部修改之后age = 20
block调用完 age = 20
复制代码
```

### `__block`分析

- 终端执行这行指令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`把`main.m`生成`main.cpp`

首先能发现 多了`__Block_byref_age_0`结构体

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
    // 这里多了__Block_byref_age_0类型的结构体
  __Block_byref_age_0 *age; // by ref
    // fp是函数地址  desc是描述信息  __Block_byref_age_0 类型的结构体  *_age  flags标记
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp; //fp是函数地址
    Desc = desc;
  }
};


复制代码
```

再仔细看结构体`__Block_byref_age_0`，可以发现第一个成员变量是isa指针，第二个是指向自身的指针`__forwarding`

```
// 结构体 __Block_byref_age_0
struct __Block_byref_age_0 {
    void *__isa; //isa指针
    __Block_byref_age_0 *__forwarding; // 指向自身的指针
    int __flags;
    int __size;
    int age; //使用值
};
复制代码
```

查看main函数里面的代码

```
  // 这是原始的代码 __Block_byref_age_0
 __attribute__((__blocks__(byref))) __Block_byref_age_0 age = {
 (void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};
        
             
// 这是原始的 block代码
YZBlock block = ((void (*)())&__main_block_impl_0(
(void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));

复制代码
```

代码太长，简化一下，去掉一些强转的代码，结果如下

```
// 这是原始的代码 __Block_byref_age_0
__attribute__((__blocks__(byref))) __Block_byref_age_0 age = {(void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};
        
//这是简化之后的代码 __Block_byref_age_0
__Block_byref_age_0 age = {
     0, //赋值给 __isa
     (__Block_byref_age_0 *)&age,//赋值给 __forwarding,也就是自身的指针
      0, // 赋值给__flags
      sizeof(__Block_byref_age_0),//赋值给 __size
      10 // age 使用值
    };
        
// 这是原始的 block代码
YZBlock block = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));
        
// 这是简化之后的 block代码
YZBlock block = (&__main_block_impl_0(
             		__main_block_func_0,
           		&__main_block_desc_0_DATA,
	           	 &age,
            	570425344));
        
 ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
        //简化为
block->FuncPtr(block);

复制代码
```

其中`__Block_byref_age_0`结构体中的第二个`(__Block_byref_age_0 *)&age`赋值给上面代码结构体`__Block_byref_age_0`中的第二个`__Block_byref_age_0 *__forwarding`,所以`__forwarding` 里面存放的是指向自身的指针

```
//这是简化之后的代码 __Block_byref_age_0
__Block_byref_age_0 age = {
     0, //赋值给 __isa
     (__Block_byref_age_0 *)&age,//赋值给 __forwarding,也就是自身的指针
      0, // 赋值给__flags
      sizeof(__Block_byref_age_0),//赋值给 __size
      10 // age 使用值
    };
复制代码
```

结构体`__Block_byref_age_0`中代码如下，第二个`__forwarding`存放指向自身的指针,第五个`age`里面存放局部变量

```
// 结构体 __Block_byref_age_0
struct __Block_byref_age_0 {
    void *__isa; //isa指针
    __Block_byref_age_0 *__forwarding; // 指向自身的指针
    int __flags;
    int __size;
    int age; //使用值
};
复制代码
```

调用的时候，先通过`__forwarding`找到指针，然后去取出age值。

```
(age->__forwarding->age));
复制代码
```



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092427cff07f0.png)





![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09242afff1b4d.png)



### 小结

- `__block`可以用于解决block内部无法修改auto变量值的问题
- `__block`不能修饰全局变量、静态变量（static）
  - 编译器会将`__block`变量包装成一个对象

调用的是，从`__Block_byref_age_0`的指针找到 `age`所在的内存，然后修改值



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09243232b2d65.png)



## 内存管理问题

### bloc访问OC对象

#### 代码如下

当block内部访问外面的OC对象的时候

eg:

```
// 定义block
typedef void (^YZBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
     
        NSObject *obj = [[NSObject alloc]init];
        YZBlock block = ^{
            NSLog(@"%p",obj);
        };
         block();
    }
    return 0;
}

复制代码
```

在终端使用clang转换OC为C++代码

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m
复制代码
```



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092437ffd7ac6-20210628163233287.png)



因为是在ARC下，所以会copy，栈上拷贝到堆上，结构体`__main_block_desc_0`中有`copy`和`dispose`

```
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
}
复制代码
copy`会调用 `__main_block_copy_0
static void __main_block_copy_0(struct __main_block_impl_0*dst, 
struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->obj, 
(void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);}
复制代码
```

其内部的`_Block_object_assign`会根据代码中的修饰符 `strong`或者`weak`而对其进行强引用或者弱引用。

查看`__main_block_impl_0`

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  //strong 强引用
  NSObject *__strong obj;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSObject *__strong _obj, int flags=0) : obj(_obj) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

复制代码
```

可以看上修饰符是`strong`，所以，调用`_Block_object_assign`时候，会对其进行强引用。

由前面可知

- 当block在栈上时，并不会对__block变量产生强引用
- 当block被copy到堆时
  - 会调用block内部的copy函数
  - copy函数内部会调用`_Block_object_assign`函数
  - `_Block_object_assign`函数会对`__block`变量形成强引用（retain）
- 当block从堆中移除时
  - 会调用block内部的dispose函数
  - dispose函数内部会调用`_Block_object_dispose`函数
  - `_Block_object_dispose`函数会自动释放引用的`__block变量（release）`

#### 拷贝

拷贝的时候，

- 会调用block内部的copy函数
  - copy函数内部会调用`_Block_object_assign`函数
  - `_Block_object_assign`函数会对`__block`变量形成强引用（retain）

中我们知道，如下代码

```
  __block int age = 10;
    YZBlock block = ^{
        age = 20;
        NSLog(@"block内部修改之后age = %d",age);
    };
复制代码
```

局部变量age是在栈上的，在block内部引用age,但是当block从栈上拷贝到堆上的时候，怎么能保证下次block访问age的时候，能访问到呢？因为我们知道栈上的局部变量，随时会销毁的。

假设现在有两个栈上的block,分别是block0和block1,同时引用了了栈上的`__block变量`。现在对block0进行copy操作，我们知道，栈上的block进行copy，就会复制到堆上，也就是说block0会复制到堆上，因为block0持有`__block变量`，所以也会把这个`__block变量`复制到堆上，同时堆上的block0对堆上的`__block变量`是强引用，这样能达到block0随时能访问`__block变量`。



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c0924381b27417.png)



还是上面的例子，刚才block0拷贝到堆上了，现在如果block1也拷贝到堆上，因为刚才变量已经拷贝到堆上，就不需要再次拷贝，只需要把堆上的block1也强引用堆上的变量就可以了。



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092450d7d1a7a.png)



#### 释放

当释放的时候

- 会调用block内部的dispose函数
  - dispose函数内部会调用`_Block_object_dispose`函数
  - `_Block_object_dispose`函数会自动释放引用的`__block变量（release）`

上面的代码中，如果在堆上只有一个block引用`__block变量`,当block销毁时候，直接销毁堆上的`__block变量`，但是如果有两个block引用`__block变量`，就需要当两个block都废弃的时候，才会废弃`__block变量`。



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092448f6daa19.png)



其实，说到底，就是谁使用，谁负责

### 对象类型的`auto变量`、`__block`变量

把前面的都放在一起整理一下，有 auto 变量 num , `__block`变量int, obj 和weakObj2如下

```
 __block int age = 10;
    int num = 8;
    NSObject *obj = [[NSObject alloc]init];
    NSObject *obj2 = [[NSObject alloc]init];
    __weak NSObject *weakObj2 = obj2;
    YZBlock block = ^{
        NSLog(@"age = %d",age);
        NSLog(@"num = %d",num);
        NSLog(@"obj = %p",obj);
        NSLog(@"weakObj2 = %p",weakObj2);
        NSLog(@"block内部修改之后age = %d",age);
	};
    
block();
复制代码
```

执行终端指令

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m
复制代码
```

生成代码如下所示



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09243a7eccf2d.png)



### 被__block修饰的对象类型

- 当`__block`变量在栈上时，不会对指向的对象产生强引用
- 当`__block`变量被copy到堆时
  - 会调用`__block`变量内部的copy函数
  - copy函数内部会调用`_Block_object_assign`函数
  - `_Block_object_assign`函数会根据所指向对象的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用（retain）或者弱引用（注意：这里仅限于ARC时会retain，MRC时不会retain）
- 如果`__block`变量从堆上移除
  - 会调用`__block`变量内部的dispose函数
  - dispose函数内部会调用`_Block_object_dispose`函数
  - `_Block_object_dispose`函数会自动释放指向的对象（release）

### `__block`的`__forwarding`指针

```
//结构体__Block_byref_obj_0中有__forwarding
 struct __Block_byref_obj_0 {
  		void *__isa;
		__Block_byref_obj_0 *__forwarding;
		 int __flags;
 		int __size;
 		void (*__Block_byref_id_object_copy)(void*, void*);
 		void (*__Block_byref_id_object_dispose)(void*);
 		NSObject *__strong obj;
};

// 访问的时候
age->__forwarding->age

复制代码
```

为啥什么不直接用age,而是`age->__forwarding->age`呢？

这是因为，如果`__block`变量在栈上，就可以直接访问，但是如果已经拷贝到了堆上，访问的时候，还去访问栈上的，就会出问题，所以，先根据`__forwarding`找到堆上的地址，然后再取值



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c09243cdf01312.png)



### 总结

- 当block在栈上时，对它们都不会产生强引用
- 当block拷贝到堆上时，都会通过copy函数来处理它们
  - `__block`变量（假设变量名叫做a）
- `_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/)`;
- 对象类型的auto变量（假设变量名叫做p） `_Block_object_assign((void*)&dst->p, (void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/)`;
- 当block从堆上移除时，都会通过dispose函数来释放它们 `__block`变量（假设变量名叫做a） `_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/)`;
- 对象类型的auto变量（假设变量名叫做p） `_Block_object_dispose((void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/)`;

## 循环引用问题

继续探索一下block的循环引用问题。

看如下代码，有个Person类，里面两个属性，分别是block和age

```
#import <Foundation/Foundation.h>

typedef void (^YZBlock) (void);

@interface YZPerson : NSObject
@property (copy, nonatomic) YZBlock block;
@property (assign, nonatomic) int age;
@end


#import "YZPerson.h"

@implementation YZPerson
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end

复制代码
```

main.m中如下代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        YZPerson *person = [[YZPerson alloc] init];
        person.age = 10;
        person.block = ^{
             NSLog(@"person.age--- %d",person.age);
        };
        NSLog(@"--------");

    }
    return 0;
}
复制代码
```

输出只有

> iOS-block[38362:358749] --------

也就是说程序结束，person都没有释放，造成了内存泄漏。

### 循环引用原因

下面这行代码，是有个person指针，指向了YZPerson对象

```
YZPerson *person = [[YZPerson alloc] init];
复制代码
```

执行完

```
 person.block = ^{
             NSLog(@"person.age--- %d",person.age);
        };
复制代码
```

之后，block内部有个强指针指向person，下面代码生成cpp文件

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
    //强指针指向person
  YZPerson *__strong person;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, YZPerson *__strong _person, int flags=0) : person(_person) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
复制代码
```

而block是person的属性

```
@property (copy, nonatomic) YZBlock block;
复制代码
```

当程序退出的时候，局部变量person销毁，但是由于MJPerson和block直接，互相强引用，谁都释放不了。



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092458a4316fa.png)



### `__weak`解决循环引用

为了解决上面的问题，只需要用`__weak`来修饰，即可

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        YZPerson *person = [[YZPerson alloc] init];
        person.age = 10;
        
        __weak YZPerson *weakPerson = person;
        
        person.block = ^{
            NSLog(@"person.age--- %d",weakPerson.age);
        };
        NSLog(@"--------");

    }
    return 0;
}
复制代码
```

编译完成之后是

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
    // block内部对weakPerson是弱引用
  YZPerson *__weak weakPerson;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, YZPerson *__weak _weakPerson, int flags=0) : weakPerson(_weakPerson) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
复制代码
```

当局部变量消失时候，对于YZPseson来说，只有一个若指针指向它，那它就销毁，然后block也销毁。

### `__unsafe_unretained`解决循环引用

除了上面的`__weak`之后，也可以用`__unsafe_unretained`来解决循环引用

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        YZPerson *person = [[YZPerson alloc] init];
        person.age = 10;
        
        __unsafe_unretained YZPerson *weakPerson = person;
        
        person.block = ^{
            NSLog(@"person.age--- %d",weakPerson.age);
        };
        NSLog(@"--------");

    }
    return 0;
}
复制代码
```

对于的cpp文件为

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  YZPerson *__unsafe_unretained weakPerson;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, YZPerson *__unsafe_unretained _weakPerson, int flags=0) : weakPerson(_weakPerson) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
复制代码
```

虽然`__unsafe_unretained`可以解决循环引用，但是最好不要用，因为

- `__weak`：不会产生强引用，指向的对象销毁时，会自动让指针置为nil
- `__unsafe_unretained`：不会产生强引用，不安全，指向的对象销毁时，指针存储的地址值不变

### `__block`解决循环引用

eg:

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
       __block YZPerson *person = [[YZPerson alloc] init];
        person.age = 10;
        person.block = ^{
            NSLog(@"person.age--- %d",person.age);
            //这一句不能少
            person = nil;
        };
        // 必须调用一次
        person.block();
        NSLog(@"--------");
    }
    return 0;
}
复制代码
```

上面的代码中，也是可以解决循环引用的。但是需要注意的是，`person.block();`必须调用一次，为了执行`person = nil;`.

对应的结果如下

- 下面的代码，block会对`__block`产生强引用

```
__block YZPerson *person = [[YZPerson alloc] init];
person.block = ^{
        NSLog(@"person.age--- %d",person.age);
        //这一句不能少
        person = nil;
};
复制代码
```

- person对象本身就对block是强引用

```
@property (copy, nonatomic) YZBlock block;
复制代码
```

- `__block`对person产生强引用

```
struct __Block_byref_person_0 {
  void *__isa;
__Block_byref_person_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
    //`__block`对person产生强引用
 YZPerson *__strong person;
};
复制代码
```

所以他们的引用关系如图



![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16c092459f38a181.png)



当执行完`person = nil`时候,`__block`解除对person的引用,进而，全都解除释放了。 但是必须调用`person = nil`才可以，否则，不能解除循环引用

### 小结

通过前面的分析，我们知道，ARC下，上面三种方式对比，最好的是`__weak`

### MRC下注意点

如果再MRC下，因为不支持弱指针`__weak`，所以，只能是`__unsafe_unretained`或者`__block`来解决循环引用

## 结束

回到最开始的问题

- block的原理是怎样的？本质是什么？
- `__block`的作用是什么？有什么使用注意点？
- block的属性修饰词为什么是copy？使用block有哪些使用注意？
- block一旦没有进行copy操作，就不会在堆上
- block在修改NSMutableArray，需不需要添加__block？