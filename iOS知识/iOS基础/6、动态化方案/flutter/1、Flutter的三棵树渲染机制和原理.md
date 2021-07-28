# Flutter的三棵树渲染机制和原理

Flutter是一个优秀的UI框架，借助它开箱即用的Widgets我们能够构建出漂亮和高性能的用户界面。那这些Widgets到底是如何工作的又是如何完成渲染的。

在本文中呢，我们就来探析Widgets背后的故事-Flutter渲染机制之三棵树。

## 目录

- 什么是三棵树？
- 三棵树的协同
- 三棵树的工作原理

## 什么是三棵树？

在Flutter中和Widgets一起协同工作的还有另外两个伙伴：Elements和RenderObjects；由于它们都是有着树形结构，所以经常会称它们为三棵树。

- Widget：Widget是Flutter的核心部分，是用户界面的不可变描述。做Flutter开发接触最多的就是Widget，可以说Widget撑起了Flutter的半边天；
- Element：Element是实例化的 Widget 对象，通过 Widget 的 createElement() 方法，是在特定位置使用 Widget配置数据生成；
- RenderObject：用于应用界面的布局和绘制，保存了元素的大小，布局等信息；

## 初次运行时的三棵树的

初步认识了三棵树之后，那Flutter是如何创建布局的？以及三棵树之间他们是如何协同的呢？接下来就让我们通过一个简单的例子来剖析下它们内在的协同关系：

```dart
class ThreeTree extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.red,
      child: Container(color: Colors.blue)
    );
  }
}
复制代码
```

上面这个例子很简单，它由三个Widget组成：ThreeTree、Container、Text。那么当Flutter的runApp()方法被调用时会发生什么呢？

当runApp()被调用时，第一时间会在后台发生以下事件：

- Flutter会构建包含这三个Widget的Widgets树；
- Flutter遍历Widget树，然后根据其中的Widget调用createElement()来创建相应的Element对象，最后将这些对象组建成Element树；
- 接下来会创建第三个树，这个树中包含了与Widget对应的Element通过createRenderObject()创建的RenderObject；

下图是Flutter经过这三个步骤后的状态：

![Flutter三棵树](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/733089b4d7b247138137c024259e8732~tplv-k3u1fbpfcp-zoom-1.image)

从图中可以看出Flutter创建了三个不同的树，一个对应着Widget，一个对应着Element，一个对应着RenderObject。每一个Element中都有着相对应的Widget和RenderObject的引用。可以说Element是存在于可变Widget树和不可变RenderObject树之间的桥梁。Element擅长比较两个Object，在Flutter里面就是Widget和RenderObject。它的作用是配置好Widget在树中的位置，并且保持对于相对应的RenderObject和Widget的引用。

### 三棵树的作用

简而言之是为了性能，为了复用Element从而减少频繁创建和销毁RenderObject。因为实例化一个RenderObject的成本是很高的，频繁的实例化和销毁RenderObject对性能的影响比较大，所以当Widget树改变的时候，Flutter使用Element树来比较新的Widget树和原来的Widget树：

```dart
//framework.dart
 @protected
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    Element newChild;
    if (child != null) {
      assert(() {
        final int oldElementClass = Element._debugConcreteSubtype(child);
        final int newWidgetClass = Widget._debugConcreteSubtype(newWidget);
        hasSameSuperclass = oldElementClass == newWidgetClass;
        return true;
      }());
      if (hasSameSuperclass && child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        newChild = child;
      } else if (hasSameSuperclass && Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        assert(child.widget == newWidget);
        assert(() {
          child.owner._debugElementWasRebuilt(child);
          return true;
        }());
        newChild = child;
      } else {
        deactivateChild(child);
        assert(child._parent == null);
        newChild = inflateWidget(newWidget, newSlot);
      }
    } else {
      newChild = inflateWidget(newWidget, newSlot);
    }

    assert(() {
      if (child != null)
        _debugRemoveGlobalKeyReservation(child);
      final Key key = newWidget?.key;
      if (key is GlobalKey) {
        key._debugReserveFor(this, newChild);
      }
      return true;
    }());

    return newChild;
  }
...
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
复制代码
```

- 如果某一个位置的Widget和新Widget不一致，才需要重新创建Element；
- 如果某一个位置的Widget和新Widget一致时(两个widget相等或runtimeType与key相等)，则只需要修改RenderObject的配置，不用进行耗费性能的RenderObject的实例化工作了；
  - 因为Widget是非常轻量级的，实例化耗费的性能很少，所以它是描述APP的状态（也就是configuration）的最好工具；
  - 重量级的RenderObject（创建十分耗费性能）则需要尽可能少的创建，并尽可能的复用；

> 看到这里你是否会觉得整个Flutter APP就像是一个RecycleView呢？

因为在框架中，Element是被抽离开来的，所以你不需要经常和它们打交道。每个Widget的build（BuildContext context）方法中传递的context就是实现了BuildContext接口的Element。

## 更新时的三棵树

因为Widget是不可变的，当某个Widget的配置改变的时候，整个Widget树都需要被重建。例如当我们改变一个Container的颜色为橙色的时候，框架就会触发一个重建整个Widget树的动作。因为有了Element的存在，Flutter会比较新的Widget树中的第一个Widget和之前的Widget。接下来比较Widget树中第二个Widget和之前Widget，以此类推，直到Widget树比较完成。

```dart
class ThreeTree extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.orange,
      child: Container(color: Colors.blue,),
    );
  }
}
复制代码
```

Flutter遵循一个最基本的原则：判断新的Widget和老的Widget是否是同一个类型：

- 如果不是同一个类型，那就把Widget、Element、RenderObject分别从它们的树（包括它们的子树）上移除，然后创建新的对象；
- 如果是一个类型，那就仅仅修改RenderObject中的配置，然后继续向下遍历；

在我们的例子中，ThreeTree Widget是和原来一样的类型，它的配置也是和原来的ThreeTreeRender一样的，所以什么都不会发生。下一个节点在Widget树中是Container Widget，它的类型和原来是一样的，但是它的颜色变化了，所以RenderObject的配置也会发生对应的变化，然后它会重新渲染，其他的对象都保持不变。

![Flutter三棵树](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25936d5b417e4a15a726735e888f1e99~tplv-k3u1fbpfcp-zoom-1.image)

> 注意这三个树，配置发生改变之后，Element和RenderObject实例没有发生变化。

上面这个过程是非常快的，因为Widget的不变性和轻量级使得他能快速的创建，这个过程中那些重量级的RenderObject则是保持不变的，直到与其相对应类型的Widget从Widget树中被移除。

### 当Widget的类型发生改变时

```dart
class ThreeTree extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.orange,
      child: FlatButton(
        onPressed: () {},
        child: Text('三棵树'),
      ),
    );
  }
}
复制代码
```

和刚才流程一样，Flutter会从新Widget树的顶端向下遍历，与原有树中的Widget类型进行对比。

![Flutter三棵树](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/427881c7330349bf99281f0f85d1e721~tplv-k3u1fbpfcp-zoom-1.image)

因为FlatButton的类型与Element树中相对应位置的Element的类型不同，Flutter将会从各自的树上删除这个Element和相对应的ContainerRender，然后Flutter将会重建与FlatButton相对应的Element和RenderObject。

![Flutter三棵树](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e03c24f452bf4f54a1b677535f36779b~tplv-k3u1fbpfcp-zoom-1.image)

当新的RenderObject树被重建后将会计算布局，然后绘制在屏幕上面。Flutter内部使用了很多优化方法和缓存策略来处理，所以你不需要手动来处理这些。

以上便是Flutter的整体渲染机制，可以看出Flutter利用了三棵树很巧妙的解决的性能的问题。

