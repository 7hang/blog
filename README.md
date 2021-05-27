

## iOS基础

### OC基础

> OC语言底层原理研究方式主要通过读源码、clang转换为c++、看内存地址三种方式研究底层实现原理

#### 编译原理

>简单来说分编译前端和编译后端，中间产物为IR。clang将OC语言编译为IR（swift使用的是swiftc），LLVM将IR编译为对应汇编。
>
>clang不会将OC转换为c++.

[iOS编译原理](https://awanglilong.github.io/2021/04/16/iOScompile/)简单介绍OC的编译步骤和原理

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

### iOS机制

#### 动态库和静态库

>动态库优点是包小，可共用，能动态加载。缺点是加载速度慢。但可共用和动态加载Apple不允许使用。所以优点是包小，缺点是加载速度慢。

[如何打造一个让人愉快的框架](https://onevcat.com/2016/01/create-framework/#cocoa-touch-framework) 介绍如何开发开源框架，介绍动态库和静态库。

[iOS 静态库和动态库](https://www.cnblogs.com/dins/p/ios-jing-tai-ku-he-dong-tai-ku.html) 动态库包小，静态库冷启动速度快

#### 沙盒机制

[iOS沙盒机制](https://zhuyunsun.github.io/2020/04/21/iOS%E6%B2%99%E7%9B%92%E6%9C%BA%E5%88%B6/)

#### APP瘦身

[iOS App瘦身实践](https://mp.weixin.qq.com/s/xzlFQJ2b-rrw5QIszSLXXQ)

#### APP性能

[iOS卡顿问题处理](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

[iOS拾遗——为什么必须在主线程操作UI](https://juejin.cn/post/6844903763011076110)

[APP性能检测方案汇总](https://www.jianshu.com/p/95df83780c8f)

#### APP绘制原理



#### iOS架构

[iOS应用架构谈](https://casatwy.com/iosying-yong-jia-gou-tan-kai-pian.html)iOS架构师写的

[跳出面向对象思想(一)继承](https://casatwy.com/tiao-chu-mian-xiang-dui-xiang-si-xiang-yi-ji-cheng.html)但是一些基本体系的继承都在使用

#### 源码解读

[AFNetworking源码解读](https://awanglilong.github.io/2021/04/21/AFNetworking-code-read/)

[Aspect](https://halfrost.com/ios_aspect/#toc-17)源码分析  实现有点像KVO的实现原理

[fishhook](https://juejin.cn/post/6844904175625568270)源码分析

[Analyze](https://github.com/draveness/analyze)iOS开源框架源码分析

[Laucp'sBlog](https://chipengliu.github.io/)源码分析

[WebViewJavascriptBridge原理解析](https://www.jianshu.com/p/d45ce14278c7)

[JJException](https://github.com/jezzmemo/JJException)

[iOS Crash防护](https://juejin.cn/post/6874435201632583694)

### 面试问题

[iOSInterviewQuestions](https://github.com/ChenYilong/iOSInterviewQuestions)

[iOS知识思维导图](https://github.com/MisterBooo/ReadyForBAT)

[2020年iOS面试题总结](https://www.xuebaonline.com/2020%E5%B9%B4iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93(%E4%B8%80)/)

[2020年iOS面试题总结(二)](https://www.xuebaonline.com/2020%E5%B9%B4iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93(%E4%BA%8C)/)

[2020年iOS面试题总结(三)](https://www.xuebaonline.com/2020%E5%B9%B4iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93(%E4%B8%89)/)

[出一套 iOS 高级面试题](https://juejin.cn/post/6844903645243260941#heading-2)

[iOS面试了20几家总结出来的面试题（一）](https://juejin.cn/post/6854573212165111822)

[iOS面试了20几家总结出来的面试题（二）](https://juejin.cn/post/6854573212169142285#heading-15)

[阿里、字节：一套高效的iOS面试题](https://juejin.cn/post/6844904064937902094#heading-19)

[腾讯社招iOS面试记录](https://juejin.cn/post/6844903633117528078)

[全新角度剖析--iOS面试](https://juejin.cn/post/6899689319809286158)

[iOS面试珠玑](https://juejin.cn/post/6844903615157501965)

[iOS 面试题整理](https://juejin.cn/post/6844903871219761165)

[2020年面向iOS开发人员的知识点总结（更新中）](https://juejin.cn/post/6844904201328279565)

[iOS 第二梯队面试败北感悟](https://juejin.cn/post/6844903591010910216)

[进入大厂的面试经验](https://juejin.cn/post/6844904086358212621)

## 计算机基础

### 设计模式

[设计模式SOLID原则](https://segmentfault.com/a/1190000023114300)

[图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/)时序图对理解设计模式非常有帮助

### 算法

[fucking-algorithm](https://github.com/labuladong/fucking-algorithm)提出算法结题框架

[小浩算法](https://www.geekxh.com/)常见200算法题目解法

[五分钟学算法](https://www.cxyxiaowu.com/)算法动画做的最好的

### 网络协议

[IP 基础知识](https://www.cnblogs.com/xiaolincoding/p/12830287.html)

[键入网址后，到网页显示](https://www.cnblogs.com/xiaolincoding/p/12508499.html)

[TCP 三次握手和四次挥手](https://www.cnblogs.com/xiaolincoding/p/12638546.html)

[三次握手四次挥手优化](https://www.cnblogs.com/xiaolincoding/p/13067971.html)

[TCP 重传、滑动窗口、流量控制、拥塞控制](https://www.cnblogs.com/xiaolincoding/p/12732052.html)

[HTTP简介](https://www.cnblogs.com/xiaolincoding/p/12442435.html)

[HTTP/1.1 优化](https://www.cnblogs.com/xiaolincoding/p/14442218.html)

[HTTPS原理简介](https://www.cnblogs.com/xiaolincoding/p/14274353.html)

[基础面试题](https://github.com/wolverinn/Waking-Up)

### 操作系统

| [小林图解系统](https://github.com/awanglilong/blog/blob/main/%E7%B3%BB%E7%BB%9F/%E5%9B%BE%E8%A7%A3%E7%B3%BB%E7%BB%9F-%E5%B0%8F%E6%9E%97coding-v1.0.pdf) | 成系统的，简单易懂的讲解操作系统文档。 |
| ------------------------------------------------------------ | -------------------------------------- |
| [CPU 执行程序的](https://www.cnblogs.com/xiaolincoding/p/13796525.html) |                                        |
| [内存管理](https://www.cnblogs.com/xiaolincoding/p/13213699.html) |                                        |
| [进程、线程基础知识](https://www.cnblogs.com/xiaolincoding/p/13289992.html) |                                        |
| [进程间通信](https://www.cnblogs.com/xiaolincoding/p/13402297.html) |                                        |
| [调度算法](https://www.cnblogs.com/xiaolincoding/p/13631224.html) |                                        |
| [文件系统](https://www.cnblogs.com/xiaolincoding/p/13499209.html) |                                        |







