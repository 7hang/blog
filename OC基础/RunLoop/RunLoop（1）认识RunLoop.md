# RunLoop（一）认识RunLoop

# 一、引子

## 1. 不同的应用程序

我们在开发iOS App或者macOS Cocoa App时，常见的main函数，只包含如下代码：

```objective-c
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

另外，我们也可以看macOS下的命令行程序：

```c++
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
    return 0;
}
```

对比这两种程序：

比如iOS App，启动后会一直运行，等待用户触摸、输入等，在接收到点击后，就会立即响应，完成本次响应后，会等待下次用户操作。只要用户不主动退出或者程序闪退，会一直在循环等待。

而macOS下的命令行程序，启动后，执行程序（可能有用户输入），执行完毕后会立即退出。

两者最大的区别在于：**是否能持续响应用户输入**。

## 2. 事件循环

之所以，iOS App能持续响应，保证程序运行状态，在于其有一个事件循环——[Event Loop](https://en.wikipedia.org/wiki/Event_loop)。

事件循环机制，即线程能随时响应并处理事件的机制。这种机制要求线程不能退出，而且需要高效的完成事件调度与处理。

事件循环在很多编程语言，或者说不同的操作系统层面都支持。比如JS中的事件循环、Windows下的消息循环，在iOS/macOS下，该机制就称为`RunLoop`。

事件循环在本质上是如下一个编程实现：

```objective-c
int main(int argc, char * argv[]) {
    @autoreleasepool {
        int retVal = 0;
        do {
            // 睡眠中等待消息
            int message = sleep_and_wait();
            // 处理消息
            retVal = process_message(message);
        } while (0 == retVal);
    }
    return 0;
}
```

`RunLoop`是iOS/macOS下的事件循环机制。

# 二、Runloop

`RunLoop`是iOS/macOS下的事件循环机制，在OC编码实现上，它又是一个对象，该对象管理了其需要处理的时间和消息，并提供了一个入口函数来执行。

一旦线程执行入口函数之后，就会一直处于函数内部**“接受消息->等待->处理”**的循环中，直到退出循环。

在iOS App中：

```objective-c
@autoreleasepool {
       return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
   }
```

以上代码会执行以下步骤：

1. 创建auto-release pool；
2. 调用UIApplicationMain，该函数内部会创建一个UIApplication singleton对象；
3. 开始执行Run Loop；
4. 这些步骤完毕后，代表app 已经开始运行，调用UIApplication 的delegate的 `-application:didFinishLaunchingWithOptions:`方法；



一旦iOS App执行上面的步骤之后，就能够与用户进行交互。

RunLoop就保证了这个交互中的所有事件处理，那么`RunLoop`能处理哪些常见的事件呢？

有以下常见事件：

1. 定时器（Timer）、方法调用（PerformSelector）
2. GCD Async Main Queue
3. 事件响应、手势识别、界面刷新
4. 网络请求
5. 自动释放池AutoreleasePool

![image-20181224040343583](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-200344.png)

一旦有了RunLoop的加持后：

![image-20181224040906411](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-23-200907.png)