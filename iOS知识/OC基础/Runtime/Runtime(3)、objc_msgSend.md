# Runtime(三)、objc_msgSend

# 一、介绍与应用

## 1.1 `objc_msgSend`

在`Objective-C`中调用方法，称为`消息传递`，消息有`名称（name）`或`选择子（selector）`，可以接收参数，而且可能还有返回值。

`objc_msgSend`其实就是`消息传递`在底层C语言的函数实现，在`Objective-C`中，大部分方法调用都是经过`objc_msgSend`来实现的。当然，除去`load`方法·等特殊情况。

一般，给对象发送消息如下：

```objective-c
id retrunValue = [someObject messageName:parameter]；
```

someObject是`消息接收者`，messageName是`选择子`，`选择子`与`参数`合起来称为`消息`。在底层，编译器收到之后，将其转换`obj_msgSend`函数，其函数声明如下：

```objective-c
void objc_msgSend(void /* id self, SEL op, ... */ )
```

- 第一个参数为消息接收者
- 第二个参数为SEL

上面给对象发送之后转换即为：

```objective-c
id retrunValue = objc_msgSend(someObject, @selector(messageName:), parameter);
```

`objc_msgSend`会根据接受者与选择子的类型来调用适当的方法。

## 1.2 应用

下面我们通过直接调用`objc_msgSend`方法，来看看它是如何调用的。

```objective-c
NSMutableArray *array = [[NSMutableArray alloc] init];
[array addObject:@"dog"];
NSInteger index = [array indexOfObject:@"dog"];
NSString *last = [array lastObject];
[array removeLastObject];
```

将上面方法，改为`objc_msgSend`调用如下：

```c++
NSMutableArray *array = ( (NSMutableArray * (*) (id, SEL)) objc_msgSend) ( (id)[NSMutableArray class], @selector(alloc) );
        array = ( (NSMutableArray * (*) (id, SEL)) objc_msgSend) ( (id)array, @selector(init));
        ( (void (*) (id, SEL, NSString *)) objc_msgSend) ( (id)array, @selector(addObject:), @"dog");
NSInteger index = ( (NSInteger (*) (id, SEL, NSString *)) objc_msgSend) ( (id)array, @selector(indexOfObject:), @"dog");
NSString *last = ( (NSString * (*) (id, SEL)) objc_msgSend) ( (id)array, @selector(lastObject));
        ( (void (*) (id, SEL)) objc_msgSend) ( (id)array, @selector(removeLastObject));
```

## 1.3 为什么需要objc_msgSend

在C语言中，使用`静态绑定（static binding）`来实现函数调用，即在编译期就决定运行时所应调用的函数。

看一个实例：

```c
#import <stdio.h>
void printHello() {
    printf("Hello world!\n");	
}

void printGoodbye() {
	    printf("Good bye!\n");	
}

void doTheThing(int type) {
   if(type == 0){
      printHello();
   }else{
      printGoodbye();
   }
   return 0;
}
```

如果不考虑内联，那么编译器在编译代码的时候就已经知道程序中有`printHello`和`printGoodbye`的函数了，于是就直接生成调用这些函数的指令。而函数地址实际上是硬编码在指令之中。

若是将刚才的代码改成下面：

```c
#import <stdio.h>
void printHello() {
    printf("Hello world!\n");	
}

void printGoodbye() {
    printf("Good bye!\n");	
}

void doTheThing(int type) {
    void (*fnc)();
    if(type == 0){
        fnc = printHello;
    }else{
        fnc = printGoodbye;
    }
    fnc();
    return 0;
}
```

这时就用到了`动态绑定`，因为所要调用的函数直到运行期才能确定。编译器在这种情况下生成的指令与刚才不同，在第一个例子中，`if`与`else`都有函数调用指令。而在第二个例子中，只有一个函数调用指令，不过待调用的函数地址无法硬编码在指令之中，而是要在运行期读取出来。

在`Objective-C`，如果要向某对象发送消息，就会使用`动态绑定`机制来决定需要调用的方法。在底层，所有方法都是普通的C语言函数，然而对象收到消息之后，调用哪个方法则由运行期决定，甚至可以在运行期改变。

`objc_msgSend`，就是承载`Objective-C` 中`动态绑定机`制的函数。

## 1.4 更多的`objc_msgSend`函数

类比`objc_msgSend`函数，还有几个类似的方法可以在`<objc/message.h>`头文件里找到：

```c
//Sends a message with a data-structure return value to an instance of a class.
void objc_msgSend_stret(id self, SEL op, ...)
double objc_msgSend_fpret(id self, SEL op, ...)

id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
```

关于上面三个函数，摘抄一段说明，没有去实证：

> `objc_msgSend_stret`：如果待发送的消息要返回结构体，那么可交由此函数处理。只有当CPU的寄存器能够容纳得下消息返回类型时，这个函数才能处理此消息。若是返回值无法容纳于CPU寄存器中（比如说返回的结构体太大了），那么就由另一个函数执行派发。此时，那个函数会通过分配在栈上的某个变量来处理消息所返回的结构体。

> `objc_msgSend_fpret`：如果消息返回的是浮点数，那么可交由此函数处理。在某些架构的CPU中调用函数时，需要对“浮点数寄存器”（floating-point register）做特殊处理，也就是说，通常所用的objc_msgSend在这种情况下并不合适。这个函数是为了处理x86等架构CPU中某些令人稍觉惊讶的奇怪状况。

> `objc_msgSendSuper`：如果要给超类发消息，例如[super message:parameter]，那么就交由此函数处理。也有另外两个与objc_msgSend_stret和objc_msgSend_fpret等效的函数，用于处理发给super的相应消息。

# 二、探索`objc_msgSend`

## 2.1 分阶段流程

![image-20181217220424239](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-140425.png)

## 2.2 源码导读

透过汇编的流程，对流程进行梳理：

![image-20181217221552462](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-141552.png)

接下来的进入源码：

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/20191011060811.png)其中，最后一步：`_objc_msgForward_impcache`又是汇编，但是有高手已经反编译出来了。

# 三、消息发送

```c
void objc_msgSend(id receiver, @selector(), ...)
```

`消息发送`的第一个流程，就是消息发送，这一步主要在**类及其父类**的*方法缓存以及方法列表*中寻找是否有对应的方法。

首先，会根据`消息接收者`所属的类，查找类”方法列表“，若能找到与”选择子“名称相符的，即跳转其实现代码。否则，按接受者的继承体系继续向上查找，等找到合适的方法之后再跳转。

如果，最终未找到，那就执行`动态方法解析`的流程。

上面的过程会造成性能上的损失，鉴于此，`objc_msgSend`会在接受者第一次查找方法后，将该方法及其跳转地址缓存在`哈希表`中，每个类都有这样一块缓存，后面发送消息，会先哈希表搜寻，并实现快速跳转。当然，相对`静态绑定`这当然更慢。但是，实际上，这并不会造成程序的性能瓶颈所在。假如，真的是瓶颈，你大可以只编写纯C函数。

从[objc源码](https://opensource.apple.com/tarballs/objc4/)归纳出如下流程：

![image-20181217215122938](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-135123.png)

针对上面的流程，需要说明几点。

## 3.1 在方法列表中查找

在方法列表中查找方法，系统为了提高效率，做了如下区分：

- 已排序的方法列表：二分查找
- 未排序的方法列表：遍历

```objective-c
//查找类的某个分类或类的方法列表————一维数组
static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        //已排序：二分查找
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // 未排序：线性遍历寻找方法
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }
    return nil;
}
```

## 3.2 在父类方法列表中查找

如果在父类方法列表中查找到方法，那么就**缓存到当前`receiverClass`中。**

```c++
// Superclass cache. 从父类方法缓存中查找
 imp = cache_getImp(curClass, sel);
 if (imp) {
     //在父类缓存中找到方法
     if (imp != (IMP)_objc_msgForward_impcache) {
         // Found the method in a superclass. Cache it in this class.
         // 将父类缓存中的方法，缓存到自身类中，结束查找
         log_and_fill_cache(cls, imp, sel, inst, curClass);
         goto done;
     }
 }
```

# 四、动态方法解析

假如在消息发送过程中，没有查找到方法，那么就会进入动态方法解析。

动态方法解析就是在运行时临时添加一个方法实现，来进行消息的处理。

添加方法的函数是：

```c
/*
cls: 需要添加方法的对象
name: selector 方法名
imp: 对应的函数实现
types: 函数对应的编码
*/
class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, 
                const char * _Nullable types) 
```

对应的，下图是动态解析的一个流程：

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-194549.png)动态方法解析的对应的[示例代码](https://github.com/wenghengcong/LearnCollection/tree/master/Runtime/02动态方法解析)。

## 4.1 对象方法与类方法

动态方法解析，可以添加处理对象方法也可以处理类方法。

但是，注意区别的是，类方法，需要给其元类对象添加方法，而实例对象，是给其类对象添加方法。

这也很好理解，因为：

- 调用对象方法，查找方法是去类对象方法列表；
- 调用类方法，是去元类方法列表中找；

如下，是一个示例：

```objective-c
void notFound_eat(id self, SEL _cmd)
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    NSLog(@"current in method %s", __func__);
}

//对象方法解析
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    if (sel == @selector(eat)) {
        // 注意添加到self，此处即类对象
        class_addMethod(self, sel, (IMP)notFound_eat, "v16@0:8");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

//类方法解析
+ (BOOL)resolveClassMethod:(SEL)sel
{
    if (sel == @selector(learn)) {
        // 第一个参数是object_getClass(self)
        class_addMethod(object_getClass(self), sel, (IMP)notFound_learn, "v16@0:8");
        return YES;
    }
    return [super resolveClassMethod:sel];
}
```

## 4.2 `class_addMethod`

下面是`class_addMethod`添加的两种方式：

这个标记有何作用？

因为动态解析之后，其实还是又重新走消息发送的阶段了。之所以加这个标记，是为了打破：

**消息发送->动态方法解析->消息发送->动态方法解析….**这个无限循环。

只会执行一次：消息发送->动态方法解析->消息发送->消息转发。

## 4.4 `@dynamic`的实现

动态方法解析，最佳的一个实践用例就是，`@dynamic`的实现。

`@dynamic`是告诉编译器不用自动生成getter和setter的实现，等到运行时再添加方法实现

![image-20181218040724802](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-200725.png)

# 五、消息转发

在消息发送——没有在缓存和方法列表中找到，也没有在动态方法解析时，添加方法。就会走到消息转发流程。

消息转发流程，分类两步：

1. 寻找备援接收者
2. 完整的消息转发

以下是流程图：

![image-20181218042334458](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-202334.png)

## 5.1 备援接收者

备援接收者，含义清晰，相当于“**这条消息，我不想要接收，有个备份对象来接收**”。

备援接收者，在下面方法实现：

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

在这一步，运行期系统会问它：**是否把这条消息转给其他接收者来处理。**

若方法能找到备援者对象，将其返回，否则返回`nil`。通过此方案，可以`组合（composition）`来模拟出`多重继承（multiple inheritance）`的某些特性。在一个对象内部，可能还有其他一系列对象，该对象可以经由此方法将能够处理某选择子的相关内部对象返回，如此一来，从外部看来，好像是该对象亲自来处理这些消息似的。

需要注意的是，这一步是不能改变消息内容的，如果要达到这个目的，就得通过完整的消息转发机制来做。

![image-20181218043635432](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-17-203635.png)

## 5.2. 完整的消息转发

在没有备援接收者的情况下，就会进入完整的消息转发流程中。

完整的消息转发，也分为两步：

1. 获取方法签名
2. 进行转发

![image-20181218043850598](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-203851.png)

### 5.2.1 方法签名

方法签名，可以通过下面方式获取。

![image-20181218044902283](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-204902.png)

### 5.2.2 `forwardInvocation`

`forwardInvocation`方法，非常强大，可定制性程度极高，赋予了其极大的权限。

`NSInvocation`是一个封装了方法调用的类，把与尚未处理那条消息有关的全部细节都封于其中，此对象包含`选择子`、`目标（target）`和`参数`。在触发`NSInvocation`对象时，`消息派发系统`会将消息指派给目标对象。

当然，该方法也可以直接将消息转给`备援接收者`，但是在上一步中即可做到，所以一般到了这一步，都会修改消息内容，来做只有它能做的事。

**但需要注意的时，在使用`NSInvocation`对象`target`时，`target`是`assign`类型。**

![image-20181218044109292](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-204109.png)

## 5.3 类方法的消息转发

针对备援接收者及完整的消息转发流程，其实平时开发中，一般都认为只有对象方法可以实现`消息转发`。

其实系统也**支持对类方法的消息转发**。

只需要将智能提示后的对象方法前面`-`修改成`+`，即可实现类方法的`消息转发`。

![image-20181218044230651](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-17-204231.png)

# 六、unrecognized selector

在历经千山万水之后，仍然走到这一步。

苍天绕过谁，那就抛出我们常见的错误吧！

> -[__NSCFNumber lowercaseString]: unrecognized selector sent to instance 0x87
> *** Terminating app due to uncaught exception ‘NSInvalidArgumentException’,reason:’- [__NSCFNumber lowercaseString]: unrecognized selector sent to instance 0x87