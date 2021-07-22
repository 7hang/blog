

# dyld

## 加载dylibs

dyld是iOS平台上的二进制加载器或者说动态链接器，也是内核的得力”小助手”。它负责将程序依赖的各种动态库加载到进程。同时也肩负着修复内部和外部指针指向的责任([传送门，dyld开源代码](https://opensource.apple.com/source/dyld/))。下面我们就来看看dyld是如何帮助内核完成加载工作的。

![img](https://res.cloudinary.com/dwpjzbyux/image/upload/v1556313280/blogImages/app_launch/dyld_vxpafe.png)

从上图可以看到，ALSR将dyld也加载到了进程中一个随机地址，此时的dyld和App享有同样的权限。其实从加载完dyld之后，内核要做的事情就结束了，之后的所有步骤都由dyld完成。首先，dyld会读取Mach-O文件中的指令(Load commands)，去将其依赖的各个动态库映射到当前的逻辑地址空间，如果该动态库还依赖其他动态库，比方说下图的A库依赖C库，dyld会递归的将没加载过的dylib都加载到当前进程中(*具体由ImageLoader完成*)，直到所有的动态库加载完毕。Apple官方称，一个App通常会加载1到400个dylibs! 不过幸运的是，这其中大部分是系统的dylibs，Apple在操作系统启动的时候，也已经帮我们提前计算和缓存了这些dylibs的信息，这样dyld加载它们的时候就会快很多。

![img](https://res.cloudinary.com/dwpjzbyux/image/upload/v1556313614/blogImages/app_launch/dylibs_qo5got.png)

## Dirty & Clean Page

我们在上文提到: 当**Copy-On-Write**发生时，该page会被标记为dirty，同时会被复制。下面我们通过一个实例来进一步理解虚拟内存和Mach-O布局，以及dyld是如何产生出dirty page的。

![img](https://res.cloudinary.com/dwpjzbyux/image/upload/v1556334804/blogImages/app_launch/clean_dirty_page_gl1rhj.png)

上图展示的是两个不同的进程共享同一个dylib的使用场景。我们从左到右看，左边的进程1先加载了该动态库，通过读取**Load Commands**, dyld知道要先加载__TEXT段(可读，可执行)，上图__TEXT段大小为3个page，但是dyld只会先将第一个page映射到物理RAM。之后读取__DATA段信息(可读可写)，在Mach-O章节中，我们已经知道，__DATA段存储了全局变量，而大部分的全局变量又都被初始化为0，所以在第一次被加载的时候，虚拟内存技术会将这些全局变量直接分配到一些page上(上图是3个page)，不从磁盘中读取。接着dyld加载__LINKEDIT段，并将其映射到物理RAM2。当dyld加载__LINKEDIT(只读)段时，会被告知需要做一些”修正”以便dylib可以被顺利运行，这时，dyld就需要向__DATA段写入一些数据了。当写入发生时，该page对应的物理RAM就包含了当前进程信息，变成了dirty page。最终，这个dylib占用了8个page，包含2个clean page，一个dirty page(其余还未被映射)。

这时，如果有第二个进程同时引用了这个dylib，那么会发生什么呢？第一步同样是加载__TEXT段，不同的是，这次内核会发现在物理内存的某一处，已经加载过对应的__TEXT段，所以这次只是简单地把映射重定向下，不做任何IO操作。__LINKEDIT也是如此，区别就是这次变快了许多。当内核在寻找__DATA段的时候，它先会去检查是否还存在干净的副本，如果有，则直接映射；如果没有则需要重新从磁盘中读取，读取后dyld做修正，所以这个page也会变为dirty page。当dyld完成了修正过程后，__LINKEDIT也就不被再需要，内核可以在适当的时候释放其映射的物理RAM。

总结下: 当两个进程共享一个dylib时,使用虚拟内存技术映射物理RAM，把原本16个脏页面变成了两个脏页面和一个干净的共享页面。但上例是针对Apple自己提供的动态库，如果是我们自己写的cocoatouch framework，不排除当两个进程共用一个framework时，可以共享某些page。例如两个App都依赖Alamofire网络库时，一个App先行加载其Mach-O文件到物理内存；当另一个App加载Alamofire时，直接映射即可。

# Rebase & Binding

这个步骤就是上文说到的”修正”步骤, 同时也用到了上文提到的PIC技术。Reabse是指修正内部指针指向; Binding是指修正外部指针指向。如下图所示: 指向_malloc和_free的指针是修正后的外部指针指向。而指向__TEXT的指针则是被修复后的内部指针指向。

![img](https://res.cloudinary.com/dwpjzbyux/image/upload/v1556355653/blogImages/app_launch/rebase_binding_hyuhqt.png)

这个步骤发生的原因我们上文提到过: ASLR。由于起始地址的偏移，所有_DATA段内指向Mach-O文件内部的指针都需要增加这个偏移量, 并且这些指针的信息可以在__LINKEDIT段中找到。既然Rebase需要写入数据到__DATA段，那也就意味着**Copy-On-Write**势必会发生，dirty page的创建，IO的开销也无法避免。

Binding是__DATA段对外部符号的引用。不过和Rebase不同的是，binding是靠字符串的匹配来查找符号表的，虽说没有多少IO，但是计算多，比Rebase慢。

使用如下命令可查看所有Resabe和Binding的修复:

复制

```
xcrun dyldinfo -rebase -bind -lazy_bind NetworkTransport.app/NetworkTransport
```

# Main函数的调用

dyld调用UIApplicationMain()。

总结下: 内核加载App和dyld到进程中，dyld加载所有依赖的dylibs，修正所有DATA page，之后Objc runtime初始化所有类，最后调用main函数。
