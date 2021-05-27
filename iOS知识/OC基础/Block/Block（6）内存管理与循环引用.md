# Block（六）内存管理与循环引用

之前的`Block`系列文章，我们已经对`Block`的方方面面都作了阐述，但是针对内存管理，我们还有最后一个需要讨论的话题——循环引用。

本文将会针对循环引用做一个讨论，同时，将会对Block涉及的内存管理做一个总结。

# 一、循环引用

## 1.1 自循环引用

`Block`内调用对象，对象拥有`Block`，导致产生的双向强引用——循环引用。

![对象自循环引用](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-10-004929.png)



解决方法，就是打破该循环，利用`__weak`或`__unsafe_unretained`来打破该循环。

- `__weak`：不会产生强引用，指向的对象销毁时，会自动让指针置为nil。
- `__unsafe_unretained`：不会产生强引用，不安全，指向的对象销毁时，指针存储的地址值不变，也就是僵尸对象。

## 1.2 `__block`导致的循环引用

使用`__block`导致产生的三角强引用——循环引用。

![__block导致的循环引用](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-10-004958.png)



打破该三角循环，我们主动将`person`对象，在`Block`内部置为nil，但是**需要主动调用执行**该`Block`。

# 二、内存管理

下面总结了，捕获变量和内存管理的Block内如何处理。

![block的内存管理](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-12-10-020507.png)

到此，Block的部分，我们已经讲述完了。