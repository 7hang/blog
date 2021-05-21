# 内存管理（三）MRC与ARC

本篇主要讲述如何在开发中自如的切换MRC与ARC，虽然MRC项目以及很少存在，但是了解其本质，也就是ARC。

# 一、MRC

## 1.1 方法

MRC内存管理，在调用相关内存方法如下表：

![image-20190107004403705](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-164404.png)

## 1.2 内存管理原则

那么如何使用调用上面这些方法呢？需要根据内存的管理原则，有如下4条：

![image-20190106205023449](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-125024.png)

- 自己生成的对象，自己持有

  ```objective-c
  - (void)rule1
  {
      /*
       * 自己生成并持有该对象
       */
      id obj0 = [[NSObject alloc] init];
      id obj1 = [NSObject new];
  }
  ```

- 非自己生成的对象，自己也能持有

  ```objective-c
  - (void)rule2
  {
      /*
       * 持有非自己生成的对象
       */
      id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
      [obj retain]; // 自己持有对象
  }
  ```

- 不再需要自己持有的对象时释放

  ```objective-c
  - (void)rule3
  {
      /*
       * 不在需要自己持有的对象的时候，释放
       */
      id obj = [[NSObject alloc] init]; // 此时持有对象
      [obj release]; // 释放对象
      /*
       * 指向对象的指针仍就被保留在obj这个变量中
       * 但对象已经释放，不可访问
       */
  }
  ```

- 非自己持有的对象无法释放

  ```objective-c
  - (void)rule4
  {
      /*
       * 非自己持有的对象无法释放
       */
      id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
      // 此时将运行时crash 或编译器报error
      // MRC 下，调用该方法会导致编译器报issues。此操作的行为是未定义的，可能会导致运行时crash或者其它未知行为
      [obj release];
  }
  ```

   其中非自己生成的对象，且该对象存在，但自己不持有 这个特性是使用autorelease来实现的，示例代码如下：

  ```objective-c
  - (id) getAObjNotRetain {
     id obj = [[NSObject alloc] init]; // 自己持有对象
     [obj autorelease]; // 取得的对象存在，但自己不持有该对象
     return obj;
  }
  ```

  `autorelease` 使得对象在超出生命周期后能正确的被释放(通过调用release方法)。在调用 `release` 后，对象会被立即释放，而调用 `autorelease` 后，对象不会被立即释放，而是注册到 `autoreleasepool` 中，经过一段时间后 `pool`结束，此时调用`release`方法，对象被释放。

  

## 1.3 setter方法

 在了解了如何调用方法之后，我们仍然需要对`@property`做一定的了解，虽然MRC下，Xcode也会为我们自动生成`setter`。现在看下它们是如何实现的。

### 1.3.1 内存管理语义的属性特质

在MRC下，有如下几个：

![image-20190106210249107](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-130249.png)

### 1.3.2 实现

先给出一个示例类，示例代码见-[MRC-引用计数](https://github.com/wenghengcong/LearnCollection/tree/master/内存管理/MRC-引用计数)

```objective-c
@interface BFPerson : NSObject

@property (nonatomic, assign) NSInteger age;

@property (nonatomic, copy) NSString *name;

@property (nonatomic, retain) BFBook *book;

@property (nonatomic, unsafe_unretained) BFPen *pen;

@end
```

- assign

  针对纯量类型，使用assign，并且因为没有涉及到引用计数，其setter为简单的赋值操作。

  ```objective-c
  - (void)setAge:(NSInteger)age
  {
      _age = age;
  }
  
  -(NSInteger)age
  {
      return _age;
  }
  ```

- retain

  针对对象类型，就需要管理对象的引用计数，`retain`的含义及时要表明拥有权，即强引用。此时需要对传递进来的对象进行`retain`，增加引用计数，代表有新指针强引用它。

  至于在`retain`之前，为什么要进行`release`，可以看示例代码一探究竟。

  ```objective-c
  - (void)setBook:(BFBook *)book
  {
      if (_book != book) {
          [_book release];
          _book = [book retain];
      }
  }
  
  - (BFBook *)book
  {
      return _book;
  }
  ```

- copy

  `copy`的操作和`retain`非常相似。但是对象进行`copy`，至于`copy`后是创建新对象，还是在传递进来的原有对象上进行引用计数加1，就需要视情况而定.

  ```objective-c
  - (void)setName:(NSString *)name
  {
      if (_name != name) {
          [_name release];
          _name = [name copy];
      }
  }
  
  - (NSString *)name{
      return _name;
  }
  ```

- unsafe_unretained

  `unsafe_unretained`和`assign`类似，但是可以用在对象类型上，它不会影响传递进来的对象引用计数，只是进行简单的赋值。

  ```objective-c
  - (void)setPen:(BFPen *)pen
  {
      if (_pen != pen) {
          _pen = pen;
      }
  }
  
  - (BFPen *)pen
  {
      return _pen;
  }
  ```

## 1.4 应用

忘记release，可能导致内存泄漏。

```objective-c
- (void)test
{
    // 内存泄漏：该释放的对象没有释放
    BFPerson *person = [[BFPerson alloc] init];
}
```

下面是一个更为复杂的使用：

```objective-c
- (void)test2
{
    BFBook *book = [[BFBook alloc] init];   //book: 1
    BFPerson *person1 = [[BFPerson alloc] init];  //person: 1
    
    person1.book = book;     //book: 2
    person1.book = book;     //book: 2
    person1.book = book;     //book: 2
    
    NSLog(@"%ld", [book retainCount]);
    [book release];     //book: 1
    [person1 release];  //book: 0
}
```

# 二、ARC

ARC，Automatic Reference Count ，自动引用计数，是MRC的体面版，OC开发者不再像C开发者那样，对指针怨念幽深了，对引用计数的操作由ARC自动管理。

ARC帮我们做了一些工作，主要分为两部分：

- 编译期：在编译时，编译器会分析源码中每个对象的生命周期，然后基于这些对象的生命周期，来添加相应的引用计数操作代码。
- `Runtime`和`RunLoop`：这两者会协同工作，以保证弱引用以及自动释放池的工作顺利进行，并对程序进行一定的优化。

## 2.1 __strong

![image-20190106213818566](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-133819.png)

## 2.2 __weak

![image-20190106213858672](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-133859.png)

## 2.3 __unsafe_unretained

![image-20190106213918181](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-133918.png)

## 2.4 `@perperty`

属性的本质其实仍然没有改变，因为内存管理的根基——引用计数没有变。

# 三、Core Foundation

就算用上了ARC，无处不在的ARC，但是仍然由鞭长莫及的地方——Core Foundation。

`Core Foundation`框架中的`CF对象`，以及我们常用的`Foundation`中的`NS对象`，由Toll-free briding技术进行转换。

![image-20190106214531792](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-06-134532.png)

下面是进行转换的示例：

```objective-c
- (void)testCFObject
{
    //1.__bridge_retained
    //等价于CFBridgingRetain函数，将一个OC指针转换为一个CF指针，同时移交所有权，需要手动调用CFRelease来释放CF对象
    NSString *s1 = [[NSString alloc] initWithFormat:@"HelloHelloHelloHelloHello"];
    CFStringRef s2 = (__bridge_retained CFStringRef)s1;
    // or CFStringRef s2 = (CFStringRef)CFBridgingRetain(s1)
    // do something with s2
    //这一行必须要，不然会内存泄漏
    CFRelease(s2);
    
    //2.__bridge_transfer
    //等价于CFBridgingRelease，将一个CF指针转换为OC指针，同时移交所有权，ARC负责管理这个OC指针的生命周期。
    CFStringRef s3 = CFStringCreateWithCString(kCFAllocatorDefault, "abc", kCFStringEncodingEUC_CN);
    NSString *s4 = (__bridge_transfer NSString *)s3;
    NSLog(@"%ld",(long)s4.length);
    CFRelease(s3);
    
    //3.__bridge
    //进行OC指针和CF指针之间的转换，不涉及对象所有权转换，原CF对象需手动调用CFRelease释放对象。
    NSString * s5 = [NSString stringWithFormat:@"%ld",random()];
    CFStringRef s6 = (__bridge CFStringRef)s5;
    NSLog(@"%ld",(long)CFStringGetLength(s6));
    
    CFStringRef s7 = CFStringCreateWithFormat (NULL, NULL, CFSTR("%d"), rand());
    NSString *s8 = (__bridge NSString *)s7;
    NSLog(@"%ld",(long)s8.length);
    //这一行必须要，不然会内存泄漏
    CFRelease(s7);
}
```

# 四、混编

Xcode支持MRC与ARC混编，只要在对应的源文件指定即可，指定的关键字分别是：**-fno-objc-arc**和**-fobjc-arc**。

