# blog
### 编译原理

>简单来说分编译前端和编译后端，中间产物为IR。clang将OC语言编译为IR（swift使用的是swiftc），LLVM将IR编译为对应汇编。
>
>clang不会将OC转换为c++.

[iOS编译原理](https://awanglilong.github.io/2021/04/16/iOScompile/)简单介绍OC的编译步骤和原理

### 动态库和静态库

>动态库优点是包小，可共用，能动态加载。缺点是加载速度慢。但可共用和动态加载Apple不允许使用。所以优点是包小，缺点是加载速度慢。

[如何打造一个让人愉快的框架](https://onevcat.com/2016/01/create-framework/#cocoa-touch-framework) 介绍如何开发开源框架，介绍动态库和静态库。

[iOS 静态库和动态库](https://www.cnblogs.com/dins/p/ios-jing-tai-ku-he-dong-tai-ku.html) 动态库包小，静态库冷启动速度快

### iOS基础

> OC语言底层原理研究方式主要通过读源码、clang转换为c++、看内存地址三种方式研究底层实现原理

#### 语言基础

[Swift（一）对象内存模型](https://mp.weixin.qq.com/s/zIkB9KnAt1YPWGOOwyqY3Q)  Swift的对象的内存模型也是结构体，复杂度比OC低 [Swift 中的类](https://www.jianshu.com/p/07f7523f2d6d)

[Swift（二）消息派发](https://awanglilong.github.io/2021/04/19/swift-message/)  结构体消息派发是直接派发，类的消息派发主要是虚派表

[Swift（三）协议的使用](https://onevcat.com/2016/11/pop-cocoa-1/#%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%BC%96%E7%A8%8B%E7%9A%84%E5%9B%B0%E5%A2%83)使用Struct和Protocol的面向协议编程，是Swift的高效方式。

#### 内存管理

[内存管理Swift弱引用](https://zhuanlan.zhihu.com/p/58179258) 

Swift的弱引用是SideTable管理，是个散列表，但是存的是指出的weak对象。

OC的弱引用是SideTable管理，是个全局三层散列表，weak_entry_t里存的是指向当前对象的，weak对象。

#### RunLoop

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)





### 