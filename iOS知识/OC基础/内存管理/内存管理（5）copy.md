# 内存管理（五）copy

本文将主要讲述拷贝这个操作以及`copy`关键字，大部分是实际代码应用的部分。

# 一、拷贝

关于拷贝，要了解两个点：

- 为什么要拷贝？
- 如何拷贝？

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-011341.png" alt="image-20190107091218999" style="zoom:50%;" />





# 二、纯量类型的拷贝

```
 一般来讲，标准的拷贝，指的是简单的赋值操作的调用，也就是使用 = 操作符来赋值一个变量给另一个变量，比如说:
int a = 5;
int b = a;
```

 那么`b`就获得了一份`a`的拷贝，`b`和`a`的内存地址是不同的，他们各占不同的内存区域。

 但是如果你这种方式企图复制一个Core Foundation对象，那么复制的仅仅是对象的引用，而对象本身并没有得到实际的复制。

 纯量类型的拷贝简单直接，而对象类型的拷贝，就复杂的多。原因在于两点：

- 对象类型有不同的分类，主要分为**容器类对象**和**非容器类对象**；
- 对象类型要处理更复杂的结构，针对对象中成员变量的处理；

# 三、非容器类对象

 我们先从最常用的字符串对象说起，不过不管是容器类对象还是非容器类对象，`copy`返回不可变对象，`mutablecopy`返回可变对象。

## 3.1 NSString

对`NSString`对象进行`copy`是浅拷贝，而`mutableCopy`是深拷贝。我们可以简单理解：

- `copy`是为了拷贝出一个不可变对象，而`NSString`源对象本身就是不可变对象，系统为了性能和内存优化的考虑，自然只简单增加引用，进行浅拷贝。
- `mutableCopy`是为了拷贝出一个可变对象，其在之后可能改变，那么为了不影响源`NSString`对象，自然要拷贝出一个新对象，进行深拷贝。

针对上面的理解，系统和我们的理解其实是完全一致的。看下面代码实例：

```objective-c
- (void)stringCopy
{
    NSString *str1 = [NSString stringWithFormat:@"123"];
    NSString *str2 = [str1 copy];                     //浅拷贝
    NSMutableString *str3 = [str1 mutableCopy];       //深拷贝
    
    //0xa000000003332313, 0xa000000003332313, 0x1c00524e0
    NSLog(@"%p, %p, %p", str1, str2, str3);
    //-1, -1, 1
    NSLog(@"retainCount: %ld, %ld, %ld ",
          [str1 retainCount],
          [str2 retainCount],
          [str3 retainCount]);
    
    [str1 release];
    [str2 release];
    [str3 release];
}
```

上面的打印，与预想中的有所出入：

- 三个`str`对象的指针出入巨大，按理说，三个对象均分配在堆中，应该是前后的关系。
- 三个`str`对象的引用计数更让人摸不着头脑，按上面的分析，`str1`、`str2`的引用计数为**2**，而不是**-1**，何况**-1**又是什么引用计数。

其实这两个问题，本质上，都是由于`Tagged Pointer`计数引起的。

 `Tagged Pointer`对象会将数据，即”123”，直接存入指针，而`str1`为什么要采用`Tagged Pointer`，`str3`不用，因为系统对`str1`进行`Tagged Pointer`存储，好处很多，也不会有变化的可能。而`str3`是可变对象，就导致了其不可能存在于指针中，否则每次改变都要检查指针是否可以存储，这样实际更为浪费。

`NSSting`拷贝总结：

- 非容器类不可变对象的`copy`是浅拷贝；
- 非容器类不可变对象的`mutableCopy`是深拷贝。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-013811.png" alt="image-20190107093810561" style="zoom:67%;" />

## 3.2 NSMutableString

针对可变字符串对象`NSMutableString`对象，假如源对象为`str1`，分别进行两种拷贝操作：

- `copy`：`str1`是可变对象，要产生一个不可变对象，所以只能新建对象，即深拷贝。
- `mutableCopy`：`str1`是可变对象，要产生另外一个可变对象，而且两个可变对象之间不能相互影响，当然也是新建对象，即深拷贝。

如下是，代码示例：

```objective-c
- (void)mutableStringCopy
{
    NSMutableString *str1 = [NSMutableString stringWithString:@"123"];
    NSString *str2 = [str1 copy];                     //深拷贝
    NSMutableString *str3 = [str1 mutableCopy];       //深拷贝
}
```

`NSMutableString`拷贝总结：

- 非容器类可变对象的`copy`、`mutableCopy`都是深拷贝。

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-07-014414.png" alt="image-20190107094413494" style="zoom:67%;" />

# 四、容器类对象

其实在上面，理解了非容器内对象拷贝的本质，或者对象拷贝的本质之后，你会发现只要遵循：

- 拷贝的目的：拷贝得到的目标对象与源对象互不影响；
- 拷贝方法：`copy`拷贝得到不可变对象，`mutableCopy`拷贝得到可变对象。

再从系统对对象的存储优化及性能方面出发，就能对拷贝这回事，理解的八九不离十。

- **两个不可变对象，是否有必要保留两份内存对象？**
- **Tagged Pointer技术什么时候使用，效率最大化？**

## 4.1 NSArray

针对容器对象，我们对容器本身，可以直接得知：copy是浅拷贝，mutableCopy是深拷贝的结论。主要看容器内的元素如何：

```objective-c
- (void)arrayCopy
{
    //copy返回不可变对象，mutablecopy返回可变对象
    //浅拷贝：arr2是和arr1同一个对象，其内部元素浅拷贝
    //深拷贝：arr3是arr1的可变副本，指向不同的对象，其内部元素浅拷贝
    NSArray *arr1 = [NSArray arrayWithObjects:[NSMutableString stringWithString:@"a"],@"b",@"c",nil];
    NSArray *arr2 = [arr1 copy];
    NSMutableArray *arr3 = [arr1 mutableCopy];
    //arr[0]: 0x1c00525a0, 0x1c00525a0, 0x1c00525a0     //浅拷贝
    NSLog(@"arr[0]: %p, %p, %p", arr1[0], arr2[0], arr3[0]);
    //arr[0] retainCount: 3, 3, 3
    NSLog(@"arr[0] retainCount: %ld, %ld, %ld ",
          [arr1[0] retainCount],
          [arr2[0] retainCount],
          [arr3[0] retainCount]);
}
```

最后我们发现，容器内元素不管是哪种拷贝，都是浅拷贝。

`NSArray`拷贝总结：

- 不可变对象的copy是浅拷贝，不可变对象的mutableCopy是深拷贝；
- 容器内元素都是浅拷贝。

## 4.2 NSMutableArray

遵循上面方式，我们直接如下有测试代码：

```objective-c
- (void)mutableArrayCopy
{
    //copy返回不可变对象，mutablecopy返回可变对象
    //深拷贝：arr2是和arr1是不同对象，其内部元素是浅拷贝
    //深拷贝：arr3也是和arr1不同对象，其内部元素是浅拷贝
    NSMutableArray *arr1 = [NSMutableArray arrayWithObjects:[NSMutableString stringWithString:@"a"],@"b",@"c", nil];
    NSArray *arr2 = [arr1 copy];
    NSMutableArray *arr3 = [arr1 mutableCopy];

    //arr[0]: 0x1c0052540, 0x1c0052540, 0x1c0052540
    NSLog(@"arr[0]: %p, %p, %p", arr1[0], arr2[0], arr3[0]);
    //arr[0] retainCount: 3, 3, 3
    NSLog(@"arr[0] retainCount: %ld, %ld, %ld ",
          [arr1[0] retainCount],
          [arr2[0] retainCount],
          [arr3[0] retainCount]);
}
```

`NSMutableArray`拷贝总结：

- 可变对象的copy，mutableCopy都是深拷贝；
- 容器内元素都是浅拷贝。

## 4.3 元素的深拷贝

针对容器对象的讨论似乎到此结束了，但是我们有个需求：如何对容器内的元素进行深拷贝。系统给我们提供了两种方案：

### 4.3.1 copyItems

调用容器类对象的copyItems方法：

```objective-c
//NSArray
- (instancetype)initWithArray:(NSArray<ObjectType> *)array copyItems:(BOOL)flag;
//NSDictionray
- (instancetype)initWithDictionary:(NSDictionary<KeyType, ObjectType> *)otherDictionary copyItems:(BOOL)flag;
```

如下：

```objective-c
BFPerson *person = [[BFPerson alloc] init];
NSMutableArray *strArr = [NSMutableArray arrayWithObjects:@"a", nil];
NSArray *arr1 = [NSArray arrayWithObjects:strArr,
                 [NSMutableString stringWithString:@"b"],person,@"c", nil];
NSArray *arr2 = [[NSArray alloc] initWithArray:arr1 copyItems:YES];
```

copyItems为YES，表示将对容器内元素最顶层结构的对象发送`copyWithZone:`消息，比如在上面只对`strArr`发送`copyWithZone:`，而其内部的元素`@"a"`并不会发送。

这样就会有一个问题，拷贝后获得的容器对象，其中的元素很大程度上依赖元素各自的`copyWithZone:`实现，因此可能会导致拷贝后，对象类型改变，比如容器内为`NSMutableArr`对象元素拷贝后，成了`NSArray`类型。

### 4.3.2 NSCoding协议

容器内的元素都实现NSCoding协议，并且通过NSKeyedArchiver归档对象，然后通过NSKeyedUnarchiver解档对象，能成功将对象进行全面的拷贝，不但是深拷贝，而且能保证对象类型不会变化。

```objective-c
BFPerson *person = [[BFPerson alloc] init];
NSArray *arr1 = [NSArray arrayWithObjects:[NSMutableArray arrayWithObjects:@"a", nil],
                 [NSMutableString stringWithString:@"b"],person, @"c",nil];
NSArray *arr2 = [NSKeyedUnarchiver unarchiveObjectWithData: [NSKeyedArchiver archivedDataWithRootObject:arr1]];
```

但是，该实现有个缺陷，就是成本过高，性能较差。

下面是NSCoding协议实现的一个示例：

```objective-c
@interface BFPerson : NSObject<NSCopying, NSCoding>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) NSMutableArray *data;
@end

@implementation BFPerson
//解档
- (instancetype)initWithCoder:(NSCoder *)aDecoder
{
    if (self = [super init]) {
        self.name = [aDecoder decodeObjectForKey:@"name"];
        self.age = [aDecoder decodeIntegerForKey:@"age"];
        self.data = [aDecoder decodeObjectForKey:@"data"];
    }
    return self;
}
//归档
- (void)encodeWithCoder:(NSCoder *)aCoder
{
    [aCoder encodeObject:self.name forKey:@"name"];
    [aCoder encodeInteger:self.age forKey:@"age"];
    [aCoder encodeObject:self.data forKey:@"data"];
}

@end
```

# 五、自定义对象的拷贝

如果自定义对象想要支持`copy`操作，需要将自定义类实现`NSCopying`协议，该协议有如下方法支持`copy`操作。

```objective-c
@protocol NSCopying
- (id)copyWithZone:(nullable NSZone *)zone;
@end
```

另外，要支持`mutabCopy`，需要实现`NSMutableCopying`的`-mutableCopyWithZone:`方法。

实现如下：

```objective-c
@implementation BFPerson
- (id)copyWithZone:(NSZone *)zone
{
    BFPerson *person = [[BFPerson alloc] init];
    person.name = self.name;
    person.age = self.age;
    return person;
}
@end
```

# 六、属性

作为`@property`内存管理语义的一个，`copy`的setter实现：

```objective-c
- (void)setName:(NSString *)name
{
    if (_name != name) {
        [_name release];
        _name = [name copy];
    }
}
```

此时，需要注意的一个问题是，如下声明：

```objective-c
@interface BFPerson : NSObject
@property (nonatomic, copy) NSMutableArray *data;
@end
```

声明了一个`NSMutableArray`的属性`data`，并且定义了`copy`，那么不管最后在使用的时候，传入的是可变还是不可变的数组，最后`data`都是不可变对象。使用时，注意不要产生崩溃。

```objective-c
// ViewController
- (void)copyProperty
{
    BFPerson *tom = [[BFPerson alloc] init];
    tom.name = @"tom";
    tom.age = 10;
    tom.data = [NSMutableArray array];
    //崩溃： -[__NSArray0 addObject:]: unrecognized selector sent to instance 0x1c4010960
    //@property (nonatomic, copy) NSMutableArray *data;
    //因为BFPerson类定义的是copy属性
    [tom.data addObject: @"a"];
}
```

解决之道，就是将`copy`声明改为`strong`。

```objective-c
@property (nonatomic, strong) NSMutableArray *data;
```