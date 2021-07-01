# Runtime（二）方法缓存

在了解类的基本结构之后，本文开始了解探讨iOS 中的消息发送，即消息调用。

首先开始讨论的是——在真正消息调用之前，我们会去方法缓存里面寻找真实的函数地址，iOS提供的缓存机制用于提高效率。

# 一、方法的相关结构

## 1.1 回顾Class结构

不管讨论，什么离不开最基本的类结构，现在我们又回到`Class`结构，不过这次**需要关注的是方法**。

### 1.1.1 Class结构

![image-20181213214129678](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-134130-20210519103902340.png)

以上是在运行时使用到类的最终结果，那么在编译期，其实在之前的文章中也有提到，类结构略有不同，主要在于`bits`里指向的是`class_ro_t`，而非`class_rw_t`。

### 1.1.2 `class_ro_t`结构

<img src="https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-134152.png" alt="image-20181213214152399" style="zoom:50%;" />

上述的结果，呈现的是定义在类里面的方法，就是我们写在代码里**硬编码方法**，而**不是**通过以下方式添加的方法：

- 分类方法；
- 通过运行时添加的方法；

程序在运行时，会重新组织`bits`里结构的内容，获取`bits.data()`，即`class_rw_t`结构，该结构是运行时的结构。

### 1.1.3 `class_rw_t`

`class_rw_t`在程序开始运行后，会加载分类方法，会将分类方法重新组织成下面的结构：**二维数组**。

![image-20181213214238046](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-134238.png)

## 1.2 `method_t` 方法结构

根据上面的`class_rw_t`结构，我们可以清晰的观察到，在底层中方法的结构体是`method_t`，即一个方法对应一个`method_t`。

下面是`method_t`结构体的组成。

![image-20181213223155534](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-143156.png)

针对`method_t`的成员变量，上面阐述的很清楚。

- 该结构包含了函数指针，指向具体实现。
- 通过`types`来声明该方法的返回值及参数，用于底层调用实现的校验。
- `SEL`是方法名。

那么我们要调用一个函数，还需要解决两个问题：

**第一个问题：如何根据`SEL`找到函数实现地址`IMP`。**

**第二个问题：方法声明校验。**

### 1.2.1 Type Encoding

下面我们先讨论**第二个问题，方法声明校验**，这个校验，是通过给方法指定一个编码实现的，相对应的编码如下：

![image-20181213223037331](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-143037.png)

# 二、方法缓存

我们继续讨论上面的**第一个问题——如何根据`SEL`找到函数实现地址`IMP`。**

在无缓存时，找到`isa`指向的类结构，遍历`class_rw_t`中的方法列表`method_array_t`即可。

那么有缓存的时候呢？

## 2.1 窥探方法缓存

我们要窥探缓存的结构，从[源码](https://opensource.apple.com/tarballs/objc4/)读起。下面是源码的的顺序图：

![image-20181213222821800](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-142822.png)

在对源码的剖析之后，我们有以下的成果：

### 2.1.1 方法缓存结构

其中`bucket_t *_buckets`就是存放缓存列表的结构，它本质是一个哈希表。

而哈希表中的存放的是`bucket_t`的结构体。

![image-20181213223301620](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-143302.png)

### 2.1.2 验证

验证的代码在[01方法缓存探索](https://github.com/wenghengcong/LearnCollection/tree/master/Runtime/01方法缓存探索)。

![image-20181213231248316](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-151248.png)

## 2.2 哈希表

通过代码的探索，我们整理了下面这些哈希表中**最重要的节点处理**。

![image-20181213223414603](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/2018-12-13-143415.png)

### 2.2.1 哈希表的处理

上面有一些需要注意的点：

#### （1）hash函数

`mask`为缓存空间大小-1，所以hash之后一定不会超过缓存空间大小；

```c++
static inline mask_t cache_hash(cache_key_t key, mask_t mask) 
{
    return (mask_t)(key & mask);
}
```

#### （2）碰撞处理

碰撞的处理因平台而已，在iOS下做了如下处理，其实就是简单的**开放寻址**来进行碰撞处理：

```c++
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
```

#### （3）缓存扩容

缓存空间是会动态变化的，其变化如下：

```c++
void cache_t::expand()
{
    uint32_t oldCapacity = capacity();
    //假如旧的容量大小为0，就分配4（INIT_CACHE_SIZE）
    //旧容量大小不为0，分配当前容量的2倍
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;
    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        newCapacity = oldCapacity;
    }
    reallocate(oldCapacity, newCapacity);
}
```

#### （4）注意点

- `mask`为缓存空间大小-1，所以hash之后一定不会超过缓存空间大小；

### 2.2.2 读写缓存

#### （1） 缓存方法

我们看看方法缓存的哈希表，是如何存放方法的。

1. 传入key（**@selector(method)**）通过hash——**cache_hash**获得索引`index`；
2. 检查当前index是否被占用
   - 如果被占用，即本次哈希冲突，重新进行寻址——**cache_next**，算出index，回到2。
3. 如果没占用，存放到`index`处。

#### （2）查找缓存

那么又是如何查找缓存的呢？

1. 传入key（**@selector(method)**）通过hash——**cache_hash**获得索引`index`；

2. 根据

   ```
   index
   ```

   获取

   ```
   bucket_t
   ```

   ，检查该

   ```
   bucket._key
   ```

   是否与传入的key一致。

   - 若不一致，即由于存方法时，有hash冲突，再次hash——`cache_next`，算出index，回到2。

3. 如果key一致性通过，即获取该`bucket._imp`返回；

4. 调用`bucket._imp`处的函数。

### 2.2.3、冲突的处理方法

#### 1、开放定址法（建立闭散列表）

开放定址指散列表的地址对任何记录数据都是开放的，即可存储使用。但散列表长度一旦确定，总的可用地址是有限的。闭散列表表长不小于所需存储的记录数，发生冲突总能找到空的散列地址将其存入。查找时，按照一种固定的顺序检索散列表中的相应项，直到找到一个关键字等于k或找到一个空单元将k插入，故是动态查找结构。

1）线性探测法

从发生冲突位置的下一个位置开始寻找空的散列地址。发生冲突时，线性探测下一个散列地址是：Hi=(H(key)+di)%m，（di=1,2,3...,m-1）。闭散列表长度为m。它实际是按照H(key)+1，H(key)+2,...,m-1,0,1,H(key)-1的顺序探测，一旦探测到空的散列地址，就将关键码记录存入。

该方法会产生堆积现象，即使是同义词也可能会争夺同一个地址空间，今后在其上的查找效率会降低。

2）二次探测法

发生冲突时，下一位置的探测采用公式：Hi=(H(key)+di)%m，(di=1^2,-1^2,2^2,-2^2,.....,q^2,-q^2,q<=根号下m)

在一定程度上可解决线性探测中的堆积现象。

3）随机探测法

di为{1,2,3,...,m-1}中的数构成的一个随机数列中顺序取的一个数

4）再散列函数法

除基本散列函数外，事先设计一个散列函数序列，RH1,RH2,...,RHk，k为某个正整数。RHi均为不同的散列函数。对任一关键码，若在某一散列函数上发生冲突，则再用下一个散列函数，直到不发生冲突为止。

5）建立公共溢出区（单链表或顺序表实现）

另外开辟一个存储空间，当发生冲突时，把同义词均顺序放入该空间。若把散列表看成主表或父表，则公共的同义词表就是一个次表或子表。查找时，现在散列表中查，找不到时再去公共同义词子表顺序查找。

#### 2、拉链法（链地址法、建立开散列表）

将所有散列地址相同的记录存储在同一个单链表中，该单链表为同义词单链表，或同义词子表。该单链表头指针存储在散列表中。散列表就是个指针数组，下标就是由关键码用散列函数计算出的散列地址。初始，指针数组每个元素为空指针，相当于所有单链表头指针为空，以后每扫描到一条记录，按其关键码的散列地址，在相应的单链表中加入含该记录的节点。开散列表容量可很大，仅受内存容量的限制。




# 三、总结

## 3.1 方法缓存

1. 每个类对象都存有一个cache——方法缓存列表；
2. cache本质是哈希表，其hash函数为：f(@selector()) = @selector() & _mask；
3. 子类没有实现方法会调用父类的方法，并且将父类方法加入到子类自己的cache 里。