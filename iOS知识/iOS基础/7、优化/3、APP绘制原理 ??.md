# APP绘制原理

通过 [图形渲染原理](http://chuquan.me/2018/08/26/graphics-rending-principle-gpu) 一文，大致能够了解图形渲染过程中硬件相关的原理。本文将进一步介绍 iOS 开发过程中图形渲染原理。

# 图形渲染技术栈

下图所示为 iOS App 的图形渲染技术栈，App 使用 `Core Graphics`、`Core Animation`、`Core Image` 等框架来绘制可视化内容，这些软件框架相互之间也有着依赖关系。这些框架都需要通过 OpenGL 来调用 GPU 进行绘制，最终将内容显示到屏幕之上。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ios-rendering-framework-relationship.png?x-oss-process=image/resize,w_800)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ios-rendering-framework-relationship.png?x-oss-process=image/resize,w_800)

## iOS 渲染框架

### `UIKit`

`UIKit` 是 iOS 开发者最常用的框架，可以通过设置 `UIKit` 组件的布局以及相关属性来绘制界面。

事实上， `UIKit` 自身并不具备在屏幕成像的能力，其主要负责对用户操作事件的响应（`UIView` 继承自 `UIResponder`），事件响应的传递大体是经过逐层的 **视图树** 遍历实现的。

### `Core Animation`

`Core Animation` 源自于 `Layer Kit`，动画只是 `Core Animation` 特性的冰山一角。

`Core Animation` 是一个复合引擎，其职责是 **尽可能快地组合屏幕上不同的可视内容，这些可视内容可被分解成独立的图层（即 CALayer），这些图层会被存储在一个叫做图层树的体系之中**。从本质上而言，`CALayer` 是用户所能在屏幕上看见的一切的基础。

### `Core Graphics`

`Core Graphics` 基于 Quartz 高级绘图引擎，主要用于运行时绘制图像。开发者可以使用此框架来处理基于路径的绘图，转换，颜色管理，离屏渲染，图案，渐变和阴影，图像数据管理，图像创建和图像遮罩以及 PDF 文档创建，显示和分析。

当开发者需要在 **运行时创建图像** 时，可以使用 `Core Graphics` 去绘制。与之相对的是 **运行前创建图像**，例如用 Photoshop 提前做好图片素材直接导入应用。相比之下，我们更需要 `Core Graphics` 去在运行时实时计算、绘制一系列图像帧来实现动画。

### `Core Image`

`Core Image` 与 `Core Graphics` 恰恰相反，`Core Graphics` 用于在 **运行时创建图像**，而 `Core Image` 是用来处理 **运行前创建的图像** 的。`Core Image` 框架拥有一系列现成的图像过滤器，能对已存在的图像进行高效的处理。

大部分情况下，`Core Image` 会在 GPU 中完成工作，但如果 GPU 忙，会使用 CPU 进行处理。

### `OpenGL ES`

`OpenGL ES`（OpenGL for Embedded Systems，简称 GLES），是 OpenGL 的子集。在前面的 图形渲染原理综述 一文中提到过 OpenGL 是一套第三方标准，函数的内部实现由对应的 GPU 厂商开发实现。

### `Metal`

`Metal` 类似于 `OpenGL ES`，也是一套第三方标准，具体实现由苹果实现。大多数开发者都没有直接使用过 `Metal`，但其实所有开发者都在间接地使用 `Metal`。`Core Animation`、`Core Image`、`SceneKit`、`SpriteKit` 等等渲染框架都是构建于 `Metal` 之上的。

当在真机上调试 OpenGL 程序时，控制台会打印出启用 `Metal` 的日志。根据这一点可以猜测，Apple 已经实现了一套机制将 OpenGL 命令无缝桥接到 `Metal` 上，由 `Metal` 担任真正于硬件交互的工作。

# UIView 与 CALayer 的关系

在前面的 `Core Animation` 简介中提到 `CALayer` 事实上是用户所能在屏幕上看见的一切的基础。为什么 `UIKit` 中的视图能够呈现可视化内容？就是因为 `UIKit` 中的每一个 UI 视图控件其实内部都有一个关联的 `CALayer`，即 `backing layer`。

由于这种一一对应的关系，视图层级拥有 **视图树** 的树形结构，对应 `CALayer` 层级也拥有 **图层树** 的树形结构。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-viewtree-layertree.png?x-oss-process=image/resize,w_500)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-viewtree-layertree.png?x-oss-process=image/resize,w_500)

其中，视图的职责是 **创建并管理** 图层，以确保当子视图在层级关系中 **添加或被移除** 时，**其关联的图层在图层树中也有相同的操作**，即保证视图树和图层树在结构上的一致性。

> 那么为什么 iOS 要基于 UIView 和 CALayer 提供两个平行的层级关系呢？

其原因在于要做 **职责分离**，这样也能避免很多重复代码。在 iOS 和 Mac OS X 两个平台上，事件和用户交互有很多地方的不同，基于多点触控的用户界面和基于鼠标键盘的交互有着本质的区别，这就是为什么 iOS 有 `UIKit` 和 `UIView`，对应 Mac OS X 有 `AppKit` 和 `NSView` 的原因。它们在功能上很相似，但是在实现上有着显著的区别。

> 实际上，这里并不是两个层级关系，而是四个。每一个都扮演着不同的角色。除了 **视图树** 和 **图层树**，还有 **呈现树** 和 **渲染树**。

## `CALayer`

那么为什么 `CALayer` 可以呈现可视化内容呢？因为 `CALayer` 基本等同于一个 **纹理**。纹理是 GPU 进行图像渲染的重要依据。

在 [图形渲染原理](http://chuquan.me/2018/08/26/graphics-rending-principle-gpu) 中提到纹理本质上就是一张图片，因此 `CALayer` 也包含一个 `contents` 属性指向一块缓存区，称为 `backing store`，可以存放位图（Bitmap）。iOS 中将该缓存区保存的图片称为 **寄宿图**。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-layer-contents.png?x-oss-process=image/resize,w_500)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-layer-contents.png?x-oss-process=image/resize,w_500)

图形渲染流水线支持从顶点开始进行绘制（在流水线中，顶点会被处理生成纹理），也支持直接使用纹理（图片）进行渲染。相应地，在实际开发中，绘制界面也有两种方式：一种是 **手动绘制**；另一种是 **使用图片**。

对此，iOS 中也有两种相应的实现方式：

- 使用图片：**contents image**
- 手动绘制：**custom drawing**

### Contents Image

Contents Image 是指通过 `CALayer` 的 `contents` 属性来配置图片。然而，`contents` 属性的类型为 `id`。在这种情况下，可以给 `contents` 属性赋予任何值，app 仍可以编译通过。但是在实践中，如果 `content` 的值不是 `CGImage` ，得到的图层将是空白的。

既然如此，为什么要将 `contents` 的属性类型定义为 `id` 而非 `CGImage`。这是因为在 Mac OS 系统中，该属性对 `CGImage` 和 `NSImage` 类型的值都起作用，而在 iOS 系统中，该属性只对 `CGImage` 起作用。

本质上，`contents` 属性指向的一块缓存区域，称为 `backing store`，可以存放 bitmap 数据。

### Custom Drawing

Custom Drawing 是指使用 `Core Graphics` 直接绘制寄宿图。实际开发中，一般通过继承 `UIView` 并实现 `-drawRect:` 方法来自定义绘制。

虽然 `-drawRect:` 是一个 `UIView` 方法，但事实上都是底层的 `CALayer` 完成了重绘工作并保存了产生的图片。下图所示为 `-drawRect:` 绘制定义寄宿图的基本原理。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-layer-bitmap-custom-drawing.png?x-oss-process=image/resize,w_500)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-layer-bitmap-custom-drawing.png?x-oss-process=image/resize,w_500)

- `UIView` 有一个关联图层，即 `CALayer`。

- `CALayer` 有一个可选的 `delegate` 属性，实现了 `CALayerDelegate` 协议。`UIView` 作为 `CALayer` 的代理实现了 `CALayerDelegae` 协议。

- 当需要重绘时，即调用 `-drawRect:`，`CALayer` 请求其代理给予一个寄宿图来显示。

- `CALayer` 首先会尝试调用 `-displayLayer:` 方法，此时代理可以直接设置 `contents` 属性。

  ```
  - (void)displayLayer:(CALayer *)layer;
  ```

- 如果代理没有实现 `-displayLayer:` 方法，`CALayer` 则会尝试调用 `-drawLayer:inContext:` 方法。在调用该方法前，`CALayer` 会创建一个空的寄宿图（尺寸由 `bounds` 和 `contentScale` 决定）和一个 `Core Graphics` 的绘制上下文，为绘制寄宿图做准备，作为 `ctx` 参数传入。

  ```
  - (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;
  ```

- 最后，由 `Core Graphics` 绘制生成的寄宿图会存入 `backing store`。

# Core Animation 流水线

通过前面的介绍，我们知道了 `CALayer` 的本质，那么它是如何调用 GPU 并显示可视化内容的呢？下面我们就需要介绍一下 Core Animation 流水线的工作原理。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-core-animation-pipeline-steps.png?x-oss-process=image/resize,w_800)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-core-animation-pipeline-steps.png?x-oss-process=image/resize,w_800)

事实上，app 本身并不负责渲染，渲染则是由一个独立的进程负责，即 `Render Server` 进程。

App 通过 IPC 将渲染任务及相关数据提交给 `Render Server`。`Render Server` 处理完数据后，再传递至 GPU。最后由 GPU 调用 iOS 的图像设备进行显示。

Core Animation 流水线的详细过程如下：

- 首先，由 app 处理事件（Handle Events），如：用户的点击操作，在此过程中 app 可能需要更新 **视图树**，相应地，**图层树** 也会被更新。
- 其次，app 通过 CPU 完成对显示内容的计算，如：视图的创建、布局计算、图片解码、文本绘制等。在完成对显示内容的计算之后，app 对图层进行打包，并在下一次 RunLoop 时将其发送至 `Render Server`，即完成了一次 `Commit Transaction` 操作。
- `Render Server` 主要执行 Open GL、Core Graphics 相关程序，并调用 GPU
- GPU 则在物理层上完成了对图像的渲染。
- 最终，GPU 通过 Frame Buffer、视频控制器等相关部件，将图像显示在屏幕上。

对上述步骤进行串联，它们执行所消耗的时间远远超过 16.67 ms，因此为了满足对屏幕的 60 FPS 刷新率的支持，需要将这些步骤进行分解，通过流水线的方式进行并行执行，如下图所示。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-core-animation-pipeline-workflow.png?x-oss-process=image/resize,w_800)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-core-animation-pipeline-workflow.png?x-oss-process=image/resize,w_800)

## Commit Transaction

在 Core Animation 流水线中，app 调用 `Render Server` 前的最后一步 Commit Transaction 其实可以细分为 4 个步骤：

- `Layout`
- `Display`
- `Prepare`
- `Commit`

### Layout

`Layout` 阶段主要进行视图构建，包括：`LayoutSubviews` 方法的重载，`addSubview:` 方法填充子视图等。

### Display

`Display` 阶段主要进行视图绘制，这里仅仅是设置最要成像的图元数据。重载视图的 `drawRect:` 方法可以自定义 `UIView` 的显示，其原理是在 `drawRect:` 方法内部绘制寄宿图，该过程使用 CPU 和内存。

### Prepare

`Prepare` 阶段属于附加步骤，一般处理图像的解码和转换等操作。

### Commit

`Commit` 阶段主要将图层进行打包，并将它们发送至 `Render Server`。该过程会递归执行，因为图层和视图都是以树形结构存在。

# 动画渲染原理

iOS 动画的渲染也是基于上述 Core Animation 流水线完成的。这里我们重点关注 app 与 `Render Server` 的执行流程。

日常开发中，如果不是特别复杂的动画，一般使用 `UIView` Animation 实现，iOS 将其处理过程分为如下三部阶段：

- Step 1：调用 `animationWithDuration:animations:` 方法
- Step 2：在 Animation Block 中进行 `Layout`，`Display`，`Prepare`，`Commit` 等步骤。
- Step 3：`Render Server` 根据 Animation 逐帧进行渲染。

[![img](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-animation-three-stage-process.png?x-oss-process=image/resize,w_800)](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-animation-three-stage-process.png?x-oss-process=image/resize,w_800)

