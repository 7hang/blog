# Flutter 核心渲染流程分析 [完结篇]

> 本文应是终点，亦是起点。

## 导语

作为一个 Flutter 开发者，我们仅需组合 widget 即可实现各种不同的交互。这其中 Flutter 是如何通过 widget 完成屏幕上的呈现？Native 层起到了怎样的作用？作为 UI 核心渲染流程的完结篇，这次我们不陷于每一个细节，而是从全局梳理主要流程，把握关键节点。对于细节的知识点，我会附上一些更深入的文章。希望能对你认识 Flutter 渲染机制有所帮助。

------

## 一、缘起：setState()

无论是一个简单的列表，还是一段灵巧的动画，本质都是由一个个快速切换的画面组成，术语称其为「帧」。 当我们在滑动列表，或者播放动画的时候。系统不断地生产帧，并将其绘制到屏幕上，呈现出「动」起来了的感觉。

![12dd158defe40ae707f09f613c0514cc.jpeg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04fc87e6c5df47b9854b83e05f807a78~tplv-k3u1fbpfcp-watermark.image)

而在 Flutter 中当需要更新 UI 展示的时候，我们第一时间往往想到 `setState()`。更新 UI 本质上，不就是用一个新的「帧」去替换上一个「帧」么。所以，其中必定会执行帧的调度逻辑。而 `setState` 最终调用到 `BuildOwner.scheduleBuildFor`。

```dart
  /// dart
  /// Adds an element to the dirty elements list so that it will be rebuilt
  /// when [WidgetsBinding.drawFrame] calls [buildScope].
  void scheduleBuildFor(Element element) {
    .......
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
    ........
   }
  
复制代码
```

方法中有两个关键点：

> 1、onBuildScheduled()
>
> 2、将 element 添加到 `_dirtyElements` 中

第二点没什么好说，后面会用到，关键先看第一点。跟踪引用，会发现第一个方法最终会执行到 SchedulerBinding.scheduleFrame()，这便是绘制的源头。

------

## 二、渲染起源：SchedulerBinding.scheduleFrame()

```dart
  /// dart
  /// 调用 C++ 到 Native 层，请求 Vsync 信号
  void scheduleFrame() {
    if (_hasScheduledFrame || !_framesEnabled)
      return;
     ensureFrameCallbacksRegistered();
     window.scheduleFrame();
    _hasScheduledFrame = true;
  }
复制代码
```

这个方法代码量并不多，关键在 `window.scheduleFrame()` ，这是一个 native 方法。

```dart
  void scheduleFrame() native 'Window_scheduleFrame';
复制代码
```

以安卓为例，最终会执行到 JNI_OnLoad 注册的 Java 接口 `AsyncWaitForVsyncDelegate.asyncWaitForVsync`，这个接口在 Flutter 启动时初始化。实现内容如下

```java
new FlutterJNI.AsyncWaitForVsyncDelegate() {
        @Override
        public void asyncWaitForVsync(long cookie) {
          Choreographer.getInstance()
              .postFrameCallback(
                  new Choreographer.FrameCallback() {
                    @Override
                    public void doFrame(long frameTimeNanos) {
                      float fps = windowManager.getDefaultDisplay().getRefreshRate();
                      long refreshPeriodNanos = (long) (1000000000.0 / fps);
                      FlutterJNI.nativeOnVsync(
                          frameTimeNanos, frameTimeNanos + refreshPeriodNanos, cookie);
                    }
                  });
        }
      }
复制代码
```

> 初始化可以看：[深入理解Flutter引擎启动](https://link.juejin.cn/?target=http%3A%2F%2Fgityuan.com%2F2019%2F06%2F22%2Fflutter_booting%2F)

`Choreographer.getInstance().postFrameCallback` 用于监听系统垂直同步信号，在下一个垂直信号来临时回调 `doFrame`，通过 `FlutterJNI.nativeOnVsync` 走到 c++ 中。经过复杂的链路，将下面的任务添加到到 UI Task Runner 中的事件队列中：

```C++
lib/ui/window/platform_configuration.cc

void PlatformConfiguration::BeginFrame(fml::TimePoint frameTime) {
  .................
  // 调用 dart 中的 _window.onBeginFrame
  tonic::LogIfError(
      tonic::DartInvoke(begin_frame_.Get(), {Dart_NewInteger(microseconds),}));
  // 执行 microTask                                          
  UIDartState::Current()->FlushMicrotasksNow();
  // 调用 dart 中的 _window.onDrawFrame
  tonic::LogIfError(tonic::DartInvokeVoid(draw_frame_.Get()));
}
复制代码
```

> 回调的初始化流程可看：[Flutter渲染机制—UI线程](https://link.juejin.cn/?target=http%3A%2F%2Fgityuan.com%2F2019%2F06%2F15%2Fflutter_ui_draw%2F)（文章基于 sdk 版本较早，部分函数位置有所修改）

这个方法有三个主要流程：

1、执行 `window.onBeginFrame`，这个方法映射到 dart 中的 `handleBeginFrame()`

2、接着执行 MicroTask 队列（**所以 MicroTask 不仅只在某个 Event 执行后被调度**）

3、最后执行 `window.onDrawFrame`，对应 dart 中的 `drawFrame()`

2 很好理解，就是执行 MicroTask 下面我们主要分析 1 和 3。

> 总结：整个流程由 Dart 发起，通过 C++ 回调到 Native 层，注册一次垂直同步信号的监听。等到信号来到，再通知 Dart 进行渲染。可以看出，Flutter 上的渲染，是先由 Dart 侧主动发起，而不是被动等待垂直信号的通知。这可以解释，比如一些静态页面时，整个屏幕不会多次渲染。并且由于是 Native 层的垂直同步信号，所以也完全适配高刷的设备。

------

## 三、Flutter 调度生命周期

分析 `handleBeginFrame` 和 `drawFrame` 其实并不复杂，从 Flutter 的调度状态并可掌握。

在 SchedulerBinding 中定义五种调度状态的枚举（按先后顺序排列）：

| 枚举值                              | 说明                                                         |      |
| ----------------------------------- | ------------------------------------------------------------ | ---- |
| SchedulerPhase. transientCallbacks  | handleBeginFrame 触发，执行`_transientCallbacks`集合中的任务。顾名思义，`_transientCallbacks` 是一个临时性的集合，往往通过 Ticker 向里面添加一些动画任务。 |      |
| SchedulerPhase. midFrameMicrotasks  | 这个状态的扭转在 handleBeginFrame 中，但实际的任务是，由 C++ 调度 MicroTask（上面的 2）。 |      |
| SchedulerPhase. persistentCallbacks | handleDrawFrame 触发，执行 `_persistentCallbacks` 集合中的任务。这些任务是一帧绘制所必须的，例如 build、layout、paint。 |      |
| SchedulerPhase. postFrameCallbacks  | handleDrawFrame 触发，在上一个的任务集合执行完成后，执行 `_postFrameCallbacks`集合中的任务。由于系统已经 layout ，所以可以在这个阶段获取 widget 的尺寸信息（RenderBox、RenderSliver）。 |      |
| SchedulerPhase.idle                 | handleDrawFrame 触发，所有的任务都已执行完成，处于空闲阶段。 |      |

一帧的调度会经过五种状态的扭转，我们再结合来看 `handleBeginFrame` 和 `drawFrame`。

### handleBeginFrame：处理渲染前的任务

这个方法中主要处理一些临时性的任务，比如动画。因为例如我们做一个 widget 放大的动画，实质上就是在每个垂直信号来临的时候，计算其对应的大小状态并且将其绘制。可能有的页面有动画，有时候没有。所以这是一个「临时性」的任务集合。对应会扭转到生命周期中的 `SchedulerPhase. transientCallbacks` 和 `SchedulerPhase. midFrameMicrotasks` 状态。

> 动画回调过程可以看：[Flutter AnimationController回调原理](https://juejin.cn/post/6886092037288886286)

### drawFrame：核心渲染流程

这里是渲染的核心流程，通过源码注释可以知道，一帧渲染一共有 10 步：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4f7d851b807477badc912b1e2c5a8fa~tplv-k3u1fbpfcp-watermark.image)

其中 1 和 2 在 `handleBeginFrame` 已经提到，在剩下的步骤中，和渲染关联比较紧密的主要有三步:

- Build：还记得一开始 setState() 将 element 加入了脏集合么？这个阶段，**Flutter 会通过 widget 更新所有脏集合中的节点（需要更新）中的 element 与 RenderObject 树**。

> Build 阶段细节：[原来我一直在错误的使用 setState()?](https://juejin.cn/post/6905996819445055495) 以及 [面试官问我State的生命周期，该怎么回答](https://juejin.cn/post/6908574202253541389)

- Layout：RenderObject 树进行布局测量，用于确定每一个展示元素的大小和位置。

> Layout 阶段细节：[总结了30个例子之后，我悟到了Flutter的布局原理](https://juejin.cn/post/6914155427651387399) 以及 [花两天时间做了15个例子的解析，彻底掌握Flutter的布局原理](https://juejin.cn/post/6916157915590033421)

- Paint：Paint 阶段会触发 RenderObject 对象绘制，生成第四棵树：Layer Tree，最终合成光栅化后完成渲染。

> Paint阶段细节：[张风捷特烈](https://juejin.cn/user/149189281194766/posts)，捷特大佬，绘制的神，推荐买小册看看。

------

## 总结

关键点：

1、`scheduleFrame` 回调 C++ 注册 Vsync 信号

2、Vsnc 来临，回调 C++ 调用 `window.onBeginFrame`，`window.onDrawFrame`

3、`drawFrame` 中，build 阶段根据 widget 生成 element 和 renderObject 树；layout 测量绘制元素的大小和位置；paint 阶段生成 layer 树；

4、理解调度生命周期每个状态的意义与作用

整个流程如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f49d75b74a3243acaab55832003a0b28~tplv-k3u1fbpfcp-watermark.image)

------

## 最后

这是 UI 机制专栏的最后一篇，但就如开头所说，我觉得这既是最后一篇，也可以是第一篇。不拘泥于细节，先从整体学习思想。而后抽丝剥茧，学习其中设计的技巧，我认为是比较好的学习方式。