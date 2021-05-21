# RunLoop（二）对象

在上文，我们了解到`RunLoop`不但表示的是一个事件循环，也是一个对象。

在本文，我们将会在源码的帮助下，窥探`RunLoop`对象，以及其相关对象。

源码的出处：[CF源码](https://opensource.apple.com/tarballs/CF/)、[Swift CF源码](https://github.com/apple/swift-corelibs-foundation/)。

# 一、RunLoop对象

## 1. CFRunLoopRef

获取RunLoop对象可以通过下面获得：

```objective-c
NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
NSRunLoop *mainRunLoop = [NSRunLoop mainRunLoop];
```



但苹果开放的源码，是`Core Foundation`，而以上获取的是`Foundation`。

`NSRunLoop`在`Core Foundation`对应的是`CFRunLoopRef`，源码中`CFRunLoopRef`的对应的结构体是`__CFRunLoop`。

对`__CFRunLoop`作了一些删减，整理如下：

![image-20181224042455523](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-202456.png)

## 2. 获取RunLoop对象

如何获取RunLoop对象的实现如下：

![image-20181224042836872](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-202837.png)

## 3. RunLoop与线程

从上面获取RunLoop的实现里可以看出，`_CFRunLoopGet0`是获取`RunLoop`对象的底层实现函数，其中传入的参数，是一个线程对象。

那么线程与RunLoop的关系又是怎么样的？

```c++
//  全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef __CFRunLoops = NULL;
// 访问__CFRunLoops的锁
static CFLock_t loopsLock = CFLockInit;

// 获取pthread 对应的 RunLoop。
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
        // pthread为空时，获取主线程
        t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        __CFUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        __CFLock(&loopsLock);
    }
    // 从全局字典里获取对应的RunLoop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) {
        // 如果取不到，就创建一个新的RunLoop
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            //设值
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
    }
    return loop;
}
```

读完上面源码，我们整理如下：

![image-20181224043211881](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-203212.png)

# 二、RunLoop相关类

## 1. 相关类

在需要了解更过RunLoop的类时，我们可以通过下面打印，获得足够多的信息：

```
NSLog(@"%@", [NSRunLoop currentRunLoop]);
```

整理输出的打印信息及结构，结合源码。

![image-20181224045206762](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-205207.png)

上图五个类，标记为三种颜色：

- 红色：`RunLoop`运行循环的模式，`CFRunLoopRef`是`RunLoop`对象本身，`CFRunLoopModeRef`是`RunLoop`当前的一个运行模式。
- 蓝色：需要RunLoop处理的消息类型，包括Source和timer。
- 绿色：监听RunLoop运行状态的一个对象。

## 2. 职责

上面五个类的具体职责如下：

![image-20181224044622347](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-204623.png)

上面的蓝色标记的 **Source/Timer/Observer** 被统称为 **mode item**，在后面会详细讲述。

# 三、Mode对象

Mode表示RunLoop的运行模式，其CF对象`CFRunLoopModeRef`。

![image-20181224051043532](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-211043.png)

其中，常见Mode，如下：

![image-20181224051132907](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-211133.png)

## 1. CommonModes

其中，需要着重说明的是，在RunLoop对象中，有一个叫`CommonModes`的概念。

先看`RunLoop`对象的组成：

```c++
struct __CFRunLoop {
    pthread_t _pthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
};
```

一个 Mode 可以将自己标记为”Common”属性，通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中。

那么添加进去之后的作用是什么？

每当 `RunLoop` 的内容发生变化时，`RunLoop` 都会自动将 **_commonModeItems** 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里。其底层实现如下：

```c++
void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef modeName) {
    if (!CFSetContainsValue(rl->_commonModes, modeName)) {
        //获取所有的_commonModeItems
        CFSetRef set = rl->_commonModeItems ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModeItems) : NULL;
        //获取所有的_commonModes
        CFSetAddValue(rl->_commonModes, modeName);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, modeName};
            // 将所有的_commonModeItems逐一添加到_commonModes里的每一个Mode
            CFSetApplyFunction(set, (__CFRunLoopAddItemsToCommonMode), (void *)context);
            CFRelease(set);
        }
    }
}
```

## 2. Mode API

`CFRunLoop`对外暴露的管理 Mode 接口只有下面2个:

```
CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);
CFRunLoopRunInMode(CFStringRef modeName, ...);
```

Mode 暴露的管理 mode item 的接口有下面几个：

```
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```

你只能通过 mode name 来操作内部的 mode，当你传入一个新的 mode name 但 `RunLoop` 内部没有对应 mode 时，`RunLoop`会自动帮你创建对应的 `CFRunLoopModeRef`。对于一个 `RunLoop` 来说，其内部的 mode 只能增加不能删除。

苹果公开提供的 Mode 有两个：`kCFRunLoopDefaultMode` (`NSDefaultRunLoopMode`) 和 `UITrackingRunLoopMode`，你可以用这两个 Mode Name 来操作其对应的 Mode。

同时苹果还提供了一个操作 Common 标记的字符串：`kCFRunLoopCommonModes` (`NSRunLoopCommonModes`)，你可以用这个字符串来操作 Common Items，或标记一个 Mode 为 “Common”。使用时注意区分这个字符串和其他 mode name。

如下：

```objective-c
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

# 四、Mode Item

`RunLoop`需要处理的消息，包括timer以及source消息，它们都属于Mode item。

`RunLoop`也可以被监听，被监听的对象是observer对象，也属于Mode item。

所有的mode item都可以被添加到Mode中，Mode中包含可以包含多个mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 `RunLoop` 会直接退出，不进入循环。

## 1. CFRunLoopSourceRef

![image-20181224053638246](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-213638.png)Source有两个版本：`Source0` 和 `Source1`。

- Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 `RunLoop`，让其处理这个事件。
- `Source1` 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 `RunLoop` 的线程，其原理在下面会讲到。

## 2. CFRunLoopTimerRef

![image-20181224053720661](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-213721.png)

`CFRunLoopTimerRef`是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 `RunLoop` 时，`RunLoop`会注册对应的时间点，当时间点到时，`RunLoop`会被唤醒以执行那个回调。

## 3. CFRunLoopObserverRef

`CFRunLoopObserverRef` 是观察者，每个 Observer 都包含了一个回调（函数指针），当 `RunLoop` 的状态发生变化时，观察者就能通过回调接受到这个变化。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-213814.png" alt="image-20181224053813902" style="zoom:50%;" />

下面是一个添加监听的示例：

```objective-c
// 创建Observer
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
    switch (activity) {
        case kCFRunLoopEntry: {
            CFRunLoopMode mode = CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent());
            NSLog(@"kCFRunLoopEntry - %@", mode);
            CFRelease(mode);
            break;
        }
            
        case kCFRunLoopExit: {
            CFRunLoopMode mode = CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent());
            NSLog(@"kCFRunLoopExit - %@", mode);
            CFRelease(mode);
            break;
        }
        default:
            break;
    }
});
// 添加Observer到RunLoop中
CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
// 释放
CFRelease(observer);
```

# 五、完整的RunLoop对象

打印`RunLoop`，整理如下；

```c++
CFRunLoop {
    current mode = kCFRunLoopDefaultMode
    common modes = {
        UITrackingRunLoopMode
        kCFRunLoopDefaultMode
    }
    common mode items = {
        // source0 (manual)
        CFRunLoopSource {order =-1, {
            callout = _UIApplicationHandleEventQueue}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventSignalCallback }}
        CFRunLoopSource {order = 0, {
            callout = FBSSerialQueueRunLoopSourceHandler}}
 
        // source1 (mach port)
        CFRunLoopSource {order = 0,  {port = 17923}}
        CFRunLoopSource {order = 0,  {port = 12039}}
        CFRunLoopSource {order = 0,  {port = 16647}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventCallback}}
        CFRunLoopSource {order = 0, {port = 2407,
            callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_}}
        CFRunLoopSource {order = 0, {port = 1c03,
            callout = __IOHIDEventSystemClientAvailabilityCallback}}
        CFRunLoopSource {order = 0, {port = 1b03,
            callout = __IOHIDEventSystemClientQueueCallback}}
        CFRunLoopSource {order = 1, {port = 1903,
            callout = __IOMIGMachPortPortCallback}}
 
        // Ovserver
        CFRunLoopObserver {order = -2147483647, activities = 0x1, // Entry
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
        CFRunLoopObserver {order = 0, activities = 0x20,          // BeforeWaiting
            callout = _UIGestureRecognizerUpdateObserver}
        CFRunLoopObserver {order = 1999000, activities = 0xa0,    // BeforeWaiting | Exit
            callout = _afterCACommitHandler}
        CFRunLoopObserver {order = 2000000, activities = 0xa0,    // BeforeWaiting | Exit
            callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
        CFRunLoopObserver {order = 2147483647, activities = 0xa0, // BeforeWaiting | Exit
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
 
        // Timer
        CFRunLoopTimer {firing = No, interval = 3.1536e+09, tolerance = 0,
            next fire date = 453098071 (-4421.76019 @ 96223387169499),
            callout = _ZN2CAL14timer_callbackEP16__CFRunLoopTimerPv (QuartzCore.framework)}
    },
 
    modes ＝ {
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = 0, {
                    callout = FBSSerialQueueRunLoopSourceHandler}}
            },
            sources1 = (null),
            observers = {
                CFRunLoopObserver >{activities = 0xa0, order = 2000000,
                    callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
            )},
            timers = (null),
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventSignalCallback}}
            },
            sources1 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventCallback}}
            },
            observers = (null),
            timers = (null),
        },
        
        CFRunLoopMode  {
            sources0 = (null),
            sources1 = (null),
            observers = (null),
            timers = (null),
        }
    }
}
```

可以看到，系统默认注册了5个Mode:

1. kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。
5. kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。