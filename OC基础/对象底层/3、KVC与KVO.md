# Objective-C（九）KVC与KVO

# 一、概述

 `KVO`全称`Key Value Observing`，是苹果提供的一套事件通知机制。允许对象监听另一个对象特定属性的改变，并在改变时接收到事件。由于`KVO`的实现机制，只针对属性才会发生作用，一般继承自`NSObject`的对象都默认支持`KVO`。

 `KVO`和`NSNotificationCenter`都是`iOS`中观察者模式的一种实现。区别在于，相对于被观察者和观察者之间的关系，`KVO`是一对一的，而不是一对多的。`KVO`对被监听对象无侵入性，不需要修改其内部代码即可实现监听。

 `KVO`可以监听单个属性的变化，也可以监听集合对象的变化。通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，当代理对象的内部对象发生改变时，会回调`KVO`监听的方法。集合对象包含`NSArray`和`NSSet`。



# 二、KVO基本使用

## 2.1 注册观察者

```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    self.person1 = [[BFPerson alloc] init];
    self.person1.age = 28;
    self.person1.name = @"weng";
    [self addObserver];
}
- (void)addObserver
{
    NSKeyValueObservingOptions option = NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew;
    [self.person1 addObserver:self forKeyPath:@"age" options:option context:@"age chage"];
    [self.person1 addObserver:self forKeyPath:@"name" options:option context:@"name change"];
}
```

## 2.2 监听回调

```objective-c
/**
 观察者监听的回调方法
 @param keyPath 监听的keyPath
 @param object 监听的对象
 @param change 更改的字段内容
 @param context 注册时传入的地址值
 */
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"监听到%@的%@属性值改变了 - %@ - %@", object, keyPath, change, context);
}
```

## 2.3 调用

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    self.person1.age = 29;
    self.person1.name = @"hengcong";
}
```

### 2.3.1 其他调用方式

```objective-c
//下面调用方式都可以出发KVO
 self.person1.age = 29;
 [self.person1 setAge:29];
 [self.person1 setValue:@(29) forKey:@"age"];
 [self.person1 setValue:@(29) forKeyPath:@"age"];
```

### 2.3.2 手动调用

KVO在属性发生改变时的调用是自动的，如果想要手动控制这个调用时机，或想自己实现KVO属性的调用，则可以通过KVO提供的方法进行调用。

下面以`age`属性为例：

#### 2.3.2.1 禁用自动调用

```objective-c
//age不需要自动调用，age属性之外的（含name）自动调用
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = NO;
    if ([key isEqualToString:@"age"]) {
        automatic = NO;
    } else {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
    return automatic;
}
```

上面方法也等同于下面两个方法：

```objective-c
+ (BOOL)automaticallyNotifiesObserversOfAge
{
    return NO;
}
```

针对每个属性，KVO都会生成一个**‘+ (BOOL)automaticallyNotifiesObserversOfXXX’**方法，返回是否可以自动调用KVO

假如实现上述方法，我们会发现，此时改变`age`属性的值，无法触发KVO，还需要实现手动调用才能触发KVO。

#### 2.3.2.2 手动调用实现

```objective-c
- (void)setAge:(NSInteger)age
{
    if (_age != age) {
        [self willChangeValueForKey:@"age"];
        _age = age;
        [self didChangeValueForKey:@"age"];
    }
}
```

实现了（1）禁用自动调用（2）手动调用实现 两步，`age`属性手动调用就实现了，此时能和自动调用一样，触发KVO。

## 2.4 移除观察者

```
- (void)dealloc
{
    [self removeObserver];
}

- (void)removeObserver
{
    [self.person1 removeObserver:self forKeyPath:@"age"];
    [self.person1 removeObserver:self forKeyPath:@"name"];
}
```

## 2.5 Crash

KVO若使用不当，极容易引发Crash。

### 2.5.1 观察者未实现监听方法

若观察者对象**-observeValueForKeyPath:ofObject:change:context:**未实现，将会Crash

> Crash：**Terminating app due to uncaught exception ‘NSInternalInconsistencyException’, reason: ‘<ViewController: 0x7f9943d06710>: An -observeValueForKeyPath:ofObject:change:context: message was received but not handled.**

### 2.5.2 未及时移除观察者

> Crash： Thread 1: EXC_BAD_ACCESS (code=1, address=0x105e0fee02c0)

```objective-c
//观察者ObserverPersonChage
@interface ObserverPersonChage : NSObject
  //实现observeValueForKeyPath: ofObject: change: context:
@end

//ViewController
- (void)addObserver
{
    self.observerPersonChange = [[ObserverPersonChage alloc] init];
    [self.person1 addObserver:self.observerPersonChange forKeyPath:@"age" options:option context:@"age chage"];
    [self.person1 addObserver:self.observerPersonChange forKeyPath:@"name" options:option context:@"name change"];
}

//点击按钮将观察者置为nil，即销毁
- (IBAction)clearObserverPersonChange:(id)sender {
    self.observerPersonChange = nil;
}

//点击改变person1属性值
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    self.person1.age = 29;
    self.person1.name = @"hengcong";
}
```

1. 假如在当前ViewController中，注册了观察者，点击屏幕，改变被观察对象person1的属性值。
2. 点击对应按钮，销毁观察者，此时**self.observerPersonChange**为nil。
3. 再次点击屏幕，此时Crash；

### 2.5.3 多次移除观察者

> **Cannot remove an observer <ViewController 0x7fc6dc00c090> for the key path “age” from <BFPerson 0x6000014acd00> because it is not registered as an observer.**

## 2.6 keyPath字符串的弊端

在注册Observe时，传入keyPath为字符串类型，keyPath极容易误写。

```objective-c
[self.person1 addObserver:self forKeyPath:@"age" options:option context:@"age chage"];
```

优化的方案是：

```objective-c
[self.person1 addObserver:self forKeyPath:NSStringFromSelector(@selector(age)) options:option context:@"age change"];
```

## 2.7 属性依赖

```objective-c
/**
 如果age改变
 观察者也会收到name改变的通知
 */
+ (NSSet<NSString *> *)keyPathsForValuesAffectingAge
{
    NSSet *set = [NSSet setWithObjects:@"name", nil];
    return set;
}
```

# 三、原理

下面是摘自官方文档给出的原理描述：

> #### Key-Value Observing Implementation Details
>
> Automatic key-value observing is implemented using a technique called *isa-swizzling*.
>
> The `isa` pointer, as the name suggests, points to the object’s class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.
>
> When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.
>
> You should never rely on the `isa` pointer to determine class membership. Instead, you should use the `class` method to determine the class of an object instance.

我们同时观察添加KVO之后，中间对象的方法列表以及未添加之前的方法列表：

| 类                   | 方法列表（忽略name属性相关方法）      |
| -------------------- | ------------------------------------- |
| BFPerson             | **test, .cxx_destruct, setAge:, age** |
| NSKVONotify_BFPerson | **setAge:, class, dealloc, _isKVOA**  |

- `isa`交换技术

  - 交换之后，调用任何`BFPerson`对象的方法，都会经过`NSKVONotify_BFPerson`，但是不同的方法，有不同的处理方式。

    - 调用**监听的属性设置方法**，如 `setAge:`，都会先调用`NSKVONotify_BFPerson`对应的属性设置方法；
    - 调用**非监听属性设置方法**，如`test`，会通过`NSKVONotify_BFPerson`的`superclass`，找到`BFPerson`类对象，再调用其`[BFPerson test]`方法

  - 交换之后，`isa`指向的并不是该类的真实反映，同样`object_getClass`返回的是`isa`指向的对象，所以也是不可靠的。

    比如使用KVO之后，通过`object_getClass`得到的是生成的中间对象`NSKVONotify_BFPerson`，而不是`BFPerson`。

  - 要想获得该类真实的对象，需要通过`class`对象方法获取。

    假如通过**[self.person1 class]**得到的是`BFPerson`对象。

- **[self.person1 class]**得到的仍然是`BFPerson`对象，为什么？

  - `NSKVONotify_BFPerson`重写了其`class`对象方法，返回的是`BFPerson`；

- _isKVOA

  返回是否是KVO；

- delloc

  做一些清理工作

到此，基本上`NSKVONotifying_BFPerson`类已经成型（相关代码参考项目），结合调用流程，我们绘制出下面对比图。



# 四、KVC基本使用

## （1）常见的API

```objective-c
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKeyPath:(NSString *)keyPath;
- (id)valueForKey:(NSString *)key; 
```

其中，有两个方法要注意：

- valueForKey与objectForKey的区别

|             | valueForKey                                                  | objectForKey                     |
| ----------- | ------------------------------------------------------------ | -------------------------------- |
| 无key的处理 | 无该key，crash，NSUndefinedKeyException                      | 无该key返回nil                   |
| 来源        | KVC主要方法                                                  | NSDictionary的方法               |
| 符号        | 若以 @ 开头，去掉 @ ，用剩下部分作为 key 执行 [super valueForKey:] | key 不是以 @ 符号开头， 两者等同 |

- setValue与setObject的区别

|           | setValue                                                     | setObject                 |
| --------- | ------------------------------------------------------------ | ------------------------- |
| value     | value可为nil，当value为nil的时候，会自动调用removeObject:方法 | value是不能为nil          |
| 来源      | KVC的主要方法                                                | NSMutabledictionary特有的 |
| key的参数 | 只能是NSString                                               | setObject: 可任何类型     |

NSKeyValueCoding类别中还有其他的一些方法，例如

```objective-c
//默认返回YES，表示如果没有找到set<Key>方法的话，会按照_key，_iskey，key，iskey的顺序搜索成员，设置成NO就不这样搜索
+ (BOOL)accessInstanceVariablesDirectly;

//KVC提供属性值确认的API，它可以用来检查set的值是否正确、为不正确的值做一个替换值或者拒绝设置新值并返回错误原因。
- (BOOL)validateValue:(inout id __nullable * __nonnull)ioValue forKey:(NSString *)inKey error:(out NSError **)outError;

//这是集合操作的API，里面还有一系列这样的API，如果属性是一个NSMutableArray，那么可以用这个方法来返回
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;

//如果Key不存在，且没有KVC无法搜索到任何和Key有关的字段或者属性，则会调用这个方法，默认是抛出异常
- (nullable id)valueForUndefinedKey:(NSString *)key;

//和上一个方法一样，只不过是设值。
- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key;

//如果你在SetValue方法时面给Value传nil，则会调用这个方法
- (void)setNilValueForKey:(NSString *)key;

//输入一组key,返回该组key对应的Value，再转成字典返回，用于将Model转到字典。
- (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
```

## （2）集合API

有序集合对应方法如下：

```objective-c
-countOf<Key>//必须实现，对应于NSArray的基本方法count:2  
    
-objectIn<Key>AtIndex:
-<key>AtIndexes://这两个必须实现一个，对应于 NSArray 的方法 objectAtIndex: 和 objectsAtIndexes:

-get<Key>:range://不是必须实现的，但实现后可以提高性能，其对应于 NSArray 方法 getObjects:range:

-insertObject:in<Key>AtIndex:

-insert<Key>:atIndexes://两个必须实现一个，类似于 NSMutableArray 的方法 insertObject:atIndex: 和 insertObjects:atIndexes:

-removeObjectFrom<Key>AtIndex:

-remove<Key>AtIndexes://两个必须实现一个，类似于 NSMutableArray 的方法 removeObjectAtIndex: 和 removeObjectsAtIndexes:

-replaceObjectIn<Key>AtIndex:withObject:

-replace<Key>AtIndexes:with<Key>://可选的，如果在此类操作上有性能问题，就需要考虑实现之
```

无序集合对应方法如下：

```objective-c
-countOf<Key>//必须实现，对应于NSArray的基本方法count:

-objectIn<Key>AtIndex:
-<key>AtIndexes://这两个必须实现一个，对应于 NSArray 的方法 objectAtIndex: 和 objectsAtIndexes:

-get<Key>:range://不是必须实现的，但实现后可以提高性能，其对应于 NSArray 方法 getObjects:range:

-insertObject:in<Key>AtIndex:

-insert<Key>:atIndexes://两个必须实现一个，类似于 NSMutableArray 的方法 insertObject:atIndex: 和 insertObjects:atIndexes:

-removeObjectFrom<Key>AtIndex:

-remove<Key>AtIndexes://两个必须实现一个，类似于 NSMutableArray 的方法 removeObjectAtIndex: 和 removeObjectsAtIndexes:

-replaceObjectIn<Key>AtIndex:withObject:

-replace<Key>AtIndexes:with<Key>://这两个都是可选的，如果在此类操作上有性能问题，就需要考虑实现之
```

## （3）使用场景

### a.动态地取值和设值

### b.访问和修改私有变量

### c.Model和字典转换

### d.修改一些控件的内部属性

例如设置：UITextField中的placeHolderText

```objective-c
[textField setValue:[UIFont systemFontOfSize:25.0] forKeyPath:@"_placeholderLabel.font"];
```

如何获取控件的内部属性？

```objective-c
unsigned int count = 0;
objc_property_t *properties = class_copyPropertyList([UITextField class], &count);
for (int i = 0; i < count; i++) {
    objc_property_t property = properties[i];
    const char *name = property_getName(property);
    NSLog(@"name:%s",name);
}
```

### e.高阶消息传递

当对容器类使用KVC时，valueForKey:将会被传递给容器中的每一个对象，而不是容器本身进行操作。结果会被添加进返回的容器中，这样，开发者可以很方便的操作集合来返回另一个集合。

```objective-c
NSArray *arr = @[@"ali",@"bob",@"cydia"];
NSArray *arrCap = [arr valueForKey:@"capitalizedString"];
for (NSString *str  in arrCap) {
    NSLog(@"%@",str);        //Ali\Bob\Cydia
}
```

### f.KVC中的函数操作集合

- 简单集合运算符

  - @avg
  - @count
  - @max
  - @min
  - @sum

  ```objective-c
  @interface Book : NSObject
  @property (nonatomic,assign)  CGFloat price;
  @end
  
  NSArray* arrBooks = @[book1,book2,book3,book4];
  NSNumber* sum = [arrBooks valueForKeyPath:@"@sum.price"];
  ```

- 对象运算符

  - @distinctUnionOfObjects
  - @unionOfObjects

  ```objective-c
  // 获取所有Book的price组成的数组，并且去重
  NSArray* arrDistinct = [arrBooks valueForKeyPath:@"@distinctUnionOfObjects.price"];
  ```

- Array和Set操作符(集合中包含集合的情形)

  - @distinctUnionOfArrays
  - @unionOfArrays
  - @distinctUnionOfSets

  ![collection_keypath](http://blog-1251606168.file.myqcloud.com/blog_2018/2018-11-27-083013.png)

  

# 五、KVC原理

项目源码在**[KVC-02-principle](https://github.com/wenghengcong/LearnCollection/tree/master/OCObject/KVO与KVC/KVC-02-principle)**

- setValue:forKey

  ![image-20181127173412899](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-27-093413.png)

- valueForKey

  ![image-20181127174325478](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-11-27-094325.png)

# 六、扩展

## 1._NSSetLongLongValueAndNotify

```
//抽出Foundation库，查看其中Notify的函数
$ nm ./Foundation | grep LongValueAndNotify                                       
22bc9290 t __NSSetLongLongValueAndNotify
22bc90a0 t __NSSetLongValueAndNotify
22bc93a0 t __NSSetUnsignedLongLongValueAndNotify
22bc9198 t __NSSetUnsignedLongValueAndNotify
22bc9d18 t ____NSSetLongLongValueAndNotify_block_invoke
22bc9ca8 t ____NSSetLongValueAndNotify_block_invoke
22bc9d4c t ____NSSetUnsignedLongLongValueAndNotify_block_invoke
22bc9ce0 t ____NSSetUnsignedLongValueAndNotify_block_invoke
```