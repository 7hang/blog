# Block（一）使用

# 一、Block入门

 **Block**是封装工作单元的对象，即可在任何时间执行的代码段。它们在本质上是可移植的匿名函数，可作为方法和函数的参数传入，或可从方法和函数中返回。Block本身具有一个已类型化的参数列表，且可能具有推断或声明的返回类型。您还可以将块赋值给变量，然后像调用函数一样调用它。

 插入符号 (^) 是用作块的语法标记。块的参数、返回值和正文（即执行的代码）存在其他类似的语法约定。

 Block共享局部作用域的数据。Block的这项特征非常有用，因为如果您实现一个方法，并且该方法定义一个块，则该块可以访问该方法的局部变量和参数（包括堆栈变量），以及函数和全局变量（包括实例变量）。这种访问是只读的，但如果使用 `__block` 修饰符声明变量，则可在Block内更改其值。即使包含有块的方法或函数已返回，并且其局部作用范围已销毁，但是只要存在对该块的引用，局部变量仍作为块对象的一部分继续存在。

 作为方法或函数参数时，Block可用作回调。被调用时，方法或函数执行部分工作，并在适当时刻，通过Block回调正在调用的代码，以从中请求附加信息，或获取程序特定行为。Block使调用方在调用时能够提供回调代码。Block从相同的词法作用范围内采集数据（就像宿主方法或函数所做的那样），而非将所需数据打包在“关联”结构中。由于Block代码无需在单独的方法或函数中实现，您的实施代码会更简单且更容易理解。



# 二、语法

![block语法](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-04-093443.png)



## 1. 局部变量

```objective-c
//returnType (^blockName)(parameterTypes) = ^returnType(parameters) {...};
void (^callBack)(NSString *) = ^void(NSString *print){
    NSLog(@"%@",print);
};

callBack(@"we are family!");
```

## 2. 属性

```objective-c
//@property (nonatomic, copy, nullability) returnType (^blockName)(parameterTypes);
@property (copy ,nonatomic)void (^callBack)(NSString *);
```

## 3. 函数参数

```objective-c
//- (void)someMethodThatTakesABlock:(returnType (^nullability)(parameterTypes))blockName;
- (void)callbackAsAParameter:(void (^)(NSString *print))callBack {
	callBack(@"i am alone");
}

//[someObject someMethodThatTakesABlock:^returnType (parameters) {...}];
[self callbackAsAParameter:^(NSString *print) {
    NSLog(@"here is %@",print);
}];
```

## 4. typedef

```objective-c
//typedef returnType (^TypeName)(parameterTypes);
//TypeName blockName = ^returnType(parameters) {...};

typedef void (^callBlock)(NSString *status);

CallBlock block = ^void(NSString *str){
    NSLog(@"%@",str);
};
```