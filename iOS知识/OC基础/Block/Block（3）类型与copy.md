# Block（三）类型与copy

上篇文章讲述了Block对象如何捕获不同类型的变量，现在开始要追踪Block本身类型的变化，而且Block对象类型和捕获是有关联的。

# 一、类型

## 1.1 类型归纳

![block类型](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-08-082034.png)



## 1.2 验证

**注意！！！**

我们此处验证是通过`class`方法获取其类型，也可以通过重写代码，但是重写代码得到的C++代码，并不能正确反映Block的类型，因为编译期和运行期还是有区别，重写并不能百分百对应运行期的逻辑。

```objective-c
int good = 1;
void (^globalBlock1)(void) = ^{
    NSLog(@"%d", good);
    NSLog(@"globalBlock1->ARC:__NSGlobalBlock__  MRC:__NSGlobalBlock__");
};

- (void)testBlockType
{
    void (^globalBlock2)(void) = ^{
        NSLog(@"globalBlock2->ARC:__NSGlobalBlock__  MRC:__NSGlobalBlock__");
    };
    
    // 注意：MRC下是__NSStackBlock，在ARC下会自动copy到堆，成为__NSMallocBlock
    int age = 10;
    void (^stackBlock1)(void) = ^{
        NSLog(@"stackBlock1->ARC:__NSMallocBlock  MRC:__NSStackBlock");
        NSLog(@"%d", age);
    };

    // 注意：^{ NSLog(@"temp block-> ARC:__NSStackBlock  MRC:__NSStackBlock"); NSLog(@"%d", age) } 为自动变量
    // 在此处为测试ARC情况下的Stack Block
    NSLog(@"class: %@ %@ %@ %@",[globalBlock1 class], [globalBlock2 class], [stackBlock1 class], [^{
        NSLog(@"stackBlock2-> ARC:__NSStackBlock  MRC:__NSStackBlock");
        NSLog(@"%d", age);
    } class]);
}
```

根据以上代码，我们得到以下验证结果：

- globalBlock1/globalBlock2
  - MRC：**_\*NSGlobalBlock_\***
  - ARC：**_\*NSGlobalBlock_\***
- stackBlock1
  - MRC：**__NSStackBlock__**
  - ARC：**__NSMallocBlock__ **
- stackBlock2
  - MRC：**__NSStackBlock__**
  - ARC：**__NSStackBlock__**

我们看到，针对`__NSGlobalBlock__`在`MRC`或`ARC`下的情形是一致的。

但是`__NSStackBlock`代码中赋值给**stackBlock1**在ARC下会自动转换为`__NSMallocBlock`。

我们先梳理一下，什么情况下会将Block对象分配在数据段（`__NSGlobalBlock`），又是什么时候分配在栈上（`__NSStackBlock`）。

- 访问了auto变量就分配在`__NSStackBlock`；
- 未访问auto变量将分配在`__NSGlobalBlock__`；
- 当`__NSStackBlock`进行copy，则会拷贝到`__NSMallocBlock`



 从上面的捕获情况，我们看看变量与Block类型存在什么关系？

 如果一个变量，可以通过**直接访问（全局变量）**，或者**指针访问（static局部变量）**，那么，通常只有一份拷贝，所有访问，都会访问该变量不变的内存地址，事实上也是这样。

 全局变量，存储于全局存储区，static全局变量也存在该区域。这部分区域，由于该区域不需要在运行时才进行分配的地址，所以，在编译期就能确定，即加入Block捕获的是以上这种类型的变量，Block类型，同样为global性质，即`__NSGlobalBlock__`类型。

 现在理解捕获auto变量，Block类型对应为`__NSStackBlock`或`__NSMallocBlock`，是不是简单多了。`auto`变量无法在编译期准确分配内存，在运行时捕获，当然会分配在对应栈或堆上。

# 二、copy

在开发中，我们捕获局部变量是很常见的事情，如果这种捕获导致，Block对象被分配到栈区，一旦脱离了变量的作用域，也就不能使用Block了，Block的使用就没那么便捷了。

如果分配在栈上的Block对象，进行`copy`操作，将会将Block对象，从栈上拷贝到堆上。

## 2.1 自动复制

**在ARC环境下**，编译器会根据情况自动将栈上的Block复制到堆上，比如以下情况

1. Block作为函数返回值时；
2. 将Block赋值给`__strong`指针时；
3. Block作为Cocoa API中，且方法名含有usingBlock的方法参数时；
4. Block作为GCD API的方法参数时；

下面我们就针对以上1、2情况做如下验证，3、4不作验证：

### 2.1.1 函数返回值

```objective-c
typedef void (^MRCBlock)(void);

MRCBlock mrcblock()
{
    int age = 10;
    return ^{
        NSLog(@"%d", age);
    };
}

- (void)testStackBlockInMRC
{
    MRCBlock block = mrcblock();
    block();
    NSLog(@"%@", [block class]);
}
```

以上代码在`MRC`下，会报错：

![image-20181209051423894](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-08-211424.png)

可以看到，这种情况下Block在栈空间，会报错。

以上，在ARC情况下，再次运行，返回的Block类型，就是`__NSMallocBlock__`。

### 2.1.2 赋值给`__strong`指针

```objective-c
- (void)testBlockWithStrong
{
    int age = 28;
    NSLog(@"%@", [^{
        NSLog(@"stack block %d", age);
    } class]);
    
    void (^block)(void) = ^{
        NSLog(@"malloc block %d", age);
    };
    
    NSLog(@"%@", [block class]);
}
```

以上代码，在ARC环境下，会输出：

> **Block类型与copy[82825:4315549] __NSStackBlock__**
>
> **Block类型与copy[82825:4315549] __NSMallocBlock__**

## 2.2 手动复制

通过调用copy方法，手动进行对Block完成从栈到堆的赋值。

### 2.2.1 属性写法

`MRC`下block属性的建议写法

```objective-c
@property (nonatomic, copy) void (^block)(void);
```

`ARC`下block属性的建议写法

```objective-c
@property (nonatomic, strong) void (^block)(void);
@property (nonatomic, copy)   void (^block)(void);
```

# 三、 总结

![block copy规律](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-09-073256.png)