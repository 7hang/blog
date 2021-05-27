# RunLoop（三）运行

在之前的篇章中，对`RunLoop`有了基本认识，以及对其底层的对象也有了一定了解。

这篇描述的是”**RunLoop是如何运行的？**‘’

为了解决这个问题，我们还是直接上源码，并给出导读的路径，如下：

★ Core Foundation [下载地址](https://opensource.apple.com/tarballs/CF/)

★ CFRounLoop.c

> __CFRunLoopRun
>
>  >>>CFRunLoopRunSpecific
>
>  >>>CFRunLoopRun

# 一、运行逻辑

![image-20181224075744930](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-235745.png)

上图左侧运行逻辑图出自于[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)。

可以看到，在RunLoop中，接收输入事件来自两种不同的来源：输入源（**input source**）和定时源（**timer source**）。

## 1. 来源按同步异步分类

上图描述了按两种来源：异步方式接收的输入源，以及同步接收的定时源。

### 1.1 Input sources

**输入源**传递异步事件，通常消息来自于其他线程或程序，按照是否来源于内核也分为下面几种：

- Port-Based Sources，基于 Port的 事件，系统底层的，一般由**内核自动发出信号**。例如 `CFSocketRef` ，在应用层基本用不到。
- Custom Input Sources，非基于Port事件，用户手动创建的 Source，则必须**从其他线程手动发送信号**。
- Cocoa Perform Selector Sources， Cocoa 提供的 `performSelector` 系列方法，也是一种事件源。和基于端口的源一样，执行`selector`请求会在目标线程上序列化，减缓许多在线程上允许多个方法容易引起的同步问题。不像基于端口的源，一个selector执行完后会自动从`Run Loop`里面移除。

### 1.2 Timer sources

**定时源**则传递同步事件，发生在特定时间或者重复的时间间隔。

定时器可以产生基于时间的通知，但它并不是**实时机制**。和输入源一样，定时器也和你的`Run Loop`的特定模式相关。如果定时器所在的模式当前未被`Run Loop`监视，那么定时器将不会开始直到`Run Loop`运行在相应的模式下。

其主要包含了两部分：

- NSTimer
- performSelector:withObject:afterDelay:

## 2. 来源按对象分类

在上一篇中，我们就是按对象将`RunLoop`接收事件按对象分类的。

### 2.1 Source1

对应于Port-Based Sources，即基于Port的，通过内核和其他线程通信。

常用于接收、分发系统事件，大部分屏幕交互事件都是由`Source1`接收，包装成`Event`，然后分发下去，最后由`Source0`去处理。

所以，其包括：

- 基于Port的线程间通信；
- 系统事件捕捉；

### 2.2 Source0

是非Port事件。在应用中，触摸事件的最终处理，以及`perforSelector:onThread`都是包装成该类型对象，最后由开发者指定回调函数，手动处理该事件。

需要注意的是`perforSelector:onThread`是否有`delay`，即是否延迟函数或者定时函数等类型。

- `perforSelector:onThread` 不是`delay`函数时， 是`Source0`事件。
- `performSelector:withObject:afterDelay`有`delay`时，则属于`Timers`事件。

所以，其包括：

- 触摸事件处理
- performSelector:onThread

### 2.3 Timers

同上[Timer sources](#Timer sources)说明。

# 二、源码详解

## 1. 入口函数

```objective-c
void CFRunLoopRun(void) {
    int32_t result;
    do {
        // 调用RunLoop执行函数
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```

## 2. RunLoop执行函数

```c++
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {
    // RunLoop正在释放，完成返回
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据modeName 取出当前的运行Mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果mode里没有source/timer/observer, 直接返回。
    int32_t result = kCFRunLoopRunFinished;
    
    if (currentMode->_observerMask & kCFRunLoopEntry )
        // 1. 通知 Observers: 进入RunLoop。
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
        // 2---11，RunLoop的运行循环的核心代码
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    
    if (currentMode->_observerMask & kCFRunLoopExit )
        // 12. 通知 Observers: 退出RunLoop
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    rl->_currentMode = previousMode;
    return result;
}
```

## 3. RunLoop消息处理函数

```c++
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    
    int32_t retVal = 0;
    do {
        if (rlm->_observerMask & kCFRunLoopBeforeTimers)
            // 2. 通知 Observers: RunLoop 即将处理 Timer 回调。
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources)
            // 3. 通知 Observers: RunLoop 即将处理 Source
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        // 4. 处理Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        // 5. 处理 Source0 (非port) 回调(可能再次处理Blocks)
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            // 处理Blocks
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            // 6. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
        }
        
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting))
            // 7. 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        __CFRunLoopSetSleeping(rl);
        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();
        do {
            msg = (mach_msg_header_t *)msg_buffer;
            // 8. RunLoop开始休眠：等待消息唤醒，调用 mach_msg 等待接收 mach_port 的消息。
            // • 一个基于 port 的Source 的事件。
            // • 一个 Timer 到时间了
            // • RunLoop 自身的超时时间到了
            // • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
        } while (1);

        __CFRunLoopUnsetSleeping(rl);
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting))
            // 9. 通知 Observers: RunLoop 结束休眠（被某个消息唤醒）
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
    handle_msg:;
        // 收到消息，处理消息。
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
        } else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // 9.1 处理Timer：如果一个 Timer 到时间了，触发这个Timer的回调。
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                __CFArmNextTimerInMode(rlm, rl);
            }
        } else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
            // 9.2 处理GCD Async To Main Queue：如果有dispatch到main_queue的block，执行block。
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
                mach_msg_header_t *reply = NULL;
                // 9.3 处理Source1：如果一个 Source1 (基于port) 发出事件了，处理这个事件
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            }
        }
        // 10. 处理Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        // 11. 根据前面的处理结果，决定流程
        // 11.1 当下面情况发生时，退出RunLoop
        if (sourceHandledThisLoop && stopAfterHandle) {
            retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            // 11.1.1 超出传入参数标记的超时时间了
            retVal = kCFRunLoopRunTimedOut;
        } else if (__CFRunLoopIsStopped(rl)) {
            // 11.1.2 当前RunLoop已经被外部调用者强制停止了
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            // 11.1.3 当前运行模式已经被停止
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            // 11.1.4 source/timer/observer一个都没有了
            retVal = kCFRunLoopRunFinished;
        }
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
        // 11.2 如果没超时，mode里不为空也没停止，loop也没被停止，那继续loop。
    } while (0 == retVal);
    
    return retVal;
}
```

## 4. 消息处理底层函数

当 `RunLoop` 进行回调时，一般都是通过一个很长的函数调用出去 (call out)，当你在你的代码中下断点调试时，打印堆栈（bt），就能在调用栈上看到这些函数。

下面是这几个函数的整理版本，如果你在调用栈中看到这些长函数名，在这里查找一下就能定位到具体的调用地点了：

```objective-c
{
    // 1. 通知 Observers: 进入RunLoop。
    // 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
 
    // 2. 通知 Observers: RunLoop 即将处理 Timer 回调。   
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
   
    // 3. 通知 Observers: RunLoop 即将处理 Source
 __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
    // 4. 处理Blocks
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

    // 5. 处理 Source0 (非port) 回调(可能再次处理Blocks)
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

    // 7. 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
    /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
 __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);

    // 8. RunLoop开始休眠：等待消息唤醒，调用 mach_msg 等待接收 mach_port 的消息。
    mach_msg() -> mach_msg_trap();


    // 9. 通知 Observers: RunLoop 结束休眠（被某个消息唤醒）
 __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);

    // 9.1 处理Timer：如果一个 Timer 到时间了，触发这个Timer的回调。
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);

    // 9.2 处理GCD Async To Main Queue：如果有dispatch到main_queue的block，执行block。
    __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);

    // 9.3 处理Source1：如果一个 Source1 (基于port) 发出事件了，处理这个事件
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
    // 10. 处理Blocks
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
    
    // 12. 通知 Observers: 退出RunLoop
    // 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

## 5. 休眠的实现

RunLoop如何实现真正意义上的休眠，而不是像下面这种方式：

```
//会一直占用线程，会一直执行while(1);
while(1);
```

其内部是通过内核态的调用。

![image-20181224103714294](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-24-023715.png)

# 三、流程图

将上面的代码逻辑抽取下面得到如下：

- 黄色：表示通知Observer各个阶段；
- 蓝色：处理消息的逻辑；
- 绿色：分支判断逻辑；

## 1. 执行逻辑

![image-20181224101703327](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-24-021703.png)

## 2. 流程图

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-24-021730.png" alt="RunLoop" style="zoom: 33%;" />