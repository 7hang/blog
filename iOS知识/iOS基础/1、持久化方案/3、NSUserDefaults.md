# NSUserDefaults

## **前言** 

字节团队最近分享的 [**iOS 稳定性问题治理：卡死崩溃监控原理及最佳实践**](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488080&idx=1&sn=39d0386b97b9ac06c6af1966f48387fc&chksm=e9d0d9b2dea750a4a7d21fd383aefa014d63f0dc79f2e3a13c97ad52bba1578dca8b50d6a40a&scene=21&cur_album_id=1590407423234719749#wechat_redirect) 提到：**`NSUserDefaults` 底层实现中存在直接或者间接的跨进程通信，在主线程同步调用容易发生卡死。**

随之而来的问题就是：**`NSUserDefaults` 还能用吗？**

经过对底层分析后，笔者的研究结论是：**可以在理解 `NSUserDefaults` 的特性后再使用**。

## **一、`NSUserDefaults` 是什么？** 

`NSUserDefaults` 是 iOS 开发者常用的持久化工具，通常用于存储少量的数据

示例:

```javascript
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
[defaults setObject:@"酷酷的哀殿" forKey:@"key"];
```

## **二、`NSUserDefaults` 的特性是什么？** 

根据本文后续的测试，我们可以发现 `NSUserDefaults` 共计以下 3 个特性：

1. 多线程安全
2. 内存级别缓存
3. 写操作会触发 `xpc` 通信

## **三、`NSUserDefaults` 是如何保证多线程安全的？** 

`NSUserDefaults` 内部在读写时，会通过锁 `lock` 保证读写安全

> 可以通过 `b os_unfair_lock_lock` 设置断点

![urozvs19vo](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/urozvs19vo.jpeg)

## **四、`NSUserDefaults` 的性能怎么样？** 

虽然 `NSUserDefaults` 是磁盘持久化存储，但是因为缓存的存在，所以，不会频繁的进行 磁盘 I/O

> 可以通过私有类 `CFPrefsPlistSource` 的实例获取所有缓存的内容

![](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/v33hajsrfs.jpeg)

我们唯一需要考虑的因素**避免数据过多导致内存压力过大**

## **三、`NSUserDefaults` 触发 `xpc` 的场景是什么？** 

`NSUserDefaults` 与 [**如何监控 iOS 的启动耗时**](https://mp.weixin.qq.com/s?__biz=MzAxMzk0OTg5MQ==&mid=2247484699&idx=1&sn=89907f685ee21ccc378203a40fa40e8f&chksm=9b9b8bb7acec02a13901493fdabee514d6d6972459b4e51bbec933da122c4dbb2bab2c71cf94&token=267246986&lang=zh_CN&scene=21#wechat_redirect) 提到的渲染过程类似，同样依赖 **xpc** 进行跨进程通信。

下面，我们通过添加合适的断点对相关流程进行简单的介绍

### **添加调试断点**

```javascript
(lldb) breakpoint set -n xpc_connection_create_mach_service -C "x/s $x0" -C "p $x1" -C "p $x2"
Breakpoint 6: where = libxpc.dylib`xpc_connection_create_mach_service, address = 0x00000001d3d4e944
(lldb) breakpoint set -n xpc_connection_send_message_with_reply_sync -C "po $x0" -C "po $x1" -C "bt"
Breakpoint 7: where = libxpc.dylib`xpc_connection_send_message_with_reply_sync, address = 0x00000001d3d4f028
```

> xpc_connection_send_message_with_reply_sync 会锁住当前线程

### **准备测试代码**

```javascript
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
[defaults setObject:@"酷酷的哀殿1" forKey:@"key"];   //xpc_connection_send_message_with_reply_sync
[defaults setObject:@"酷酷的哀殿2" forKey:@"key"];   //xpc_connection_send_message_with_reply_sync
printf("从 NSUserDefaults 读取：%s\n", [defaults stringForKey:@"key"].UTF8String);
[defaults synchronize];
NSUserDefaults *domain = [[NSUserDefaults alloc] initWithSuiteName:@"someDomain"];
[domain setObject:@"酷酷的哀殿3" forKey:@"key"];     // xpc_connection_send_message_with_reply_sync
[domain setObject:@"酷酷的哀殿4" forKey:@"key"];     // xpc_connection_send_message_with_reply_sync
printf("从 NSUserDefaults 读取：%s\n", [domain stringForKey:@"key"].UTF8String);
[domain synchronize];
```

### **测试**

通过运行测试代码，我们可以发现 `+[NSUserDefaults(NSUserDefaults) standardUserDefaults] + 68` 执行时，会创建名为 `"com.apple.cfprefsd.daemon"` 的 `xpc_connection`

![](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/1.jpeg)

随后，会通过 `xpc_connection_send_message_with_reply_sync` 发送一个信息

![](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2.jpeg)

`[defaults setObject:@"酷酷的哀殿1" forKey:@"key"];` 执行时，同样会发送一个消息

![](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/3.jpeg)

经过测试，我们可以发现只有第一次初始化或者调用 `set... forKey:` 相关的方法时，才会触发多进程通信

所以，我们可以得到以下结论：

`NSUserDefaults` 写操作会触发 `xpc` 通信，读操作和 `synchronize` 不会触发；应该**降低写操作频率**

![](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/4.jpeg)

## **四、如何异步的持久化？** 

通过官方文档，我们可以发现 `xpc` 框架存在两个不会锁住当前的线程 API

1. xpc_connection_send_message
2. xpc_connection_send_message_with_reply

所以，我们可以尝试通过以上两个 API 发送持久化信息

### **异步持久化 Demo**

下面以笔者的 `iOS 14.3` 系统为例进行演示

```javascript
    xpc_connection_t conn = xpc_connection_create_mach_service("com.apple.cfprefsd.daemon", NULL, XPC_CONNECTION_MACH_SERVICE_PRIVILEGED);


#pragma mark - 开始构建信息
//    (lldb) po $rsi
//    <OS_xpc_dictionary: dictionary[0x7fa975908010]: { refcnt = 1, xrefcnt = 1, subtype = 0, count = 8 } <dictionary: 0x7fa975908010> { count = 8, transaction: 0, voucher = 0x0, contents =
//        "CFPreferencesHostBundleIdentifier" => <string: 0x7fa9759080d0> { length = 9, contents = "test.demo" }
//        "CFPreferencesUser" => <string: 0x7fa975908250> { length = 25, contents = "kCFPreferencesCurrentUser" }
//        "CFPreferencesOperation" => <int64: 0x8ccdbf87dd7d7a91>: 1
//        "Value" => <string: 0x7fa9759084b0> { length = 16, contents = "ÈÖ∑ÈÖ∑ÁöÑÂìÄÊÆø2" }
//        "Key" => <string: 0x7fa975908430> { length = 3, contents = "key" }
//        "CFPreferencesContainer" => <string: 0x7fa9759083a0> { length = 169, contents = "/private/var/mobile/Containers/Data/Application/0C224166-1674-4D36-9CDB-9FCDB633C7E3/" }
//        "CFPreferencesCurrentApplicationDomain" => <bool: 0x7fff80002fd0>: true
//        "CFPreferencesDomain" => <string: 0x7fa975906ea0> { length = 9, contents = "test.demo" }
//    }>

    xpc_object_t hello = xpc_dictionary_create(NULL, NULL, 0);

    // 注释1：test.demo 是 bundleid。测试代码时，需要根据需要修改
    xpc_dictionary_set_string(hello, "CFPreferencesHostBundleIdentifier", "test.demo");
    xpc_dictionary_set_string(hello, "CFPreferencesUser", "kCFPreferencesCurrentUser");
    // 注释2：存储值
    xpc_dictionary_set_int64(hello, "CFPreferencesOperation", 1);
    // 注释3：存储的内容
    xpc_dictionary_set_string(hello, "Value", "this is a test");
    xpc_dictionary_set_string(hello, "Key", "key");

    // 注释4：存储的位置
    CFURLRef url = CFCopyHomeDirectoryURL();
    const char *container = CFStringGetCStringPtr(CFURLCopyPath(url), kCFStringEncodingASCII);
    xpc_dictionary_set_string(hello, "CFPreferencesContainer", container);
    xpc_dictionary_set_bool(hello, "CFPreferencesCurrentApplicationDomain", true);
    xpc_dictionary_set_string(hello, "CFPreferencesDomain", "test.demo");


    xpc_connection_set_event_handler(conn, ^(xpc_object_t object) {
        printf("xpc_connection_set_event_handler:收到返回消息: %sn", xpc_copy_description(object));
    });
    xpc_connection_resume(conn);
#pragma mark - 异步方案一 （没有回应）
//    xpc_connection_send_message(conn, hello);
#pragma mark - 异步方案二 （有回应）
    xpc_connection_send_message_with_reply(conn, hello, NULL, ^(xpc_object_t  _Nonnull object) {
        printf("xpc_connection_send_message_with_reply:收到返回消息: %sn", xpc_copy_description(object));
        NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults]; // mach_msg_trap
        printf("从 NSUserDefaults 读取：%s\n", [defaults stringForKey:@"key"].UTF8String);
    });
#pragma mark - 同步方案
//    xpc_object_t obj = xpc_connection_send_message_with_reply_sync(conn, hello);
//    NSLog(@"xpc_connection_send_message_with_reply_sync:收到返回消息:%s", xpc_copy_description(obj));
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults]; // mach_msg_trap
    printf("从 NSUserDefaults 读取：%s\n", [defaults stringForKey:@"key"].UTF8String);
```

通过控制台，我们可以发现通过 `xpc` 存储的数据 `this is a test` 可以通过 `NSUserDefaults` 读取出来

证明 `xpc_connection_send_message_with_reply` 可以成功将内容持久化

![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/5.jpeg)



## 五、性能

既然，本质上说，它是个plist文件，那么为什么要用它来存一些程序里要用到的key值，而不直接用一个plist文件来存取呢？因为我们可以容易想见，如果程序的各个类都往一个文件中读写（别忘了，NSUserDefaults是个单例，而且它访问的文件也只有一个），那么当其中的key变多后，性能岂不是要变差？而如果各个类各自读写自己的plist文件，则不会使自己的文件里存了别人的key而导致文件变大、性能变差。

从文件的角度看，确实如此。但是，NSUserDefaults帮我们做了一层优化，NSUserDefaults是带缓存的。NSUserDefaults会把访问到的key缓存到内存里，下次再访问时，如果内存中命中就直接访问，如果未命中再从文件中载入。应用会时不时调用[defaults synchronize]方法来保证内存与文件中的数据的一致性，有时在写入一个值后也最好调用下这个方法来保证数据真正写入文件。

这样的机制带来了一定的性能提升，实测在10万个key值的情况下，通过NSUserDefaults来读取value是1ms级别的，而如果从自己写的plist里读取则是0.1s级别的开销。但是，如果用plist文件自己在应用里写入1个10万条记录的文件是1s级别的开销，而同时写入10万条NSUserDefaults键值对则是10s级别的延迟。原因在于，在创建key/value pair时，要在内存中也创建一个相应的映射，而系统时不时地调用synchronize方法会导致创建key/value pair被阻塞。

总之，读取NSUserDefaults是比较高效的，而程序不同地方对NSUserDefaults的写入操作也不会带来严重性能影响，但是，如果在一个地方大规模写入NSUserDefaults值则会是性能灾难。用plist文件读入内存在访问的瓶颈主要在读入的过程中，如果文件很大会比较耗时，但是一旦载好，在内存中读取就很快；但是，对内存值的写入则会造成内存与文件数据不一致，这时为了保证数据一致性就要写入文件，写入后又要读入内存，这就导致延迟。

## **六、总结** 

本文通过分析 `NSUserDefaults` 的 3 个特性：1、多线程安全，2、内存级别缓存，3、写操作会触发 `xpc` 通信；可以得到以下结论：

只有在以下场景才适合选择 `NSUserDefaults` 作为数据持久化：

1. 少量数据存储
2. 偶尔修改
3. 多线程安全