# 积木Sketch Plugin：设计同学的贴心搭档

## 背景

### 1.UI一致性项目

积木（Tangram）Sketch插件源于美团外卖UI的一致性项目，该项目自2019年5月份被提出，是UI设计团队与研发团队共建的项目，目的是改善用户端体验的一致性，提升多技术方案间组件的通用性和复用率，整体降低视觉改版的研发成本。

一直以来，外卖业务都处于高速发展阶段，人员规模在不断扩大，项目复杂度在持续增加。目前平台承载了美团餐饮、商超、闪购、跑腿、药品等多个业务品类，用户入口也覆盖了美团App外卖频道、外卖App、大众点评等多个独立应用。因为客户端一直比较侧重业务开发，为了满足业务快速上线的需求，UI组件并没有统一的实现，而是分散到各个业务场景中，在开发过程中因UI缺乏同一的标准而导致以下问题不断凸显：

**UI/UE层面**

① UI缺乏标准化的设计规范，在不同App及不同语言平台上设计风格不统一，用户体验不一致。

② 设计资源与代码均缺乏统一的管理手段，无法实现积累沉淀，无法适应新业务的开发需求。

**RD层面**

① 组件代码实现碎片化，存在多次开发的情况，质量难以得到保证。

② 各端代码API不统一，维护拓展成本较高，变更主题、适配Dark Mode等需求难以实现。

**QA层面**

重复走查，频繁回归，每次发版均需验证组件质量。

**PM层面**

版本迭代效率低，版本需求吞吐量低，不能满足业务的快速拓展能力。

基于上述开发工作中的切实痛点，以及未来可预见的对客户端能力的开发需求，我们迫切需要一套统一的UI设计规范，以此沉淀出设计风格，建立统一的UI设计标准，从而抽离成熟的业务场景，提供高质量、可扩展、可统一配置的同时能基于Android/iOS/MRN/Mach组件开发的代码库，且具备支持多业务高层次的代码复用能力，提高UI业务的中台能力，使项目具有高度一致性。

我们通过积木Sketch插件来落地设计规范，可以保证设计元素均从既定设计标准中获取，产出符合业务设计语言的设计稿，而各平台UI组件库中也有对应实现，从而使积木插件成为UI一致性的抓手，最终可以减少开发成本，提升交付质量，服务好我们美团的多个业务团队。

![外卖UI一致性项目](https://p1.meituan.net/travelcube/6a42e6ff5e69a73d205af7c7f5d913ba192513.png)

外卖UI一致性项目



### 2. Sketch & Sketch Plugin

要想保持UI一致性，就不能打破规则。从设计阶段颜色的选择、字体的规范、控件的样式到RD开发阶段代码的统一管理、API的制定、多端的实现方式，都必须遵守一套规则，而Sketch Plugin建设则是让规范落地执行的解决方案。

Sketch容易理解且上手简单；可与团队中的每个人创建、更新和共享所有Symbol组件，实现设计资源的共享和版本管理，从此告别“final-final-final-1”。目前，我们设计团队已经全面使用Sketch进行设计。

设计语言包括Iconfont、色板、文字规范、话术、插画、动画、组件等。其实它并不是一个抽象的概念，比如大家提到“美团”就会想起“美团黄”，想到可爱的“袋鼠”，想到那些骑着摩托车、穿着印有“美团外卖”亮黄色衣服的骑手小哥。通过设计语言，我们可以更好地传达品牌主张和设计理念。

**UI团队逐步将设计语言沉淀为设计规范，并将其量化内置于积木Sketch Plugin中，使产出的设计稿和RD代码库中的组件一一对应，从而形成一个完整的闭环，进而可加速整个业务的交付流程。**

![使用Sketch Plugin可以快速设计出标准页面](https://p0.meituan.net/travelcube/f3402244f32310f4f8761d7642d3446c80781.png)

使用Sketch Plugin可以快速设计出标准页面

**为何选用Sketch Plugin而不是Symbol组件来维持UI规范统一。 ？？？？？**

### 3. 积木Sketch 插件项目

其实，市面上已存在类似插件，为什么我们还要自己动手开发呢？因为UI设计语言与自身业务关联性很强，不同业务的色彩系统、图形、栅格系统、投影系统、图文关系千差万别，其中任意一环的缺失都会导致一致性被破坏。现有插件所提供的通用设计元素无法满足外卖设计团队的需求，开发一款可以与业务强关联且功能可定制的插件，显得尤为重要。

**积木Sketch插件经过一段时间的建设，目前已具备Iconfont、标准色板、组件库、数据填充、文字模板等功能。**

我们通过Iconfont可以从美团的图标库中拉取设计团队上传的SVG图标，并直接应用于设计稿；标准色板可以限定设计师的颜色使用范围，确保设计稿中的颜色均符合设计规范；组件库中包含从外卖业务中抽离的基本控件与通用组件，具有可复用和标准化的特点，并与不同语言平台组件库中的代码一一对应，使用组件库中的组件进行设计，可以提升UI的设计效率、开发效率以及走查效率；数据填充库可以实现图片填充和文本填充，图片包含了商品及商家素材，文字则包含了菜品、商铺名等信息，通过数据填充可以使设计师采用真实数据进行填充，让设计稿更为直观，也更贴近线上环境；文字模板中内置了Head、SubTitle、Body、Caption的使用规范，根据设计稿中文字的位置，点击文字图层即可直接应用字体、行高、字距等属性。

此外，我们还根据设计同学的使用反馈，不断增添新功能。同时也在拓展插件的使用场景，增加业务线切换功能，使积木插件可以为更多的团队服务，并期待它能成为更多设计师的“贴心搭档”。

![积木Sketch Plugin已支持功能](https://p1.meituan.net/travelcube/97c4d1c48c9754d0d939e0f7747b0fab186526.png)

积木Sketch Plugin已支持功能



### 4. 为什么要写这篇文章？

相信你读完上面的内容，肯定迫不及待的想了解一下Sketch插件，以此迅速提升自己团队开发效率了吧？

其实在开始之前，我们可先了解一些不利的条件。第一点，由于Sketch更新速度极快，但是官方文档却十分简单且陈旧，因此很多知名的Sketch Plugin因每次API的变更过大纷纷放弃维护；第二点，由于开发技术栈混乱，成熟项目一般还未开源，而开源的项目基本上没有什么参考价值，绝大多数都是“update 3 years ago”；最后一点，macOS开发资料更是少的可怜。

我们阅读了大量的文档却没有理清头绪，仿佛很多Wiki讲到关键地方，比如某个非常期待的功能是怎么实现的时候，作者竟然一笔带过，让人摸不到头脑。知乎上一篇Sketch Plugin的科普文，很多网友会评论“求教学视频，我可以花钱买的”。经过一步步踩坑，我们就总结了一些开发经验，为了避免大家“重复踩坑”，晚上可以早点下班陪陪家人，我们决定写一篇文章记录下开发的过程。虽然比起那些已经更新多版的成熟项目，但还有不少的差距，至少可以让大家不再那么迷茫。





## 准备放手Coding之前

好，先别着急敲击键盘。毕竟我们连使用哪种语言去开发都没决定，这曾经也是困恼我们许久的一个问题。目前Sketch Plugin开发大概有两种方式：

① 使用JavaScript + CocoaScript的混合开发模式，Sketch团队官方维护了一套JS API，并在开发者官网写了一句非常振奋人心的话：“ Take advantage of ES6, access macOS frameworks and use the Sketch APIs without learning Objective-C or Swift.”

理想很美满，但现实很骨感。这个API目前还不算完善，很多功能无法实现，因此我们需要搭配CocoaScript访问更丰富的内部API。

② 直接采用Objective-C 或Swift，并搭配macOS的UI框架AppKit进行开发，简单粗暴，并且可以利用OC运行时直接调用Sketch内部API。但这里要特别提醒一下，你要承担的风险是：随着Sketch的不断更新，内部API的命名和使用方式可能会发生较大变化，很多知名插件都因此放弃更新。

本文采用了“混合开发模式”进行讲解，希望能够给你一些小启发。

<img src="https://p0.meituan.net/travelcube/55ad5f0d1ac1357e3c7e392f17301f96165670.png" alt="Sketch 开发原理" style="zoom:50%;" />

Sketch 开发原理



### 1. Sketch Plugin开发流派

<img src="https://p0.meituan.net/travelcube/dae6ab440329cfbb218a1abf11b3dc4b291496.png" alt="img" style="zoom: 67%;" />

### 2. 环境配置

Skpm（Sketch Plugin Manager）是Sketch提供的用于Plugin创建、Build以及发布的官方工具。Skpm采用Webpack作为打包工具，当然如果你对前端知识足够熟悉，也可以采用Rollup或者roadhog。但是，为了防止遇到各种各样的报错，这里并不建议你这么做。

Skpm提供了一系列帮助快速入门的模板，最有用的莫过于skpm/with-webview，它可以帮助我们创建一个基于WebView展示的Demo示例，而且Skpm会在构建完成后，自动创建一个Symbolic Link将插件添加到Sketch的安装目录，使Plugin立即可用。

```JSON
//基于webpack的Sketch官方打包工具skpm
npm install -g skpm
//创建示例工程
skpm create my-plugin --template=skpm/with-webview
//Install the dependencies
npm install
//构建插件
npm run build
```

### 3. 项目结构

**Plugin Bundle** 按照上面的步骤操作完成后，我们会得到如下插件目录，它以标准化的分层结构存储了源码文件以及构建生成的Sketch插件安装包。这里没有使用官方文档中最简单的Demo，而是使用目前开发中最为常用的With-Webview模板进行分析，以免出现学完“1+1”后遇到的全是“微积分”问题，并且大部分插件均是在此基础上进行拓展。

目录中的参数，相信你在看完注释后马上就能明白。可是如果此前没有前端开发经验，可能不了解在经过Webpack打包后，脚本文件的文件名会发生变更，比如resources中的webview.js经过打包后会储存在插件的Resources文件夹中，而文件名则变更为resources_webview.js，因此在进行代码编写时，如果需要在html中引用此文件，也要使用打包后的文件名，即：。这里有个小技巧，如果你不知道脚本文件打包后的文件名及路径，建议先使用Webpack进行编译，然后查看其在打包后的Plugin中的位置和名称，然后再进行引用。

```JSON
├── assets //资源文件夹，如需更改需在package.json中的skpm.assets中设置 
├── my-plugin.sketchplugin   //skpm构建过程生成的插件包
│   └── Contents
│       ├── Resources
│       │   └── _webpack_resources
│       │   └── resources_webview.js
│       │   └── resources_webview.js.map
│       └── Sketch
│           ├── manifest.json
│           ├── __my-command.js
│           └── __my-command.js.map
├── package.json
├── webpack.skpm.config.js
├── resources //资源文件
│  ├── style.css
│  ├── webview.html
│  └── webview.js
└── src //需要被webpack打包的脚本文件以及manifest清单文件
    ├── manifest.json
    └── my-command.js
```

**Manifest**

你没有看错！plugin中也有manifest.json，它与其它平台比如Android开发中的清单文件意义相同。清单文件记录了作者信息、描述、图标以及获取更新的途径等等。想想看，每天熬夜加班写代码，总得有个地方把你的名字记录下来吧。但manifest最重要的作用其实是告诉Sketch如何运行插件，以及如何将插件集成进Sketch的菜单栏中。

commands使用一个数组，记录了插件所提供的所有命令。比如下面的例子，当用户从菜单栏点击 “显示工具栏”这个条目时，就会执行script.js中的function showPlugin() 。menu则提供了插件在Sketch菜单栏中的布局信息，Sketch会在插件被加载时初始化菜单。

```JSON
{
  "commands": [
   {
      "name": "显示工具栏",
      "identifier": "roo-sketch-plugin.toolbar",
      "script": "./script.js",
      "handlers": {
        "run": "showPlugin"
      }
    }
  ],
  "menu": {
    "title": "🦘外卖积木SketchPlugin工具栏",
    "items": ["roo-sketch-plugin.toolbar"]
  }
}
```

**package.json**

简单来说，只要你的项目中用到了NPM，根目录下就会自动生成package.json文件。Node.js项目遵循模块化的架构，package.json定义了这个项目所需要的各种模块以及配置信息。使用npm install命令会根据这个配置文件，自动下载所需的模块，也就是配置项目所需的运行和开发环境。

非常值得称赞的是，Plugin开发中对于网络请求、 I/O 操作以及其它功能，可以使用与Node.js兼容的polyfill，其中许多常用modules已经预装到了Sketch中，比如[console](https://github.com/skpm/console)、[fetch](https://github.com/skpm/sketch-polyfill-fetch)、[process](https://github.com/skpm/process)、[querystring](https://github.com/skpm/querystring)、[stream](https://github.com/skpm/stream)、[util](https://github.com/skpm/util)等。

这里你只需要知道以下几点：

- 需要参与Webpack打包的脚本文件必须在resources目录下声明，否则不会参与编译（重点！考试要考！）。
- assets目录需要配置在skpm.assets下。
- 常用的命令可以定义在scripts中方便直接调用。
- dependencies字段指定了项目运行所依赖的模块，devDependencies指定项目开发所需要的模块。

```JSON
{
  "name": "roo-sketch-plugin",
  "author": "hanyang",
  "description": "外卖积木Sketch plugin，UI同学好喜欢~",
  "version": "0.1.0",
  "skpm": {
    "manifest": "src/manifest.json",
    "main": "roo-sketch-plugin.sketchplugin",
    "assets": ["assets/**/*"]
  },
  "resources": [
    "src/webview/template/webview.js"
  ],
  "scripts": {
    "build": "rm -rf roo-sketch-plugin.sketchplugin && NODE_ENV=development skpm-build",
  },
  "dependencies": {},
  "devDependencies": {}
}
```

### 4. API Reference

**Javascript API**

由于使用了与Safari相同的JS引擎，Plugin脚本可以获得完整ES6支持。官方的JavaScript API由Sketch团队维护，并允许访问和修改Sketch文档，通过API可以向Sketch用户提供数据并提供一些基本的用户界面集成。

```JavaScript
//访问、修改和创建文档从color到layer再到symbol等方方面面
var sketchDom = require('sketch/dom')
//对于异步操作，JavaScript API提供了fibers延长contex的lifeTime
var async = require('sketch/async')
//直接在Sketch中提供图像或文本数据，DataSupplier直接与Sketch用户界面集成。
var DataSupplier = require('sketch/data-supplier')
//无需重新build的情况下显示通知以及获取用户输入
var UI = require('sketch/ui')
//保存图层或文档的自定义数据，并存储插件的用户设置。
var Settings = require('sketch/settings')
```

**CocoaScript Syntax**

CocoaScript通过赋予了JavaScript调用Sketch内部API以及macOS Cocoa frameworks的能力，这意味着除了标准的JavaScript库外，还可以使用许多很棒的类与函数。CocoaScript建立在苹果的JavaScriptCore之上，而JavaScriptCore是为Safari提供支持的JavaScript引擎。

因此，当你使用CocoaScript编写代码的时候，你就是在写JavaScript。CocoaScript中的Mocha实现JS到Objective-C的Bridge，虽然Mocha包含在CocoaScript中，但文档仍保留在原始Github中。因此，你在CocoaScript的Readme中看不到任何语法教程。这里一个诀窍是，如果你想了解Mocha将原生的Sketch Objects通过bridge，从Objective-C传递到JavaScript层的属性、类或者实例方法的信息，可以将其通过console打印出来：

```JavaScript
let mocha = context.document.class().mocha()
console.log(mocha.properties())
//OC
[executeOperation:withObject:error:]
//CocoaScript
executeOperation_withObject_error()
```

通过CocoaScript 提供的Bridge使用JavaScript调用Objective-C的基本语法如下:

- Objective-C的方括号语法“[ ]”转换为JavaScript中的点“ . ”语法。
- Objective-C的属性导出到JavaScript时Getter为object.name() 而Setter为object.name = ‘Sketch’。
- Objective-C的selectors被暴露为JavaScript 的代理方法。
- “：” 冒号被转换为下划线“ _”, 最后一个下划线是可选的。
- 调用带有一个下划线的方法需要加倍为两个下划线: sketch_method变为sketch__method。
- selector的每个component被连接成不带有分隔符的单个字符串。

### 5. Actions

**行为定义**

Action指的是由于用户交互而在应用程序中发生的事件，比如“打开文档”、“关闭文档”、“保存”等。Sketch所提供的了Action API可以使插件对应用程序中的事件做出反应，有点类似Android开发中的的BroadCast或者Job Scheduler。官方文档列举了数百个可供监听的Action，但最常用到的只有下面几个：

![img](https://p0.meituan.net/travelcube/7cb45073a552c089d6fba9b3e65da942215251.png)

**监听回调**

我们只需在插件的manifest.json文件中添加一个handler即可。比如下面的例子添加了对于“OpenDocument”的监听，也就是告诉插件在新文档被打开时要去执行onOpenDocument这个function。

```JSON
 {
      "script": "action.js",
      "identifier": "my-action-listener-identifier",
      "handlers": {
        "actions": {
          "OpenDocument": "onOpenDocument"
        }
      }
}
```

当一个Action被触发时，会回调JS中的监听方法，与此同时Sketch可以向目标函数发送Action Context，其中包含动作本身的一些信息。在下面例子中，每次打开文档时都会弹出一个Toast。

```JavaScript
function onOpenDocument(context) {
    context.actionContext.document.showMessage('Document Opened')
}
```

### 6. Bridge双向通信

在常规的插件开发中，UI层一般采用Webview实现，因此你可以使用各种前端开发框架，比如React或者Vue等；而插件的逻辑层（负责调用Skecth API）显然不在WebView中，因此需要通过Bridge进行通信。逻辑层将从服务器获取到的数据传递给UI层展示，而UI层则将用户的操作反馈传递给逻辑层，使其调用Sketch API更新Layers。

![Sketch 通信原理](https://p0.meituan.net/travelcube/164fa1b4c60b16d29693937e8518580a73211.png)

Sketch 通信原理



**插件发送消息到WebView**

```JavaScript
//On the plugin:
browserWindow.webContents
  .executeJavaScript('someGlobalFunctionDefinedInTheWebview("hello")')
  .then(res => {
    // do something with the result
  })

//On the WebView:
window.someGlobalFunctionDefinedInTheWebview = function(arg) {
  console.log(arg)
}
```

**WebView发送消息给插件**

```JavaScript
//On the webview:
window.postMessage('nativeLog', 'Called from the webview')
//On the plugin:
var sketch = require('sketch')
browserWindow.webContents.on('nativeLog', function(s) {
  sketch.UI.message(s)
})
```

经过了以上步骤，我们就得到了一个基础插件，它以WebView作为内容载体，并具有双向通信功能。打开插件时，Webview会将页面加载完成的事件传递给逻辑层，逻辑层调用Sketch API弹出Toast；点击Get a random number可以从逻辑层获取一个随机数。

![skpm/with-webview 运行效果](https://p1.meituan.net/travelcube/07124d0ecbebbee3c52d23f91fe0814962893.png)

skpm/with-webview 运行效果



## 快来正式加入开发队伍

相信阅读完上面的部分，制作一个简单的插件对于你来说，已经有点“游刃有余”了。但这个时候，疑惑也随之而来，为什么Demo和我们常用插件的UI差别如此之大？

没错，官方文档只教给我们最基础的插件开发流程，一个成熟的商业项目绝不仅仅是以上这些。一个功能完善的插件应该包括以下三部分：工具栏、WebView容器以及业务数据。下面，我们会一步步为你展示如何开发一个商业化插件UI，同时也会演示美团外卖“填充功能”的实现（注：篇幅原因文档中仅保留关键代码。）

![常规Sketch插件结构](https://p0.meituan.net/travelcube/7a7832cb84f0e2d688770428c8ae2760144863.png)

常规Sketch插件结构



### 1. 创建吸附工具栏

所谓吸附式工具栏，就是展示在Skecth右侧Inspector Panel旁边的工具栏，它以吸附的方式与Sketch操作界面融为一体，这也是绝大多数插件的视觉呈现方式。工具栏中展示了当前插件可以提供的大部分功能，方便我们在操作Document时快速选取使用。

开发工具栏主要使用NSStackView、NSButton、NSImage以及NSFont这几个类，如果没有开发过macOS应用的同学可能对这些类有些陌生，可以类比iOS开发中以UI作为前缀的控件类，NS前缀主要是AppKit以及Foundation的相关类，MS前缀则是Skecth的相关类，CA、CF前缀为核心动画库和核心基础类。

下面的代码记录了创建工具栏的关键步骤，更为详细的操作可以参考一些Github仓库，比如sketch-plugin-boilerplate等。

```JavaScript
const contentView = context.document.documentWindow().contentView();
const stageView = contentView.subviews().objectAtIndex(0);

//1.创建toolbar
const toolbar = NSStackView.alloc().initWithFrame(NSMakeRect(0, 0, 27, 420));
toolbar.setBackgroundColor(NSColor.windowBackgroundColor());
toolbar.orientation = 1;

//2.创建Button
const button =  NSButton.alloc().initWithFrame(rect)
const Image = NSImage.alloc().initWithContentsOfURL(imageURL)
button.setImage(image)
button.setTitle("数据填充")
button.setFont(NSFont.fontWithName_size('Arial',11))

//3.将Button加入toolbar
toolbar.addView_inGravity(button, gravityType);

//4.将toolbar加入SketchWindow
const views = stageView.subviews()
const finalViews = []
for (let i = 0; i < views.count(); i++) {
 finalViews.push(view)
 if(view[i].identifier() === 'view_canvas'){
   finalViews.push(toolbar)
}
 stageView.subviews = finalViews
 stageView.adjustSubviews()
```

### 2. 创建WebView容器

除了通过CocoaScript创建原生NSPanel外，这里推荐使用官方的sketch-module-web-view快速创建WebView容器，它提供了丰富的API对窗口的展示样式和行为进行定制，包括Frameless Window、Drag等，同时还封装了WebView与插件层的通信的Bridge，使你可以轻松在”frontend” （the WebView）和”backend” （the plugin running in Sketch）之间发送消息。

```JavaScript
//(1)方法一：原生方式加入webview
const panel = NSPanel.alloc().init();
panel.setFrame_display(NSMakeRect(0, 0, panelWidth, panelHeight), true);
const wkwebviewConfig = WKWebViewConfiguration.alloc().init()
const webView = WKWebView.alloc().initWithFrame_configuration(
  CGRectMake(0, 0, panelWidth, panelWidth),
  wkwebviewConfig
)
panel.contentView().addSubview(webView);
webview.loadFileURL_allowingReadAccessToURL(
  NSURL.URLWithString(url),
  NSURL.URLWithString('file:///')
)
//(2)方法二：使用官方的BrowserWindow
import BrowserWindow from "sketch-module-web-view";
const browserWindow = new BrowserWindow(options);
const webViewContents = browserWindow.webContents;

 webViewContents
    .executeJavaScript(`someGlobalFunctionDefinedInTheWebview(${JSON.stringify(someObject)})`)
    .then(res => {
      // do something with the result
    })
 browserWindow.loadURL(require('./webview.html'))
```

### 3. 创建内容页面

历尽千辛万苦，我们终于拿到了WebView，这下就可以发挥你“天马行空”的想象力了。不管是React还是Vue，亦或只是一些简单的静态页面对于你而言应该都不在话下。在完成界面开发后，只需通过Window向插件发送指令即可。下面的例子演示了积木插件的“数据填充”功能。

**UI侧**

```JavaScript
import React from 'react';
import ReactDOM from 'react-dom';

//使用react搭建用户页面
ReactDOM.render(<Provider store={store}><App /></Provider>, document.getElementById('root'));

//传递用户点击填充类目给插件层，这里以填充文字为例
export const PostMessage = (name, fillData) => {
  try {
    window.postMessage("fill-text-layer", fillData);
  } catch (e) {
    console.error(name, "出现异常！！！" + fillData);
  }
}; 
```

**插件侧**

```JavaScript
  browserWindow.webContents.on('fill-text-layer', function(s) {
   //找到当前页面document
  const document = context.document;
   //获取用户选择的layers
     const selection = Document.fromNative(document).selectedLayers;
        layers.forEach(item => {
          //判断layer类型是否为文字
          if (item.type === 'Text') {
            //更新textlayer
            item.text = value;
          }
   }); 
})
```

### 4. 还想加点出彩的功能

如果你还不满足于此，说明你真的是个很爱学习，也很有潜力的开发同学。一个完善的插件需要包括交互层、API层、业务层、调试层以及发布层，每层各司其职，它们都在默默干好自己的工作。

前面的步骤，通过构件菜单栏、创建Webiew完成了交互层的开发；通过Webview的Bridge传递用户操作到插件侧代码，之后调用Sketch API对图层进行操作，这是API层的工作；而根据自身需求并依托交互层与API层的实现去编写业务代码，则是业务层的工作；至此，你应该就拥有了一个可运行的插件了。

但除此之外，在代码编写过程中还需要Lint组件辅助开发，发现问题需要使用各类Dev工具进行调试，通过QA验证后，需要Cli工具打包并发布插件更新。这一小节，我们将简单介绍一些基本的调试层和发布层知识。

![积木Sketch Plugin结构](https://p1.meituan.net/travelcube/55a179aab8410e97f353fbced0d2605e138503.png)

积木Sketch Plugin结构



**Webpack配置**

Skpm默认采用Webpack作为打包工具。Webpack是一个现代JavaScript应用程序的静态模块打包器（Module Bundler）。当Webpack处理应用程序时，它会递归地构建一个依赖关系图（Dependency Graph），其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个Bundle，需要在webpack.config.js进行配置，类似于Android中的Gradle，同样支持各种插件。

![Webpack处理流程示意](https://p1.meituan.net/travelcube/a75f6fd413ff9ca7e49c1f4485c37bcc454032.png)

Webpack处理流程示意



由于插件的开发者未必是前端同学，可能之前并没有接触过Webpack，因此我们在这里介绍它的一些常用配置，让你有更多的时间关注业务代码。第一次接触Webpack是在去年一次公司内部的技术培训上（美团技术学院提供了很多技术培训课程，加入我们就可以尽情地在知识的海洋中遨游了），美团MRN项目的打包方案就是Webpack。

在前端圈有各种各样的打包工具，比如Webpack、Rollup、Gulp、Grunt 等等。RN打包用的是Facebok实现的一套叫做Metro的工具，而美团MRN打包工具的选型是Webpack，因为Webpack具有强大的插件机制和丰富的社区生态，可以完成复杂的流水线打包工作，Webpack在Plugin开发中同样发挥了非常重要的作用。Webpack有五个核心概念：

![img](https://p1.meituan.net/travelcube/f5dbb3735e40b0e42cee3f1afa6af9ee342268.png)

在插件开发中需要处理html、css、sass、jpg、style等各种文件，只有在Webpack中配置相应的Loader后，这些文件才能被处理。而且我们很可能遇到某些文件需要使用特定的插件，而其它文件又无需处理的情况。下面的示例中列举了添加插件、对文件单独处理以及参数配置这三个常用的基本操作。

```JavaScript
module.exports = function (config, entry) { 
  //常用功能1：增加插件
    config.module.rules.push({
    test: /\.(svg)([\?]?.*)$/,
    use: [
      {
        loader: "file-loader",
        options: {
          outputPath: url => path.join(WEBPACK_DIRECTORY, url),
          publicPath: url => {return url;}}}
    ]
  });}
  
//常用功能2：对文件单独处理
if (entry.script === "src/script.js") {
    config.plugins.push(
      new htmlWebpackPlugin({ })
    );
}

//常用功能3：定制js处理
  config.module.rules.push({
    test: /\.jsx?$/,
    use: [
      { loader: "babel-loader",
        options: {
          presets: [
            "@babel/preset-react",
            "@babel/preset-env"
          ],
          plugins: [
            //引入antd组件库
            ["import",{libraryName: "antd",libraryDirectory: "es",style: "css"}]
      ]}}]
  });
```

**ESLint配置**

JavaScript是一门非常灵活的语言，很多错误往往运行时才爆出，通过配置前端代码检查方案，在编写代码过程中可直接得到错误反馈，也可以进行代码风格检查，不仅提升了开发效率，同时对不良代码编写习惯也能起到纠正作用。在ESLint中需要配置基础语法规则、React 规则、JSX规则等，由于Sketch插件的CocoaScript语法较为特殊，需要配置全局变量以此忽略AppKit中无法识别的类。

虽然，我们曾在部门组会中被多次“安利”ESLint的强大作用（这里给大家推荐一篇技术文章：[ESLint 在中大型团队的应用实践](https://tech.meituan.com/2019/08/01/eslint-application-practice-in-medium-and-large-teams.html)），但如果不是做前端或者RN开发的同学，可能对于ESLint的复杂配置并不熟悉。可以直接使用Skpm提供的ESlint Config，里面配置了包含Sketch和macOS的头文件的全局变量，而代码格式化则推荐使用Prettier。

```
npm install --save-dev eslint-config-sketch
//或者直接使用带prettier以eslint的skpm template工程
$ skpm create my-plugin --template=skpm/with-prettier
```

**内容服务端化**

Sketch推出的库（Library）功能对于维护设计系统或风格指南，起到非常重要的作用，可以给团队带来高效工作体验，甚至改变设计团队工作方式和流程。我们通过组件库可以在整个设计团队中共享组件（Symbol），Library可以实现“一处更改，处处生效”，即使是关联了远程组件库历史的设计稿检测到更新时，也会收到Sketch通知，确保工作中使用的是最新组件。

库功能对美团外卖UI一致性起着至关重要的作用，这主要体现在两方面：首先是实现设计风格沉淀，目前袋鼠UI已经形成了自己的独特风格，外卖设计团队根据设计规范，对符合UI一致性外卖业务场景的组件不断进行抽象及建设，沉淀出越来越多的通用业务组件，这些组件需要及时扩充到Library中，供团队成员使用；另外一个作用，则是保持团队使用的均为最新组件，由于各种原因，组件的设计元素（色彩、字体、圆角等属性）可能会发生变更，需要及时提醒团队成员更新组件，保持所有页面的一致性。

![Sketch内置的iOS远程组件库](https://p0.meituan.net/travelcube/612c82f558312fc591ef62ffb898816c22367.png)

Sketch内置的iOS远程组件库



![Library中的Symbol提示更新](https://p0.meituan.net/travelcube/f1909018b5b9a22f797ddabc2701d6dd627910.png)

Library中的Symbol提示更新



库组件自动更新，其实就是 “库列表” - “库 ID” - “外部组件原始 ID” 这三者的关联。Sketch内部是靠UUID进行对象识别的，通过库组件的库ID，从库面板的列表中，按照添加的时间从新到旧依次检索所有未被禁用的、链接完好的库，直到匹配到库的ID ，然后查找该库文件内是否有与库组件SymbolID匹配的组件，如果包含且内容有差异就提醒更新，更新的过程实际上是内容替换。

我们通过以下步骤使用RSS技术共享Library供整个UI设计团队使用：

- 将Library Document 托管到公司内网服务器上。
- 创建一个XML文件记录版本信息和更新地址。
- 最后使用Meyerweb URL编码器之类的工具（或直接encodeURIComponent）对XML feed URL进行编码并将其添加到以下内容：sketch://add-library?url=https://***.xml。
- 将此URI在浏览器中打开即可。

```JavaScript
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom" xmlns:content="http://purl.org/rss/1.0/modules/content/" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:sparkle="http://www.andymatuschak.org/xml-namespaces/sparkle">
  <channel>
    <title>My Sketch Library</title>
    <description>My Sketch Library</description>
    <image>
      <url></url>
    </image>
    <item>
      <title>My Sketch Library</title>
      <pubDate>Wed, 23 Jun 2019 11:19:04 +0000</pubDate>
      <enclosure url="mysketchlibrary.sketch" type="application/octet-stream" sparkle:version="1"/>
    </item>
  </channel>
</rss>
```

### 5. 开发流程小结

前面一口气讲述了很多内容，可能你一时无法消化，这里对插件的开发流程作个简要的总结：

- 首先利用JavaScript 或CocoaScript开发操作面板。
- 使用NPM安装所需依赖。
- 通过Bridge传递用户操作到插件逻辑侧，通过调用Skecth API对文档进行处理。
- 使用Webpack进行打包。
- 通过测试后发布插件更新。

![Sketch Plugin开发流程](https://p0.meituan.net/travelcube/4ce3fb76fc584045166289250f2c122b94743.jpg)

Sketch Plugin开发流程



## 别人可能没告诉你的事儿

这部分主要记录了积木Sketch Plugin开发过程中的踩坑经历，但是这里，我们没有贴大段的代码，没有直接告诉你答案，而是把分析问题的过程记录下来。“授人以鱼不如授人以渔”，相信只要你了解了这些分析技巧，即使之后遇到更多的问题，也可以轻（jia）松（ban）解决。

### 1. 与Xcode工程混合编译

首先，我们要明确一个问题，为什么要使用XCode工程？

虽然官方提供了JS API并承诺持续维护，但这项工作一直处于Doing状态，而且官方文档更新缓慢，没有明确的时间节点。因此，对于某些功能，比如我们想建一个具有Native Inspector Panel的插件，就不得不使用XCode进行开发。使用Xcode开发对于iOS开发者也更加友好，无需再学习前端界面开发知识。

这里推荐Invison的开发成员James Tang分享的博客文章《[Sketch Plugin Xcode Template](https://blog.magicsketch.io/sketch-plugin-xcode-template-c8236a6f7fff)》，里面详细描述了构建插件XCode工程的步骤，这也成为很多插件开发者遵循的范本。当然随着Sketch的不断升级，某些API已经不受支持，但作者讲述的开发流程和思路依然没有改变，具有很高的学习价值。

```
JavaScript
//利用 Mocha加载framework
var mocha = Mocha.sharedRuntime();
[mocha loadFrameworkWithName:frameworkName inDirectory:pluginRootPath]
```

除此之外，Skpm中已经内置了@skpm/xcodeproj-loader，也可在JS中直接加载Framework。

```
JavaScript
//加载framework
const framework = require('../xcode-project-name/project-name.xcodeproj/project.pbxproj');
const nativeClass = framework.getClass('NativeClassName');
//获取nib文件
const ui = framework.getNib('NativeNibFile');
//也可以直接加载xib文件
const NibUI = require('../xcode-project-name/view-name.xib')
var nib = NibUI()
let dialog = NSAlert.alloc().init()
dialog.setAccessoryView(nib.getRoot())
dialog.runModal()
```

当然你也可以直接使用Github上一些知名的开源项目，有些会直接提供Framework供你使用，比如更改原生的toolbar：

![img](https://p0.meituan.net/travelcube/e4cb24c178d193fdc1d46a43b4971a9767616.png)

### 2. 了解Electron

为什么在讲述Sketch Plugin的时候，忽然会提到Electron？这里有一个小故事，某天上班打开大象（美团内部沟通软件）。

![MacOS版大象截图](https://p1.meituan.net/travelcube/7d674973cdbb9465bb6dabe8e58cd7da1380259.png)

MacOS版大象截图



看到一条公众号推送，是公司成立了Electron技术俱乐部（美团技术团队内部自发成立了很多技术俱乐部），经过了解发现Electron基于Chromium和Node.js，可以使用HTML、CSS和JavaScript构建桌面应用程序，Electron负责其中比较复杂的部分，而开发者只需关心应用的核心需求即可。大象的Mac端就大量使用了Electron技术，用Web框架去开发桌面应用，可以直接复用Web现有的开发成果并获得出色的运行效率。

![img](https://p0.meituan.net/travelcube/94a4001f3e8f4b7ad95acb40135a7f7b187370.png)

我们就进行了简单的学习，在之后的一段时间并没有再去关注这项技术，直到某天在插件开发的过程中忽然遇到一个问题：在插件WebView显示的情况下，在桌面空白处点击使Sketch软件失去焦点，整个App就会被隐藏。试了几个流行的插件，发现大部分均有此问题，这给设计师的工作造成了诸多不便。试想，我只是去打开Finder找一个文件，你为什么要把我的软件最小化？在Github上留言后，很快得到了项目开发者Mathieu Dutour的官方回复，原来只需要设置一个hidesOnDeactivate属性即可。

等等！这不是Electron中的属性么？仔细查看Readme才发现作者写道“The API is mimicking the [BrowserWindow](https://www.electronjs.org/docs/api/browser-window) API of Electron.”这下可方便多了！你想自定义窗口的表现，只需按照Electron的API设置即可，想想看其实Electron的工作方式是不是和Sketch Plugin如出一辙？

![img](https://p0.meituan.net/travelcube/03b7febff302d8484466124694dbd17572114.png)

### 3. 更新原生属性面板

为了更好地提升积木Sketch Plugin的使用体验，UI同学通过建立公共Wiki记录我们设计团队在插件使用过程中的反馈建议，其中有一条很奇怪：“通过插件面板更新Layer属性后，右侧面板不刷新。”和上一个问题一样，经测试其它插件大部分也有此问题，但是如何去更新右侧属性面板呢？翻阅了Sketch的API文档还是“丈二和尚，摸不着头脑”。这个时候想起了macOS开发的一个神器Interface Inspector，它可以在运行时分析正在运行的Mac应用程序的界面结构和属性，非常强大。

开心的下载下来后，发现这个软件上次的更新时间是6年前，忽然有了一种不祥的预感。果然Attach任何App时都会提示无法Attach，在macOS Catalina版本已经无法运行。可是这怎么能难倒“万能”的程序员呢？我们查看系统报错，发现是mach_inject_bundle_stub错误，查阅发现mach_inject_bundle_stub是Github上的一个开源库，所以自己下载源码重新编译个Bundle包就可以了。

Attach成功后，就可以对Sketch的面板进行属性分析了，是不是忽然感觉打开了新世界的大门？经过查阅发现右侧面板在MSInspectorController中。如下图所示：

![Interface Inspector对Sketch进行运行时分析](https://p0.meituan.net/travelcube/a4dd04ff6a57ff721abc907ce8222d75553307.png)

Interface Inspector对Sketch进行运行时分析



下一步需要用Class-Dump工具来提取Sketch的头文件，查看可以对inspector面板进行操作的所有方法：

![通过class-dump得到的头文件](https://p0.meituan.net/travelcube/30ff92bfaa46efa23b84f8fc7f2c800e47712.png)

通过class-dump得到的头文件



不出所料，我们发现了reload()，猜测调用这个方法可以刷新面板，测试一下发现问题被修复了。如果你使用Sketch的JavaScript API的话，名称不一定能完全对应，但是基本差不多，稍加分析也可以找到。这里只是教大家一个思路，这样即使遇到其它问题，按照上面的步骤试试看，没准就可以解决。

```
JavaScript
// reload the inspector to see the changes 
var sketch = require('sketch')
var document = sketch.getSelectedDocument()
document.sketchObject.inspectorController().reload()
```

## 未来等你加入

如你所见，积木Sketch Plugin可以帮助设计团队提升设计效率、沉淀设计语言以及减少走查负担；让RD同学面对新项目时，可以专注于业务需求而无需把时间耗费在组件的编写上；减少QA工作量，保证控件质量无需频繁回归测试；帮助PM提高版本迭代效率及版本需求吞吐量，提供业务的快速拓展能力。

目前，积木插件开发还处于较为初级的阶段，包括Mach（外卖自研动态化框架）实时预览、模板代码自动生成、自建插画库等功能已经在路上。除此之外，我们还规划了很多激动人心的功能，需要制作更多精美的前端页面，需要更完善的后台管理。

