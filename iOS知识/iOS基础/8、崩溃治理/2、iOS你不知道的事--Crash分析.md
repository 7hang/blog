# Crash分析

大家平时在开发过程中，经常会遇到`Crash`，那也是在正常不过的事，但是作为一个优秀的iOS开发人员，必将这些用户不良体验降到最低。

- 线下Crash,我们直接可以调试，结合stack信息，不难定位！
- 线上Crash当然也有一些信息，毕竟苹果爸爸的产品还是做得非常不错的！

![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbe63882667ab)

通过iPhone的`Crash log`也可以分析一些，但是这个是需要用户配合的,因为需要用户在手机 中 `设置-> 诊断与用量->勾选 自动发送 ,然后在xcode中 Window->Organizer->Crashes 对应的app`,就是当前app最新一版本的`crash log` ,并且是解析过的,可以根据`crash 栈` 等相关信息 ,尤其是程序代码级别的 有超链接,一键可以直接跳转到程序崩溃的相关代码,这样更容易定位bug出处.

为了能够第一时间发现程序问题，应用程序需要实现自己的崩溃日志收集服务，成熟的开源项目很多，如 [KSCrash](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Fgithub.com%2Fkstenerud%2FKSCrash)，[plcrashreporter](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Fgithub.com%2Fplausiblelabs%2Fplcrashreporter)，[CrashKit](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Fgithub.com%2Fkaler%2FCrashKit) 等。追求方便省心，对于保密性要求不高的程序来说，也可以选择各种一条龙Crash统计产品，如 [Crashlytics](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttp%3A%2F%2Ftry.crashlytics.com%2F)，[Hockeyapp](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttp%3A%2F%2Fhockeyapp.net%2Ffeatures%2Fcrashreports%2F) ，[友盟](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttp%3A%2F%2Fwww.umeng.com%2Fumeng30_error_type)，[Bugly](https://link.juejin.cn/?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttp%3A%2F%2Fbugly.qq.com%2F) 等等

> 但是，所有的但是，这不够！因为我们不再是一个简单会用的iOS开发人员，必将走向底层，了解原理，掌握装逼内容和技巧是我们的必修课

## 首先我们来了解一下`Crash`的底层原理

iOS系统自带的 `Apple’s Crash Reporter`记录在设备中的Crash日志，`Exception Type`项通常会包含两个元素：`Mach异常`和`Unix信号`。

```
Exception Type:         EXC_BAD_ACCESS (SIGSEGV)    
Exception Subtype:      KERN_INVALID_ADDRESS at 0x041a6f3
复制代码
```

Mach异常是什么？它又是如何与Unix信号建立联系的？

`Mach`是一个XNU的微内核核心，`Mach`异常是指最底层的内核级异常，被定义在下 。每个`thread，task，host`都有一个异常端口数组，`Mach`的部分`API`暴露给了用户态，用户态的开发者可以直接通过`Mach API`设置`thread，task，host`的异常端口，来捕获`Mach`异常，抓取`Crash`事件。

所有`Mach`异常都在`host`层被`ux_exception`转换为相应的`Unix信号`，并通过`threadsignal`将信号投递到出错的线程。iOS中的 `POSIX API`就是通过`Mach`之上的 `BSD`层实现的。

![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbec07ddf3984)



因此，`EXC_BAD_ACCESS (SIGSEGV)`表示的意思是：`Mach`层的`EXC_BAD_ACCESS`异常，在`host`层被转换成`SIGSEGV信号`投递到出错的线程。

## iOS的异常Crash

- KVO问题

- NSNotification线程问题

- 数组越界

- 野指针

- 后台任务超时

- 内存爆出

- 主线程卡顿超阀值

- 死锁

  ....

  下面我就拿出最常见的两种`Crash`分析一下

- `Exception`

![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbee0367fd5a2)

- `Signal`

![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbee423dd1ad3)



## `Crash`分析处理

**上面我们也知道：既然最终以信号的方式投递到出错的线程，那么就可以通过注册相应函数来捕获信号.到达`Hook`的效果**

```
+ (void)installUncaughtSignalExceptionHandler{
    NSSetUncaughtExceptionHandler(&LGExceptionHandlers);
    signal(SIGABRT, LGSignalHandler);
}
复制代码
```

[关于Signal参考](https://user-gold-cdn.xitu.io/2019/7/4/16bbbf27f1c3df97)

我们从上面的函数可以Hook到信息，下面我们开始进行包装处理.这里还是面向统一封装，因为等会我们还需要考虑`Signal`

```
void LGExceptionHandlers(NSException *exception) {
    NSLog(@"%s",__func__);
    
    NSArray *callStack = [LGUncaughtExceptionHandle lg_backtrace];
    NSMutableDictionary *mDict = [NSMutableDictionary dictionaryWithDictionary:exception.userInfo];
    [mDict setObject:callStack forKey:LGUncaughtExceptionHandlerAddressesKey];
    [mDict setObject:exception.callStackSymbols forKey:LGUncaughtExceptionHandlerCallStackSymbolsKey];
    [mDict setObject:@"LGException" forKey:LGUncaughtExceptionHandlerFileKey];
    
    // exception - myException

    [[[LGUncaughtExceptionHandle alloc] init] performSelectorOnMainThread:@selector(lg_handleException:) withObject:[NSException exceptionWithName:[exception name] reason:[exception reason] userInfo:mDict] waitUntilDone:YES];
}
复制代码
```

下面针对封装好的`myException`进行处理,在这里要做两件事

- 存储，上传：方便开发人员检查修复
- 处理Crash奔溃，我们也不能眼睁睁看着`BUG`闪退在用户的手机上面，希望“起死回生，回光返照”

```
- (void)lg_handleException:(NSException *)exception{
    // crash 处理
    // 存
    NSDictionary *userInfo = [exception userInfo];
    [self saveCrash:exception file:[userInfo objectForKey:LGUncaughtExceptionHandlerFileKey]];
}

复制代码
```

下面是一些封装的一些辅助函数

- 保存奔溃信息或者上传：针对封装数据本地存储，和相应上传服务器！
- 

```
- (void)saveCrash:(NSException *)exception file:(NSString *)file{
    
    NSArray *stackArray = [[exception userInfo] objectForKey:LGUncaughtExceptionHandlerCallStackSymbolsKey];// 异常的堆栈信息
    NSString *reason = [exception reason];// 出现异常的原因
    NSString *name = [exception name];// 异常名称
    
    // 或者直接用代码，输入这个崩溃信息，以便在console中进一步分析错误原因
    // NSLog(@"crash: %@", exception);
    
    NSString * _libPath  = [[NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) objectAtIndex:0] stringByAppendingPathComponent:file];

    if (![[NSFileManager defaultManager] fileExistsAtPath:_libPath]){
        [[NSFileManager defaultManager] createDirectoryAtPath:_libPath withIntermediateDirectories:YES attributes:nil error:nil];
    }
    
    NSDate *dat = [NSDate dateWithTimeIntervalSinceNow:0];
    NSTimeInterval a=[dat timeIntervalSince1970];
    NSString *timeString = [NSString stringWithFormat:@"%f", a];
    
    NSString * savePath = [_libPath stringByAppendingFormat:@"/error%@.log",timeString];
    
    NSString *exceptionInfo = [NSString stringWithFormat:@"Exception reason：%@\nException name：%@\nException stack：%@",name, reason, stackArray];
    
    BOOL sucess = [exceptionInfo writeToFile:savePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
    
    NSLog(@"保存崩溃日志 sucess:%d,%@",sucess,savePath);
}
复制代码
```

- 获取函数堆栈信息，这里可以获取响应调用堆栈的符号信息，通过数组回传

```
+ (NSArray *)lg_backtrace{
    
    void* callstack[128];
    int frames = backtrace(callstack, 128);//用于获取当前线程的函数调用堆栈，返回实际获取的指针个数
    char **strs = backtrace_symbols(callstack, frames);//从backtrace函数获取的信息转化为一个字符串数组
    int i;
    NSMutableArray *backtrace = [NSMutableArray arrayWithCapacity:frames];
    for (i = LGUncaughtExceptionHandlerSkipAddressCount;
         i < LGUncaughtExceptionHandlerSkipAddressCount+LGUncaughtExceptionHandlerReportAddressCount;
         i++)
    {
        [backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
    }
    free(strs);
    return backtrace;
}

复制代码
```

- 获取应用信息，这个函数提供给Siganl数据封装

```
NSString *getAppInfo(){
    NSString *appInfo = [NSString stringWithFormat:@"App : %@ %@(%@)\nDevice : %@\nOS Version : %@ %@\n",
                         [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleDisplayName"],
                         [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"],
                         [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleVersion"],
                         [UIDevice currentDevice].model,
                         [UIDevice currentDevice].systemName,
                         [UIDevice currentDevice].systemVersion];
    //                         [UIDevice currentDevice].uniqueIdentifier];
    NSLog(@"Crash!!!! %@", appInfo);
    return appInfo;
}
复制代码
```

做完这些准备，你可以非常清晰的看到程序奔溃，哈哈哈！（好像以前奔溃还不清晰似的），这里说一下：我的意思你非常清晰的知道奔溃之前做了一些什么! 下面是检测我们奔溃之前的沙盒存储的信息：`error.log`



![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbf4cc6e8b559)



下面我们来一个骚操作：在监听的信息的时候来了一个`Runloop`,我们监听所有的`mode`,开启循环(**一个相对于我们应用程序自启的`Runloop`的平行空**）

```
SCLAlertView *alert = [[SCLAlertView alloc] initWithNewWindowWidth:300.0f];
[alert addButton:@"奔溃" actionBlock:^{
    self.dismissed = YES;
}];
[alert showSuccess:exception.name subTitle:exception.reason closeButtonTitle:nil duration:0];
// 本次异常处理
CFRunLoopRef runloop = CFRunLoopGetCurrent();
CFArrayRef   allMode = CFRunLoopCopyAllModes(runloop);
while (!self.dismissed) {
    // machO
    // 后台更新 - log
    // kill
    // 
    for (NSString *mode in (__bridge NSArray *)allMode) {
        CFRunLoopRunInMode((CFStringRef)mode, 0.0001, false);
    }
}

CFRelease(allMode);
复制代码
```

在这个`平行空间`我们开启一个弹框，这个弹框，跟着我们的应用程序保活，并且具备相应的响应能力，到目前为止：此时此刻还有谁！这不就是`回光返照`？只要我们的条件成立，那么在相应的这个`平行空间`继续做一些我们的工作，程序不死:`what is dead may never die,but rises again harder and stronger`

## signal 函数拦截不到的解决方式

在debug模式下，如果你触发了崩溃，那么应用会直接崩溃到主函数，断点都没用，此时没有任何log信息显示出来，如果你想看log信息的话，你需要在`lldb`中，拿`SIGABRT`来说吧，敲入`pro hand -p true -s false SIGABRT`命令，不然你啥也看不到。



![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbf7839bde9d0)



然后断开断点，程序进入监听，下面剩下的操作就是包装异常，操作类似`Exception`

![img](https://user-gold-cdn.xitu.io/2019/7/4/16bbbf7bb9227aef)



最后我们需要注意的针对我们的监听回收相应内存：



```
   NSSetUncaughtExceptionHandler(NULL);
    signal(SIGABRT, SIG_DFL);
    signal(SIGILL, SIG_DFL);
    signal(SIGSEGV, SIG_DFL);
    signal(SIGFPE, SIG_DFL);
    signal(SIGBUS, SIG_DFL);
    signal(SIGPIPE, SIG_DFL);

    if ([[exception name] isEqual:UncaughtExceptionHandlerSignalExceptionName])
    {
        kill(getpid(), [[[exception userInfo] objectForKey:UncaughtExceptionHandlerSignalKey] intValue]);
    }
    else
    {
        [exception raise];
    }
复制代码
```

到目前为止，我们响应的`Crash`处理已经入门，如果你还想继续探索也是有很多地方比如：

[sdsdsdssd](https://link.juejin.cn/?target=http%3A%2F%2Fwww.baidu.com)

- 我们能否hook系统奔溃，异常的方法`NSSetUncaughtExceptionHandler`,已达到拒绝传递 `UncaughtExceptionHandler`的效果
- 我们在处理异常的时候，利用`Runloop回光返照`，有没有更加合适的方法
- `Runloop回光返照`我们怎么继续保证应用程序稳定执行