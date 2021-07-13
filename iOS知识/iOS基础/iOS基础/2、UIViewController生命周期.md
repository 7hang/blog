#### 前言

在iOS开发中`UIViewController`扮演者非常重要的角色，它是视图`view`和数据`model`的桥梁，通过`UIViewController`的管理有条不紊的将数据展示在视图上。作为UIKit中最基本的一个类，一般复杂的项目都离不开`UIViewController`作为基类。所以了解`UIViewController`的整个生命周期是有必要的。

#### UIViewController的生命周期

我们先看一下下面有关`UIViewController`生命周期有关的几个函数：

```objective-c
+ (void)initialize {
    NSLog(@"========   类初始化方法： initialize   =======\n");
}

- (instancetype)init {
     self = [super init];
    NSLog(@"========   实例初始化方法： init   =======\n");
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    NSLog(@"========   从归档初始化：  initWithCoder:(NSCoder *)aDecoder   =======\n");
    return self;
}

- (void)loadView {
    [super loadView];
    NSLog(@"========   加载视图： loadView   =======\n");
}

#pragma mark- life cycle
- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"========   将要加载视图： viewDidLoad   =======\n");
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"========   视图将要出现： viewWillAppear:(BOOL)animated   =======\n");
}

- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    NSLog(@"========   将要布局子视图： viewWillLayoutSubviews   =======\n");
}

- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    NSLog(@"========   已经布局子视图： viewDidLayoutSubviews   =======\n");
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"========   视图已经出现： viewDidAppear:(BOOL)animated   =======\n");
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    NSLog(@"========   视图将要消失： viewWillDisappear:(BOOL)animated   =======\n");
}

- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    NSLog(@"========   视图已经消失： viewDidDisappear:(BOOL)animated   =======\n");
}

- (void)dealloc {
    NSLog(@"========   释放： dealloc   =======\n");
}
复制代码
```

看到这么多的函数是不是感觉理解起来很复杂，其实从上往下看一遍之后就可以了解到他们关系其实很明朗。我们运行一下看看他们之间的调用顺序：

```objective-c
2019-05-21 09:29:41.532655+0800 ThinTableVIew1[77845:3751647] ========  ViewController1 类初始化方法： initialize   =======

2019-05-21 09:29:41.533321+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  实例初始化方法： init   =======

2019-05-21 09:29:41.539746+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  加载视图： loadView   =======

2019-05-21 09:29:41.539975+0800 ThinTableVIew1[77845:3751647] ========  ViewController1 将要加载视图： viewDidLoad   =======

2019-05-21 09:29:41.540280+0800 ThinTableVIew1[77845:3751647] ========  ViewController1 视图将要出现： viewWillAppear:(BOOL)animated   =======

2019-05-21 09:29:41.581539+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  将要布局子视图： viewWillLayoutSubviews   =======

2019-05-21 09:29:41.581755+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  已经布局子视图： viewDidLayoutSubviews   =======

2019-05-21 09:29:42.086186+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  视图已经出现： viewDidAppear:(BOOL)animated   =======

2019-05-21 09:30:11.567953+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  视图将要消失： viewWillDisappear:(BOOL)animated   =======

2019-05-21 09:30:11.568210+0800 ThinTableVIew1[77845:3751647] ========  ViewController 视图将要出现： viewWillAppear:(BOOL)animated   =======

2019-05-21 09:30:12.074866+0800 ThinTableVIew1[77845:3751647] ======== ViewController1  视图已经消失： viewDidDisappear:(BOOL)animated   =======

2019-05-21 09:30:12.075103+0800 ThinTableVIew1[77845:3751647] ========  ViewController 视图已经出现： viewDidAppear:(BOOL)animated   =======

2019-05-21 09:30:12.075378+0800 ThinTableVIew1[77845:3751647] ========  ViewController1 释放： dealloc   =======
复制代码
```

`ViewController`是上层视图`ViewController1`是push过去的二级视图。我们可以看到`ViewController1`从初始化到释放的整个过程。

1. `+ (void)initialize`：函数并不会每次创建对象都调用，只有在第一次初始化的时候才会调用，再次创建将不会调用`initialize`方法。
2. `init`方法和`initCoder`方法相似，知识被调用的环境不一样。如果用代码初始化，会调用`init`方法，从nib文件或者归档(`xib`、`storyboard`)进行初始化会调用`initCoder`。`initCoder`是`NSCoding`协议中的方法，`NSCoding`是负责编码解码，归档处理的协议。
3. `loadView`：是开始加载`view`的起始方法，除非手动调用，否则在`ViewController`的生命周期中只调用一次。
4. `viewDidLoad`：是我们最常用的方法，类成员对象和变量的初始化我们都会放在这个方法中。在创建类后无论视图展现还是消失，这个方法也只会在布局是调用一次。
5. `viewWillAppear:(BOOL)animated`：方法 是在视图将要展现出来的时候调用。
6. `viewWillLayoutSubviews`：方法是在将要布局子视图的时候调用。
7. `viewDidLayoutSubviews`：方法是在子视图布局完成后调用。
8. `viewDidAppear:(BOOL)animated`：方法是视图已经出现。
9. `viewWillDisappear:(BOOL)animated`：方法是视图即将消失。
10. `viewDidDisappear:(BOOL)animated`：视图已经消失。
11. `dealloc`：`ViewController`被释放时调用。

#### 总结

从上面的打印我们可以看到`ViewController`的整个生命周期，从初始化到被释放。其中在退出`ViewController`控制器的时候可以看到程序会先调用`ViewController`的`viewWillDisappear:(BOOL)animated`方法然后会调用上一级视图的`viewWillAppear:(BOOL)animated`方法，然后才会调用当前视图的`viewDidDisappear:(BOOL)animated`方法。