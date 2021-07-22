# OC 集合

添加到 iOS 集合内的对象必须是对象。
作为集合，需要满足三个基本的特性：**获取元素**、**查找元素**、**遍历元素**；
如果集合是可变的还需要另外支持 **添加元素** 和 **删除元素**。

## Array

添加到 `Array` 内的元素都是有序的，同一对象可多次被添加到集合。和其它集合相比，`Array` 遍历内部元素十分方便。`Array` 内元素必须是对象 （`NSPointArray` 除外）。

### Array 的分类

- **NSArray** 初始化后元素就不可再增删。

- **NSMuatbleArray** 初始化后可以随时添加或删除元素对象。

- **NSPointArray** 与 `NSMuatbleArray` 相似，区别是可以指定对元素的强/弱引用。也就是说 `NSPointArray` 内部元素为对象 或 `nil`。

  

  ![img](https://raw.githubusercontent.com/awanglilong/blog/main/uPic/16546f4355187763.png)

  

### 关于数组的一些知识点：

1. `indexOfObject` 与 `indexOfObjectIdenticalTo` 的区别？

两个方法都是来判断某一对象是否是 `Array` 集合内元素，如果是则返回该对象在集合内的索引。两个方法的区别就在于两个 API 判定的依据：`indexOfObject` 会使用 `isEqualTo` 方法将 `Object` 与集合元素进行比较。而 `indexOfObjectIdenticalTo` 则会比较 `Object` 与集合元素的指针是否相同。

1. 关于数组排序，单一条件的值比较排序使用 `SortedArrayUsingComparator` 或 `SortedArrayUsingSelector` 很容易，如果给你两个甚至多个排序条件呢？比如对学生分数排序，排序条件为，优先按总分降序排列，总分相同再比较数学成绩降序排列，数学成绩也相同比较语文成绩降序排列··· ··· 前面的两个排序方法当然也能满足我们的需求，但是现在要讨论的是另一种更优雅的实现：使用 `NSSortDescriptor`。

```
    Student *p1 = [[Student alloc] initWithName:@"zhonger" id:@"1" score:552 math:130 chinese：90 english：120];
    Student *p2 = [[Student alloc] initWithName:@"zhonger" id:@"1" score:552 math:130 chinese：90 english：120];
    Student *p3 = [[Student alloc] initWithName:@"zhonger" id:@"1" score:552 math:130 chinese：90 english：120];
    Student *p4 = [[Student alloc] initWithName:@"zhonger" id:@"1" score:552 math:130 chinese：90 english：120];
    Student *p5 = [[Student alloc] initWithName:@"zhonger" id:@"1" score:552 math:130 chinese：90 english：120];

    NSArray *students = @[p1, p2, p3, p4, p5];

    NSSortDescriptor *des1 = [[NSSortDescriptor alloc]initWithKey:@"score" ascending:NO];
    NSSortDescriptor *des2 = [[NSSortDescriptor alloc] initWithKey:@"math" ascending:NO];
    NSSortDescriptor *des3 = [[NSSortDescriptor alloc] initWithKey:@"chinese" ascending:NO];

    NSArray *newArray1 = [students sortedArrayUsingDescriptors:@[des1,des2,des3]];
    NSLog(@"%@",newArray1);
复制代码
```

是不是比之前的排序方法可读性提升了很多，而且代码后期也更好维护。

## Dictionary



![img](https://user-gold-cdn.xitu.io/2018/8/17/16546f4354e10372?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



`Dictionary` 内部元素以 `key:value` 键值对的形式存在，添加到 `Dictionary` 内的键值对是无序的，并且具有同一 `key` 值的键值对对象重复添加会相互覆盖。当某个键值对添加到 `Dictionary` 时，`Dictionary` 会对 `key` 进行深拷贝，而对 `value` 进行浅拷贝，已经添加到集合内部的键值对，key 值将不可更改。而 value 值为强引用，当 `value` 为可变对象时，修改 `value` 对象，字典内的 `value` 也会被修改。

`key` 为任何遵循 `NSCopying` 协议并实现了 `hash` 和 `isEqual` 方法的对象。
`value` 为任何对象（包括集合）

`key` 需要很好地实现哈希方法，否则字典的设值、查找、获取、增加等操作耗时将线性增加。一般我们都用 `NSString` 作为 `key` 值，是因为 `NSString` 很好地实现了哈希方法。这些操作的耗时为常量。

字典可以根据 `value` 值对 `key` 进行排序，排序后得到的是一个 `key` 数组。

### Dictionary 的分类

- **NSDictionary** 初始化后就不可再增删键值对。
- **NSMutableDictionary** 初始化后可以随时添加或删除键值对。
- **NSMapTable** 类似 `NSMutableDictionary` 可以指定对 `value` 的强/弱引用。也就是说 `NSMapTable` 内部元素的 `value` 值可以为 `nil`。



![img](https://user-gold-cdn.xitu.io/2018/8/17/16546f43fc34f008?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 关于字典的一些知识点：

我们在使用 `Dictionnary` 时，一般将 `String` 作为 `key` 值 ，从而可以满足绝大部分需求。但有时也会遇到使用自定义对象作为 key 的情况。那么作为 `key` 的对象都需要满足哪些条件呢？

1. `key` 必须遵循 `NSCopying` 协议，因为当元素渐入字典后，会对 `key` 进行深拷贝。
2. `key` 必须实现 `hash` 与 `isEqual` 方法。（便于快速获取元素并保证 `key` 的唯一性）。

> 并不建议使用特别大的对象作为 `key`，可能会导致性能问题。

## Set



![img](https://user-gold-cdn.xitu.io/2018/8/17/16546f43527796ac?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



添加到 Set 内的元素都是无序的，同一对象只能被添加一次。和其它集合相比，Set 查找内部元素十分快速。

> `set` 集合元素要很好地实现 `hash` 方法（时间复杂度为O(1)），否则时间复杂度为 O(k)

### Set 的分类

- **NSSet** 初始化后就不可再增删元素。
- **NSMuatbleSet** 初始化后可随时增删元素。
- **NSCountedSet** 每个元素都带有一个计数，添加元素，计数为一。重复添加某个元素则计数加一；移除元素计数减一，计数为零则移除
- **NSHashTable** 和 **NSMutableSet** 类似，区别是可指定对元素的强/弱引用，内部元素可以为 `nil`。



![img](https://user-gold-cdn.xitu.io/2018/8/17/16546f43fc041ebc?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



## Index set



![img](https://user-gold-cdn.xitu.io/2018/8/17/16546f434efc8862?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```
IndexSet` 保存的 `Array` 子集，存储元素为 `Array` 的 `index` 索引，存储形式为索引范围。 eg： `（0，3，4）->（NSRange(0，1),NSRange(3，2)）
```

### Index set 的分类

- **NSIndexSet** 初始化后就不可再增删元素。
- **NSMutableIndexSet** 初始化后可随时增删元素。

使用场景一般为记录数组的某些元素，比如列表多选行，可以使用 `NSIndexSet` 来记录选择了哪行，而不是另外创建一个可变数组