# 内存管理（六）autorelease

[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool/)

本文分为三部分：

[一](https://wenghengcong.com/posts/c458827d/#一、自动释放池)、[二](https://wenghengcong.com/posts/c458827d/#二、autorelease与方法返回值)两节主要讲述自动释放池的概念和应用，其中二中还详述了autorelease与方法返回值的关系。

[三](https://wenghengcong.com/posts/c458827d/#三、原理)探索了自动释放池的原理，但是没有对源码进行更多的描述，只是阐述了机制的运作。

# 一、自动释放池

自动释放池，autorelease pool，是MRC就存在的一种机制，这种机制使得”**对象延迟释放**“成为可能。

![image-20190107151958428](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-071959.png)

只要将对象放入自动释放池后，系统会在合适的时机，对对象进行释放操作。

不管是MRC，还是ARC下，自动释放池的机制都是相同的，不同的关键字所代表的含义也是一致的。比如`autorelease`和`__autorelease`，`@autoreleasepool{}`，但需要注意的是`NSAutoreleasePool`和前面几个机制不一样。

## 1.1 MRC

![image-20190107152040102](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-072040.png)

## 1.2 ARC

__weak修饰的变量会加到自动释放池中，防止被释放。

![image-20190107152241173](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-072241.png)

ARC下，另外一个常见的使用——降低内存峰值

![image-20190107153428043](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-073502.png)



# 二、autorelease与方法返回值

在MRC时代，autorelease另外一个常用的场景就是方法返回值。

![image-20190107152618795](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-072619.png)

下面，我们声明一个类，并且通过不同的方式返回一个类对象：

```objective-c
@interface BFPerson : NSObject

@property (nonatomic, retain) NSString *name;

- (instancetype)initWithName:(NSString *)name;
+ (instancetype)personWithName:(NSString *)name;

@end
```

按照苹果的命名规则，`-initWithName:`是`retained return value`，方法内部没有对对象进行释放，调用者拥有对象，需要释放。

而`-personWithName:`就是`unretained return value`，内部进行了释放，但是是**延迟释放**，调用者不拥有对象，不需要进行释放。

## 2.1 retained return value

![image-20190107152726653](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-072726.png)

## 2.2 unretained return value

![image-20190107152753050](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-072754.png)

# 三、原理

 自动释放池的机制，奥秘在于”自动”，而”自动”很容易让我们联想到iOS中最常见的”自动”机制——RunLoop。

 而释放池，是一个池子，其实是一种数据结构，这种结构需要支持在池子里面的对象，能在恰当的时机发送`release`消息。

 所以，了解自动释放池的原理，就是两个目标：**掌握”自动”释放的时机**，以及**如何构建这个池子**。

## 3.1 池子

我们先尝试如下代码：

```objective-c
@autoreleasepool {
    BFPerson *person = [[[BFPerson alloc] init] autorelease];
}
```

将其转为C++：

```shell
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc ViewController.m
```

我们得到如下：

```objective-c
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
            BFPerson *person = ((BFPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((BFPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((BFPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("BFPerson"), sel_registerName("alloc")), sel_registerName("init")), sel_registerName("autorelease"));
}
```

刨去复杂的结构转换，关注`__AtAutoreleasePool __autoreleasepool;`，是的，这个就是”池子”本身。

```objective-c
struct __AtAutoreleasePool {
    // 构造函数
   __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
    // 析构函数
   ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
   void * atautoreleasepoolobj;
};
```

根据其构造和析构函数，最后转换为：

![image-20190107163117207](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-083117.png)

在[objc源码](https://opensource.apple.com/tarballs/objc4/)源码中对应的函数为：

```objective-c
void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

其实，`AutoreleasePoolPage`才是管理`autorelease`对象的真正结构。

![image-20190107164027577](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-084028.png)

上图展示了`AutoreleasePoolPage`对象结构，下面一览其存储结构：

![image-20190107164122642](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-084122.png)

 `AutoreleasePoolPage`这个”池子”负责了所有的`autorelease`对象的管理，每个Page除了存放自己的成员变量之外，其余空间用来存放`autorelease`对象，如果当前Page不够存储，会新建Page，并进行双链表的连接，然后将`autorelease`对象存储在新Page。而`next`指针则指向了下一个存放`autorelease`对象的地址。

## 3.2 时机

在了解其自动释放池的结构之后，就剩下如何触发”自动释放”这个问题。

在了解这个时机之前，我们先看下上面一节重写为C++代码中**push**和**pop**都做了什么。

![image-20190107231115608](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-151115.png)

即`push`和`pop`，是对应进行清理工作的两个动作。那么这两个动作何时调用，即时机，就是我们需要探讨的。

我们在[RunLoop（四）应用](http://wenghengcong.com/posts/25ecb79e/)中讲述了RunLoop是如何调度`autoreleasepool`的。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

启动时，会在Runloop里注册两个Observer。

- 第一个Observer会监听
  - 即将进入Loop，其回调内会调用**objc_autoreleasePoolPush()\**，向当前的**AutoreleasePoolPage**增加一个**哨兵对象**标志创建自动释放池。这个Observer的order是-2147483647优先级最高，确保发生在所有回调操作之前。
- 第二个Observer会监听
  - 即将进入休眠：会调用**objc_autoreleasePoolPop()** 和 **objc_autoreleasePoolPush()** ，根据情况从最新加入的对象一直往前清理直到遇到哨兵对象。
  - 即将退出RunLoop：会调用**objc_autoreleasePoolPop()** 释放自动自动释放池内对象。这个Observer的order是2147483647，优先级最低，确保发生在所有回调操作之后。

最后，我们绘制如下图：

![image-20190107231316610](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-151317.png)