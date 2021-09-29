# iOS NSNotificationCenter 使用姿势详解

最近在做平板的过程中，发现了一些很不规范的代码。偶然修复支付bug的时候，看到其他项目代码，使用通知的地方没有移除，我以为我这个模块的支付闪退是因为他通知没有移除的缘故。而在debug和看了具体的代码的时候才发现和这里没有关系。在我印象中，曾经因为没有移除通知而遇到闪退的问题。所以让我很意外，于是写了个demo研究了下，同时来讲下`NSNotificationCenter`使用的正确姿势。

## NSNotificationCenter

对于这个没必要多说，就是一个消息通知机制，类似广播。观察者只需要向消息中心注册感兴趣的东西，当有地方发出这个消息的时候，通知中心会发送给注册这个消息的对象。这样也起到了多个对象之间解耦的作用。苹果给我们封装了这个`NSNotificationCenter`，让我们可以很方便的进行通知的注册和移除。然而，有些人的姿势还是有点小问题的，下面就看看正确的姿势吧！

## 正确姿势之`remove`

只要往`NSNotificationCenter`注册了，就必须有`remove`的存在，这点是大家共识的。但是大家在使用的时候发现，在`UIViewController` 中`addObserver`后没有移除，好像也没有挂！我想很多人可能和我有一样的疑问，是不是因为使用了ARC？在你对象销毁的时候自动置为`nil`了呢？或者苹果在实现这个类的时候用了什么神奇的方式呢？下面我们就一步步来探究下。

首先，向`NSNotificationCenter`中`addObserver`后，并没有对这个对象进行引用计数加1操作，所以它只是保存了地址。为了验证这个操作，我们来做下代码的测试。

一个测试类，用来注册通知：



```objc
@implementation MRCObject

- (id)init
{
    if (self = [super init]) {
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
    }
    return self;
}

- (void)test
{
    NSLog(@"=================");
}

- (void)dealloc
{
    [super dealloc];
}

@end
```

这个类很简单，就是在初始化的时候，给他注册一个通知。但是在销毁的时候不进行`remove`操作。我们在VC中创建这个对象后，然后销毁，最后发送这个通知：



```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    MRCObject *obj = [[MRCObject alloc] init];
    [obj release];

    [[NSNotificationCenter defaultCenter] postNotificationName:@"test" object:nil];
}
```

在进入这个vc后，我们发现挂了。。而打印出的信息是：



```objc
2015-01-19 22:49:06.655 测试[1158:286268] *** -[MRCObject test]: message sent to deallocated instance 0x17000e5b0
```

我们可以发现，向野指针对象发送了消息，所以挂掉了。从这点来看，苹果实现也基本差不多是这样的，只保存了个对象的地址，并没有在销毁的时候置为`nil`。

**这点就可以证明，`addObserver`后，必须要有`remove`操作。**

现在我们在`UIViewController`中注册通知，不移除，看看会不会挂掉。



```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
}
```

首先用`navigationController`进入到这个页面，然后`pop`出去。最后点击发送通知的按钮事件：



```objc
- (void)didButtonClicked:(id)sender
{
    [[NSNotificationCenter defaultCenter] postNotificationName:@"test" object:nil];
}
```

无论你怎么点击这个按钮，他就是不挂！这下，是不是很郁闷了？我们可以找找看，你代码里面没有`remove`操作，但是`NSNotificationCenter`那边已经移除了，不然肯定会出现上面野指针的问题。看来看去，也只能说明是`UIViewController`自己销毁的时候帮我们暗地里移除了。

那我们如何证明呢？由于我们看不到源码，所以也不知道有没有调用。这个时候，我们可以从这个通知中心下手！！！怎么下手呢？我只要证明`UIViewController`在销毁的时候调用了`remove`方法，就可以证明我们的猜想是对的了！这个时候，就需要用到我们强大的**类别**这个特性了。我们为`NSNotificationCenter`添加个类别，重写他的`- (void)removeObserver:(id)observer`方法：



```objc
- (void)removeObserver:(id)observer
{
    NSLog(@"====%@ remove===", [observer class]);
}
```

这样在我们VC中导入这个类别，然后`pop`出来，看看发生了什么！



```objc
2015-01-19 22:59:00.580 测试[1181:288728] ====TestViewController remove===
```

怎么样？是不是可以证明系统的`UIViewController`在销毁的时候调用了这个方法。（不建议大家在开发的时候用类别的方式覆盖原有的方法，由于类别方法具有更高的优先权，所以有可能影响到其他地方。这里只是调试用）。

以上也提醒我们，在你不是销毁的时候，千万不要直接调用`[[NSNotificationCenter defaultCenter] removeObserver:self];` 这个方法，因为你有可能**移除了系统注册的通知**。

## 正确姿势之注意重复`addObserver`

在我们开发中，我们经常可以看到这样的代码：



```objc
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
}

- (void)viewWillDisappear:(BOOL)animated
{
    [super viewWillDisappear:animated];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:@"test" object:nil];
}
```

就是在页面出现的时候注册通知，页面消失时移除通知。你这边可要注意了，一定要成双成对出现，如果你只在`viewWillAppear 中 addObserver`没有在`viewWillDisappear 中 removeObserver`那么当消息发生的时候，你的方法会被调用多次，这点必须牢记在心。

## 正确姿势之多线程通知

首先看下苹果的官方说明：

> Regular notification centers deliver notifications on the thread in which the notification was posted. Distributed notification centers deliver notifications on the main thread. At times, you may require notifications to be delivered on a particular thread that is determined by you instead of the notification center. For example, if an object running in a background thread is listening for notifications from the user interface, such as a window closing, you would like to receive the notifications in the background thread instead of the main thread. In these cases, you must capture the notifications as they are delivered on the default thread and redirect them to the appropriate thread.

意思很简单，`NSNotificationCenter`消息的接受线程是基于发送消息的线程的。也就是同步的，因此，有时候，你发送的消息可能不在主线程，而大家都知道操作UI必须在主线程，不然会出现不响应的情况。所以，在你收到消息通知的时候，注意选择你要执行的线程。下面看个示例代码



```objc
//接受消息通知的回调
- (void)test
{
    if ([[NSThread currentThread] isMainThread]) {
        NSLog(@"main");
    } else {
        NSLog(@"not main");
    }
    dispatch_async(dispatch_get_main_queue(), ^{
        //do your UI
    });

}

//发送消息的线程
- (void)sendNotification
{
    dispatch_queue_t defaultQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(defaultQueue, ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"test" object:nil];
    });
}
```

## 总结

通知平常使用的知识点差不多就这么多。希望对大家有帮助。最后，代码一定要养成良好的习惯，该移除的还是要移除。