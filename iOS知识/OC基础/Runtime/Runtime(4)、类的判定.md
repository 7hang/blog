# Runtime（五）类的判定

我们经常需要在开发中判定某一个类，比如下面场景：

- 判定在某一个页面：`isMemberOfClass`来指定只有在某页面下的操作。

- 判断是否某个类，用于容错，这很常见。

  ```objective-c
  if([BFUserModel isKindOfClass:BFModel]) {
      //.....针对user model
  }
  ```

这些涉及到**类的判定**的，我们下面会从**一个关键字和两个方法**去阐述。

- `super`关键字：调用父类方法的关键字
- `isMemberOfClass`方法：判断某个instance（class）是否是对应的class（meta-class）对象
- `isKindOfClass`方法：判断某个instance（class）是否是对应的class（meta-class）继承体系下的对象。



**除了，需要关注本文讲的内容。还需关注的是，本文将iOS底层挖掘的方式大致都展现了。**

# 一、`super`

`super`平时开发中，调用的非常频繁，在`viewcontroller`以及`view`的生命各个周期方法，我们都会先调用父类的对应实现。

下面我们根据实例，代码[在这儿](https://github.com/wenghengcong/LearnCollection/tree/master/Runtime/super)。去分析`super`的含义。

我们定义下面两个有继承关系的类：

```objective-c
@interface BFPerson : NSObject
- (void)eat;
@end
@implementation BFPerson
- (void)eat
{
    NSLog(@"%s", __func__);
}
@end
//--------------
@interface BFBoy : BFPerson
- (void)eat;
@end

@implementation BFBoy
- (void)eat
{
    [super eat];
}
@end
```

下面是调用代码：

```objective-c
BFBoy *boy = [[BFBoy alloc] init];
[boy eat];
```

## 1.1 如何分析

我们下面将会通过各种方式来逐一对`super`进行剖析。

### 1.1.1 C++代码分析

先将代码重写为C++代码

```shell
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc BFBoy.m
```

重写后，下面是`-[BFBoy eat]`函数的实现（删去了类型转换）：

```c++
static void _I_BFBoy_eat(BFBoy * self, SEL _cmd) {
    (void *)objc_msgSendSuper)(
        (__rw_objc_super){
            self, 
            class_getSuperclass(objc_getClass("BFBoy"))
        }, 
        sel_registerName("eat"));
}
```

从上面转换我们可以初步看出：

`[super eat]`转换后调用了`objc_msgSendSuper`类的函数，并且其参数比较陌生。是一个结构体，其含义似乎表示的是`super`结构体。

当然，我们从之前历次的分析来看，**重写其实并不完全是运行时的行为表现，所以，这种方式仅做参考。**

### 1.1.2 Developer Document登场

之前我们在分析各种各样的问题的时候，其实该方法，或者该工具一直没有正式登场。

下面我们在Xcode中，**⌘+⇧+0** 调出开发文档。

并在其中搜索上一步提到的`objc_msgSendSuper`。

![image-20181216210727865](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-130728.png)

以上是，**苹果开发文档给出的说明，权威清晰**。缺点是，有时候文档并没有说明或者泛泛而谈，甚至不知所云。

### 1.1.3 源码

同样的，[objc 源码](https://opensource.apple.com/tarballs/objc4/)也是我们**找定义、找方法、找逻辑**的最佳选择。

![image-20181216211057367](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-131057.png)

这种方式，只要**获取的源码是最新的，其对本质的还原度，最直接、还原度最高。缺点是，阅读源码费时费力费脑。**

但阅读源码是基本功，要炉火纯青。

### 1.1.4 LLDB调试

以上都是从文档或者未运行的转译代码中，对代码本质的挖掘，如果以上过程能完成，基本对本质了解已经七七八八。

但是，OC是一门动态性强大的语言，所以，一切以运行时为准。

我们还需要观察运行时的状态。

至于如何运用LLDB进行调试，可以查看[待补-iOS调试](https://wenghengcong.com/posts/bb109840/Next_link)。

### 1.1.5 LLVM中间代码分析

这是一种全新的方式，是在编译过程中，产生的中间代码，来对中间代码进行剖析。

那么编译过程是如何产生中间代码的？

如下图：

![image-20181216221320861](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-141321.png)

当然，有了中间代码，我们还要学会看，可参考[官方文档](https://llvm.org/docs/LangRef.html)。

我们攫取了一些常用的语法：

![image-20181216221542427](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-141543.png)

### 1.1.6 汇编分析

有时候，苹果没有给出文档，也无相关源码，甚至Debug时，也是毫无头绪，我们可以通过汇编指令级别的分析，来管中窥豹，或许能发现个所以然。

#### 1.1.6.1 Perform Action

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-141707.png" alt="drawing" style="zoom:50%;" />

下面是对应的汇编源码：

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-142338.png" alt="img" style="zoom:50%;" />

#### 1.1.6.2 Debug Workflow

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-142230.png" alt="img" style="zoom:50%;" />下面是调试时的汇编指令：



![image-20181216222501081](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-142502.png)

结合上面两种汇编方式查看，我们最终确定：

`super`在底层，最终调用的方法是：`objc_msgSendSuper2`。

## 1.2 `super`的本质

经过上面层层剥开，我们现在能确认`objc_msgSendSuper2`，是理解`super`的关键。

而`objc_msgSendSuper2`两个参数，对我们的理解又至关重要：

```c
id _Nullable objc_msgSendSuper2(struct objc_super * _Nonnull super,
                                SEL _Nonnull op,
                                ...);
```

其中`objc_super`：

```c
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class super_class;
};
```

根据上面C++重写后的代码：

```c++
static void _I_BFBoy_eat(BFBoy * self, SEL _cmd) {
    (void *)objc_msgSendSuper)(
        (__rw_objc_super){
            self, 
            class_getSuperclass(objc_getClass("BFBoy"))
        }, 
        sel_registerName("eat"));
}
```

- `receiver`就是`self`本身，即`BFBoy`实例对象**boy**，它表示消息的接收者仍然是**boy**，即仍然是子类对象。
- `super_class`为`BFBoy`的父类`BFPeron`，它表示，要从父类开始查找对应的方法。

综上所述，总结为下图：

![image-20181216232653527](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-152654.png)

## 1.3 小试身手—实例测试

在了解了`super`对应的原理之后，我们就看一个小实例。

```objective-c
@implementation BFBoy
- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"[self class] = %@", [self class]);               
        NSLog(@"[self superclass] = %@", [self superclass]);
        NSLog(@"[super class] = %@", [super class]);       
        NSLog(@"[super superclass] = %@", [super superclass]); 
    }
    return self;
}
@end
```

在`BFBoy`的初始化方法中，上面打印的分别是什么？

结果如下：

![image-20181216230518863](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-150518.png)



和你预想的一致吗？

我们仍然对这个进行一些分析：

1. `super`相关的调用转化为`objc_msgSendSuper2`调用；

2. `objc_msgSendSuper2`开始从`BFPerson`查找`class/superclass`方法；

3. 一直找到`NSObject`，调用`NSObject`的`class/superclass`方法；

   我们从objc源码中找到下面实现：

   ```objective-c
   //NSObject.mm
   - (Class)class {
       return object_getClass(self);
   }
   - (Class)superclass {
       return [self class]->superclass;
   }
   ```

假如父类`BFPerson`未实现的方法，在`NSObject`实现了，而`class/superclass`中的`self`，就是`BFBoy`实例对象boy本身，那么上面的结果你明白了吗？

# 二、类的判定

## 2.1 `isMemberOfClass`

从`NSObject.mm`源码中得到：

```objective-c
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

从源码可以得出：

- `instance对象`：判断对应的`class对象`是否和`传入的对象`相同。
- `class对象`：判断对应的`meta-class对象`是否和`传入的对象`相同。
- 传入的对象，`intance对象`、`class对象`、`meta-class`对象都可以。

## 2.2 `isKindOfClass`

```objective-c
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

- `instance对象`：判断对应的`class对象及其父类对象`是否和`传入的对象`相同。

- `class对象`：判断对应的`meta-class对象`及其父类对象是否和`传入的对象`相同。

- 传入的对象，`intance对象`、`class对象`、`meta-class`对象都可以。

## 2.3 实例

```objective-c
// 这句代码的方法调用者不管是哪个类（只要是NSObject体系下的），都返回YES
 NSLog(@"%d", [NSObject isKindOfClass:[NSObject class]]);        // 1
 NSLog(@"%d", [NSObject isMemberOfClass:[NSObject class]]);      // 0
 NSLog(@"%d", [BFPerson isKindOfClass:[BFPerson class]]);        // 0
 NSLog(@"%d", [BFPerson isMemberOfClass:[BFPerson class]]);      // 0
 NSLog(@"------------------");
 
 //正常使用实例对象
 BFPerson *person = [[BFPerson alloc] init];
 NSLog(@"%d", [person isMemberOfClass:[BFPerson class]]);		//1
 NSLog(@"%d", [person isMemberOfClass:[NSObject class]]);		//0
 NSLog(@"%d", [person isKindOfClass:[BFPerson class]]);			//1
 NSLog(@"%d", [person isKindOfClass:[NSObject class]]);			//1
```

上面的输出，有一行需要着重指出：

```
NSLog(@"%d", [NSObject isKindOfClass:[NSObject class]]); 
```

翻译一下：NSObject类的**元类对象**，及其该元类对象的父类对象与NSObject**类对象**相等吗？

第一反应，不相等，元类对象怎么会和类对象相等。但是其结果为1。

如下图：

- **NSObject元类对象在其寻找父类，找到根元类后，会指向类对象，这是一个陷阱。**

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-16-202202.png" alt="image-20181217042201468" style="zoom:50%;" />

# 三、总结

## 3.1 super

**[super message]**的底层实现。

1. 消息接收者仍然是子类对象
2. 从父类开始查找方法的实现

## 3.2 类的判定

![image-20181217044306549](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-16-204307.png)

![image-20181217044319962](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-16-204320.png)

