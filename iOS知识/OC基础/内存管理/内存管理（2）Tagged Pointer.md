# 内存管理（二）Tagged Pointer

本文主要研究Tagged Pointer技术，针对该技术需要解决的问题、以及在实际应用中的价值做一些简单的探讨。

**另外，本文中涉及到的示例代码，请在真机iOS设备上测试**，因为Tagged Pointer技术针对不同的平台，具体实现细节是有差异的，否则无法得出和本文一致的测试结果。

# 一、对象的内存

下面我们针对iOS中对象进行一些探究，代码如下。

```objective-c
__weak NSNumber *weakNumber;
__weak NSString *weakString;
__weak NSDate *weakDate;
__weak NSObject *weakObj;
int num = 123;

@autoreleasepool {
    weakObj = [[NSObject alloc] init];
    weakNumber = [NSNumber numberWithInt:num];
    weakString = [NSString stringWithFormat:@"string%d", num];
    weakDate   = [NSDate dateWithTimeIntervalSince1970:0];
}
NSLog(@"weakObj is %@", weakObj);
NSLog(@"weakNumber is %@", weakNumber);
NSLog(@"weakString is %@", weakString);
NSLog(@"weakDate is %@", weakDate);
```

在**第7行**，首先定义了4个`__weak***`对象，构建了一个autoreleasepool，所以在**12行之后**，所有`__weak`修饰的弱引用对象，都会被释放。经过上面分析，我们得出，对象会打印出`null`。

但是，实际上，我们得到了如下的输出。

> **TaggedPointer[3570:3928309] weakObj is (null)**
>
> **TaggedPointer[3570:3928309] weakNumber is 123**
>
> **TaggedPointer[3570:3928309] weakString is string123**
>
> **TaggedPointer[3570:3928309] weakDate is Thu Jan 1 08:00:00 1970**

可以看到，**只有NSObject对应的对象值是null，其他的值，均正常打印。**

这是因为`NSNumber`、`NSString`、`NSDate`，在这里采用了**Tagged Pointer**技术。

# 二、Tagged Pointer

## 2.1 Tagged Pointer技术

### 2.1.1 简介

![image-20190103130446423](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/20190603201832.png)

### 2.2.2 未引入Tagged Pointer

![image-20190103130510065](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-050510.png)

### 2.2.3 引入Tagged Pointer

![image-20190103130534599](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-050534.png)

### 2.2.4 判断是否是Tagged Pointer

![image-20190103144358980](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-064359.png)

## 2.2 应用

### 2.2.1 支持的对象类型

可以从[objc源码](https://opensource.apple.com/tarballs/objc4/)中找出支持Tagged Pointer 的对象类型，如下：

```objective-c
typedef uint16_t objc_tag_index_t;
enum
{
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSDate            = 6, 
    ....
};
```

即针对`NSString`、`NSNumber`、`NSDate`、`NSIndexPath`这些类型，都支持Tagged Pointer技术。

### 2.2.2 NSNumber

我们通过`NSNumber`以及`NSString`对象来观察**Tag+Data**的存储形式。

如下所示，我们创建了很多`NSNumber`对象：

```objective-c
NSNumber *number1 = @1;                          //0xb000000000000012
NSNumber *number2 = @2;                          //0xb000000000000022
NSNumber *number3 = @(0xFFFFFFFFFFFFFFF);        //0x1c0022560
NSNumber *number4 = @(1.2);                      //0x1c0024b80
int num4 = 5;
NSNumber *number5 = @(num4);                     //0xb000000000000052
long num5 = 6;
NSNumber *number6 = @(num5);                     //0xb000000000000063
float num6 = 7;
NSNumber *number7 = @(num6);                     //0xb000000000000074
double num7 = 8;
NSNumber *number8 = @(num7);                     //0xb000000000000085

//值：0xb000000000000012 0xb000000000000022 0x1c0022560 0x1c0024b80 0xb000000000000052 0xb000000000000063 0xb000000000000074 0xb000000000000085
NSLog(@"%p %p %p %p %p %p %p %p", number1, number2, number3, number4, number5, number6, number7, number8);
```

由上表我们得出：

- 很大的数字，超过Tagged Pointer表示上限的时候，将会转为对象存储，存放在堆上；
- 如果是含有小数点的浮点数，将会直接以对象方式存储；
- 其余类型的数字，包括不含小数部分的浮点型和整型都会以Tagged Pointer存储。

并且，针对以上部分，我们整理出**Tagged Pointer**的存储格式如下，以number1为例：

![image-20190103145741152](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-065741-20210520161405033.png)

### 2.2.3 NSString

同上面`NSNumber`的处理逻辑，`NSString`处理的类似。

```objective-c
NSString *str1 = @"a";                                          //0x1049cc248
NSString *str2 = [NSString stringWithFormat:@"a"];              //0xa000000000000611
NSString *str3 = [NSString stringWithFormat:@"bccd"];           //0xa000000646363624
NSString *str4 = [NSString stringWithFormat:@"c"];              //0xa000000000000631
NSString *str5 = [NSString stringWithFormat:@"cdasjkfsdljfiwejdsjdlajfl"];//0x1c02418f0
NSLog(@"%@ %@ %@ %@ %@",
      [str1 class],   //__NSCFConstantString
      [str2 class],   //NSTaggedPointerString
      [str3 class],   //NSTaggedPointerString
      [str4 class],   //NSTaggedPointerString
      [str5 class]);  // __NSCFString
```

根据以上结果，我们将NSString分类三类：

- 常量类型：__NSCFConstantString，定义的字符串常量。
- Tagged Pointer类型：NSTaggedPointerString，通过对象方法创建的短字符串。
- NSString对象类型：__NSCFString，包括NSString、NSMutableString等创建的字符串对象。

以上，整理如下：

![image-20190103150923946](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-070924.png)

NSString以Tagged Pointer的存储格式如下：

![image-20190103151645748](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-071646.png)

## 2.3 内存管理

![image-20190103151136639](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-071136.png)

# 三、一个面试问题的研究

该面试题如下：

![image-20190103151957866](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2019-01-03-071958.png)

