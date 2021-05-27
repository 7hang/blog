# RunLoop（四）应用

假如你已经阅读完前面的三篇`RunLoop`的文章，就会对`RunLoop`是什么，有一个比较全貌的了解。

本文，将针对`RunLoop`在iOS/macOS中的应用，做一些常用场景的分析。

# 一、线程保活

线程保活，就是利用`RunLoop`的运行循环，使得某个线程能长时间的运行，在有任务时执行任务，没有任务时也保持运行，不退出线程。

# 二、定时器

## 1. 定时器的使用

定时器，在开发中一般使用`NSTimer`，可以产生基于时间的通知，但它并不是实时机制。和输入源一样，定时器也`RunLoop`的特定模式相关。如果定时器所在的模式当前未被`RunLoop`监视，那么定时器并不会被调用。

`NSTimer` 其实就是 `CFRunLoopTimerRef`，两者是 toll-free bridged 的。

`NSTimer`有两种创建方式。

```objective-c
// a. timerWithTimeInterval
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block;

// b. scheduledTimerWithTimeInterval
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block;
```

其中，方式a，仅仅创建，并返回，如果要是的`NSTimer`被调用，需要手动假如`RunLoop`。

```objective-c
// 方式1：创建timer，手动添加到default mode，滑动时定时器停止
static int count = 0;
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
    NSLog(@"%d", ++count);
}];
[[NSRunLoop currentRunLoop] addTimer:timer forMode: NSDefaultRunLoopMode];
```

方式b，创建并且会自动将其添加到当前的`RunLoop`中的`default mode`下。

```objective-c
// 方式2：创建timer默认添加到default mode，滑动时定时器停止
    static int count = 0;
    [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"%d", ++count);
    }];
```

这种方式下，不用再往mode里添加，也能正常调用Timer。

## 2. 滑动时失效

在上面创建`NSTimer`的方式中，不管方式a，还是b，假如页面在滑动ScrollView时，定时器都会停止调用。

因为在滑动ScrollView时，`RunLoop`处于**UITrackingRunLoopMode**运行模式下，该模式中如果不手动添加对应的Timer，是不会有定时器的，所以在滑动时，也就不会调用定时器的回调。

那么，解决方式，就是将定时器，添加到**UITrackingRunLoopMode**下。

```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
```

或者利用之前的Common Mode。

```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

## 3. 不准时

一个 `NSTimer` 注册到 `RunLoop` 后，`RunLoop` 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。`RunLoop`为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

`CADisplayLink` 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 `NSTimer` 并不一样，其内部实际是操作了一个 Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 `NSTimer` 相似），造成界面卡顿的感觉。在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。Facebook 开源的 [AsyncDisplayLink](https://github.com/facebookarchive/AsyncDisplayKit) 就是为了解决界面卡顿的问题，

# 三、其他应用

## 1. AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

```c
observers = (
    // activities = 0x1，监听的是Entry
    "<CFRunLoopObserver>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x1138221b1), context = <CFArray >{type = mutable-small, count = 1, values = (\n\t0 : <0x7ff6e6002058>\n)}}",
    "<CFRunLoopObserver >{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x1138221b1), context = <CFArray>{type = mutable-small, count = 1, values = (\n\t0 : <0x7ff6e6002058>\n)}}"
),
```

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 `RunLoop` 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

- 第一个Observer会监听**Entry**（即将进入Loop），其回调内会调用**objc_autoreleasePoolPush()**向当前的**AutoreleasePoolPage**增加一个**哨兵对象**标志创建自动释放池。这个Observer的order是-2147483647优先级最高，确保发生在所有回调操作之前。
- 第二个Observer会监听`RunLoop`的 **BeforeWaiting**（准备进入休眠）和**Exit**（即将退出Loop）两种状态。
  - 在即将进入休眠时会调用**objc_autoreleasePoolPop()** 和 **objc_autoreleasePoolPush()** 根据情况从最新加入的对象一直往前清理直到遇到哨兵对象。
  - 即将退出RunLoop时会调用**objc_autoreleasePoolPop()** 释放自动自动释放池内对象。这个Observer的order是2147483647，优先级最低，确保发生在所有回调操作之后。

当然你如果需要显式释放，例如循环，可以自己创建AutoreleasePool，否则一般不需要自己创建。

## 2. 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

比如，UIButton点击事件，通过Source1接收后，包装成Event，最后进行分发是由Source0事件回调来处理的。

## 3. 手势识别

```c
"<CFRunLoopObserver 0x60000137cf00 [0x110c33b68]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1133f4473), context = <CFRunLoopObserver context 0x60000097d9d0>}"
```

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 **BeforeWaiting**（Loop即将进入休眠）事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

## 4. 界面更新

```c++
"<CFRunLoopObserver>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x1152506ae), context = <CFRunLoopObserver context 0x0>}"
```

当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 **BeforeWaiting（即将进入休眠）**和 **Exit （即将退出Loop）**事件，回调去执行一个很长的函数：
**_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()**。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

这个函数内部的调用栈大概是这样的：

```c++
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];
```

通常情况下这种方式是完美的，因为除了系统的更新，以及setNeedsDisplay等方法手动触发下一次RunLoop运行的更新。但是如果当前正在执行大量的逻辑运算可能UI的更新就会比较卡，因此facebook推出了[AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit)来解决这个问题。AsyncDisplayKit其实是将UI排版和绘制运算尽可能放到后台，将UI的最终更新操作放到主线程（这一步也必须在主线程完成），同时提供一套类UIView或CALayer的相关属性，尽可能保证开发者的开发习惯。这个过程中AsyncDisplayKit在主线程RunLoop中增加了一个Observer监听即将进入休眠和退出RunLoop两种状态,收到回调时遍历队列中的待处理任务一一执行。

## 5. PerformSelecter

perforSelector有下面三类：

```objective-c
// 1.和RunLoop不相干，底层直接调用objc_sendMsg方法
- (id)performSelector:(SEL)aSelector withObject:(id)object;
// 2. 和RunLoop相关，封装成Source0事件，依赖于RunLoop，若线程无对应的RunLoop，会调用objc_sendMsg执行
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
// 3. 和RunLoop相关，封装成Timers事件
- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;
```

在苹果开源的[objc源码](https://opensource.apple.com/tarballs/objc4/)，我们从**NSObject.mm**得到如下源码：

```objective-c
- (id)performSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL))objc_msgSend)(self, sel);
}

- (id)performSelector:(SEL)sel withObject:(id)obj {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj);
}
```

从源码可以直接看出，在不包含`delay`时，其直接调用`objc_msgSend`，与`RunLoop`无关。

但是，找不到`- (void) performSelector: withObject: afterDelay:`的源码。苹果并没有开源。

我们通过**GNUstep**项目，找到该方法的[Foundation的源码](http://www.gnustep.org/resources/downloads.php)。

```objective-c
- (void) performSelector: (SEL)aSelector
	      withObject: (id)argument
	      afterDelay: (NSTimeInterval)seconds
{
  NSRunLoop		*loop = [NSRunLoop currentRunLoop];
  GSTimedPerformer	*item;
  item = [[GSTimedPerformer alloc] initWithSelector: aSelector
					     target: self
					   argument: argument
					      delay: seconds];
  [[loop _timedPerformers] addObject: item];
  RELEASE(item);
  [loop addTimer: item->timer forMode: NSDefaultRunLoopMode];
}
// GSTimedPerformer对象
@interface GSTimedPerformer: NSObject
{
@public
  SEL		selector;
  id		target;
  id		argument;
  NSTimer	*timer;
}
```

其实本质上，是转换为一个包含`NSTimer`定时器的`GSTimedPerformer`对象，实质上是个`Timers`事件，添加到`RunLoop`中。

**注意：GNUstep项目只是一个开源实现，其实现和苹果实现大部分一致，所以可参考性很强，但并不是完全一致。**

## 6. 关于GCD

在RunLoop的源代码中可以看到用到了GCD的相关内容，但是RunLoop本身和GCD并没有直接的关系。

当调用了dispatch_async(dispatch_get_main_queue(), <#^(void)block#>)时libDispatch会向主线程RunLoop发送消息唤醒RunLoop，RunLoop从消息中获取block，并且在__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__回调里执行这个block。

不过这个操作仅限于主线程，其他线程dispatch操作是全部由libDispatch驱动的。

## 7. 关于网络请求

iOS 中，关于网络请求的接口自下至上有如下几层:

```
CFSocket
CFNetwork       ->ASIHttpRequest
NSURLConnection ->AFNetworking
NSURLSession    ->AFNetworking2, Alamofire
```

• CFSocket 是最底层的接口，只负责 socket 通信。
• CFNetwork 是基于 CFSocket 等接口的上层封装，ASIHttpRequest 工作于这一层。
• NSURLConnection 是基于 CFNetwork 的更高层的封装，提供面向对象的接口，AFNetworking 工作于这一层。
• NSURLSession 是 iOS7 中新增的接口，表面上是和 NSURLConnection 并列的，但底层仍然用到了 NSURLConnection 的部分功能 (比如 com.apple.NSURLConnectionLoader 线程)，AFNetworking2 和 Alamofire 工作于这一层。

下面主要介绍下 NSURLConnection 的工作过程。

通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookieStorage 是处理各种 Cookie 的。

当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-24-031908.png" alt="RunLoop_network" style="zoom:50%;" />

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。