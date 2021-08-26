# 使用KSCrash进行崩溃日志的采集

1. KSCrash的功能特性
2. KSCrash的日志处理
3. KSCrash的集成扩展

## 1.KSCrash的功能特性

我挑选了几个重要的功能

### a.支持在设备上进行离线符号化的工作

和PLCrashReporter中的PLCrashReporterSymbolicationStrategyAll枚举值类似，提供了一种本地符号化的功能。大多数平台的日志解析都需要我们上传对应的符号表文件，用于日志的符号化。但是后台无法提供Mac机或者个人开发者的条件受限，其实可以暂时使用这种方式直接得到解析过后的日志。

在后面也有提到此种方式会是

> On-device symbolication requires basic symbols to be present in the final build. To enable this, go to your app's build settings and set Strip Style to Debugging Symbols. Doing so increases your final binary size by about 5%, but you get on-device symbolication.

开启设备符号化需要在最终版本中包含基本符号，所以要在build settings 中设置 **Strip Style**为**Debugging Symbols**。也会造成**最终的二进制文件大小增加5%左右**,这也是之前PLCrashReporter中提到的，不过当时查到的数据是30-50%，确实测试后没有如此大的差距，也算是解了疑惑，由于打包包含了基本符号表导致的二进制大小增加。

> *但是得到的行号还是可能有误的，如果需要具体的行号，还是需要dsym的解析*

### b.跟踪未被捕获的C ++异常的真实原因。

通常，如果你的应用程序由于未被捕获的C ++异常而终止，那么得到的只是

```css
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem_kernel.dylib          0x9750ea6a 0x974fa000 + 84586 (__pthread_kill + 10)
1   libsystem_sim_c.dylib           0x04d56578 0x4d0f000 + 292216 (abort + 137)
2   libc++abi.dylib                 0x04ed6f78 0x4ed4000 + 12152 (abort_message + 102)
3   libc++abi.dylib                 0x04ed4a20 0x4ed4000 + 2592 (_ZL17default_terminatev + 29)
4   libobjc.A.dylib                 0x013110d0 0x130b000 + 24784 (_ZL15_objc_terminatev + 109)
5   libc++abi.dylib                 0x04ed4a60 0x4ed4000 + 2656 (_ZL19safe_handler_callerPFvvE + 8)
6   libc++abi.dylib                 0x04ed4ac8 0x4ed4000 + 2760 (_ZSt9terminatev + 18)
7   libc++abi.dylib                 0x04ed5c48 0x4ed4000 + 7240 (__cxa_rethrow + 77)
8   libobjc.A.dylib                 0x01310fb8 0x130b000 + 24504 (objc_exception_rethrow + 42)
9   CoreFoundation                  0x01f2af98 0x1ef9000 + 204696 (CFRunLoopRunSpecific + 360)
...
```

无法追踪异常情况到底从何处抛出
 使用KSCrash，可以获得未捕获的异常类型，描述以及从何处引发:

```php
Application Specific Information:
*** Terminating app due to uncaught exception 'MyException', reason: 'Something bad happened...'

Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   Crash-Tester                    0x0000ad80 0x1000 + 40320 (-[Crasher throwUncaughtCPPException] + 0)
1   Crash-Tester                    0x0000842e 0x1000 + 29742 (__32-[AppDelegate(UI) crashCommands]_block_invoke343 + 78)
2   Crash-Tester                    0x00009523 0x1000 + 34083 (-[CommandEntry executeWithViewController:] + 67)
3   Crash-Tester                    0x00009c0a 0x1000 + 35850 (-[CommandTVC tableView:didSelectRowAtIndexPath:] + 154)
4   UIKit                           0x0016f285 0xb4000 + 766597 (-[UITableView _selectRowAtIndexPath:animated:scrollPosition:notifyDelegate:] + 1194)
5   UIKit                           0x0016f4ed 0xb4000 + 767213 (-[UITableView _userSelectRowAtPendingSelectionIndexPath:] + 201)
6   Foundation                      0x00b795b3 0xb6e000 + 46515 (__NSFireDelayedPerform + 380)
7   CoreFoundation                  0x01f45376 0x1efa000 + 308086 (__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ + 22)
8   CoreFoundation                  0x01f44e06 0x1efa000 + 306694 (__CFRunLoopDoTimer + 534)
9   CoreFoundation                  0x01f2ca82 0x1efa000 + 207490 (__CFRunLoopRun + 1810)
10  CoreFoundation                  0x01f2bf44 0x1efa000 + 204612 (CFRunLoopRunSpecific + 276)
...
```

### c.死锁检测

这是一个不稳定的功能。
 如果你的主线程死锁，你的用户界面将无响应，用户将不得不手动关闭该应用程序（为此，不会有崩溃报告）。启用死锁检测后，会建立一个看门狗定时器。如果任何东西的主线程持续时间超过了看门狗定时器的持续时间，KSCrash将关闭该应用程序，并给出堆栈跟踪，显示当前主线程正在执行的操作。

这很好，但你必须小心：应用初始化通常发生在主线程上。如果你的初始化代码比看门狗定时器花费的时间更长，您的应用程序将在启动过程中被强制关闭！

### d.自定义崩溃处理代码（KSCrash.h中的onCrash）

如果你想在发生崩溃后执行一些额外的处理（可能会向报告中添加更多上下文数据），则可以这样做。

但是，您必须确保只使用异步安全的代码，并且最重要的是永远不要从该方法调用Objective-C代码！但是某些类别的崩溃会忽略这种警告的处理程序代码会导致崩溃处理程序崩溃！

### e.支持的崩溃类型

- Mach内核异常
- signals异常
- C ++异常
- Objective-C异常
- 主线程死锁（实验）
- 自定义崩溃

### f.可插拔的后台报告架构

支持Hockey、QuincyKit、Victory、Email四种日志发送方式，也可以增加自己的发送接口。前三者可能大家几乎不用到，邮件方式或者发送到自己后台才是大多数人选择的方案。

### g.KSCrash日志重定向

通过一个kRedirectConsoleLogToDefaultFile宏定义控制，调用`redirectConsoleLogsToDefaultFile`接口，可以将打印到控制台的日志信息写入文件。 默认会写到**/Library/Caches/KSCrashReports/**目录。

## 2.KSCrash的日志处理

### crash堆栈临时信息

同PLCrashReporter一样，KSCrash在崩溃产生的时候并没有直接生成可读的crash日志，而是会在/Library/Caches/KSCrashReports/Simple-Example目录下生成一个带有reporterID的json文件，保存的当前crash的堆栈信息。

### crash日志生成

根据reporterID得到json文件，使用`- (NSString*) toAppleFormat:(NSDictionary*) JSONReport;`进行日志的解析和拼接成string格式。

### crash日志发送

根据可插拔式的发送服务来进行日志的发送，不同的插件发往不同的服务端。调用`kscrash_i_callCompletion(self.onCompletion, self.reports, success, nil);`这个block，将string格式的日志文件转为NSData后进行发送，并且会回调到调用层

```objc
    [installation sendAllReportsWithCompletion:^(NSArray* reports, BOOL completed, NSError* error)
     {
         if(completed)
         {
             NSLog(@"Sent %d reports", (int)[reports count]);
         }
         else
         {
             NSLog(@"Failed to send reports: %@", error);
         }
     }];
```

### 发送策略

内部默认根据本地保存的所有crash.json文件进行发送。所有删除策略决定每次发送的日志。提供了三种模式，

```cpp
// 从不删除
KSCDeleteNever,
// 成功后删除
KSCDeleteOnSucess,
// 每次删除
KSCDeleteAlways
```

一般情况下 我们可以选择KSCDeleteOnSucess。

## 3.KSCrash的集成扩展

1. 基本的集成步骤demo中写的很明白，主要是创建一个`installation`对象，根据不同的发送渠道，然后执行`install`方法。
2. `addConditionalAlertWithTitle`会显示一个提示窗，在点击sure后，KSCrash会读取本地的日志文件，再进行发送。「所以 如果我们需要隐式的，偷偷地发送日志，需要去源码中删除这个提示，自己进行发送的处理。不过后来，找到了更合适的方法。」



```objc
KSCrashInstallationHockey* installation = [KSCrashInstallationHockey sharedInstance];
installation.appIdentifier = @"PUT_YOUR_HOCKEY_APP_ID_HERE";
[installation setReportStyle:KSCrashEmailReportStyleApple useDefaultFilenameFormat:YES]; 

// Optional: Add an alert confirmation (recommended for email installation)
[installation addConditionalAlertWithTitle:@"Crash Detected"
                                 message:@"The app crashed last time it was launched. Send a crash report?"
                               yesAnswer:@"Sure!"
                                noAnswer:@"No thanks"];

[installation install];
```

### 常规的日志发送

1.除了email形式也许可以给个人开发者使用，其余的都不满足我们的日志发送需求。那可以使用提供的`KSCrashInstallationStandard`对象。

他拥有一个url属性，KSCrash会自动向这个url发送日志的json格式文件。



```objc
@property(nonatomic,readwrite,retain) NSURL* url;
```

关于request的创建和发送的内容都在`KSCrashReportSinkStandard`中。

2.此外如果需要增加发送完成的回调设置，可以在`sendAllReportsWithCompletion`接口中设置。

```objc
    KSCrashInstallation* installation = [self makeStandardInstallation];
    
    [installation install];
    // 设置发送成功后删除
    [KSCrash sharedInstance].deleteBehaviorAfterSendAll = KSCDeleteOnSucess;
    
    [installation sendAllReportsWithCompletion:^(NSArray* reports, BOOL completed, NSError* error)
     {

     }];
```

3.如果这样还是满足不了你的需求，报文格式、加解密这些步骤需要增加，那就要去request的创建环节进行修改了。

```objc
- (void) filterReports:(NSArray*) reports
          onCompletion:(KSCrashReportFilterCompletion) onCompletion
{
    self.reports = reports;
    self.onCompletion = onCompletion;

    self.reachableOperation = [KSReachableOperationKSCrash operationWithHost:[self.url host]
                                                                   allowWWAN:YES
                                                                       block:^
    {
      
    }];
}
```

这样整个KSCrash的日志存放、日志读取、日志发送和自定义扩展都能大概有个了解了。感兴趣的可以查看源码，了解各个发送渠道、日志读取处理的完整操作，进一步关于crash日志的生成和解析。

