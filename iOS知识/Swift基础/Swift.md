Swift

[Swift（一）对象内存模型](https://mp.weixin.qq.com/s/zIkB9KnAt1YPWGOOwyqY3Q)  Swift的对象的内存模型也是结构体，复杂度比OC低 [Swift 中的类](https://www.jianshu.com/p/07f7523f2d6d)

[Swift（二）消息派发](https://awanglilong.github.io/2021/04/19/swift-message/)  结构体消息派发是直接派发，类的消息派发主要是虚派表

[Swift（三）协议的使用](https://onevcat.com/2016/11/pop-cocoa-1/#%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%BC%96%E7%A8%8B%E7%9A%84%E5%9B%B0%E5%A2%83)使用Struct和Protocol的面向协议编程，是Swift的高效方式。

#### 内存管理

[内存管理Swift弱引用](https://zhuanlan.zhihu.com/p/58179258) 

Swift的弱引用是SideTable管理，是个散列表，但是存的是指出的weak对象。

OC的弱引用是SideTable管理，是个全局三层散列表，weak_entry_t里存的是指向当前对象的，weak对象。

