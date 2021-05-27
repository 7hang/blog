# 积木Sketch插件进阶开发指南

## 背景

### 1. 积木工具链体系

前段时间我们在美团技术团队公众号上发表了《[积木Sketch Plugin：设计同学的贴心搭档](https://tech.meituan.com/2020/05/21/waimai-sketch-plugin.html)》一文，不曾想到，这个仅在美团外卖C端使用的插件受到了更多的关注，美团多个业务团队纷纷向我们抛出“橄榄枝”，表示想要接入以及并表达了愿意共同开发的意向，其它互联网同行也纷纷询问相关的技术，一时让我们有些“受宠若惊”。回想起写第一篇文章的时候，我们的内心还是有些不安的。作为UI同学的一个设计工具，有些RD甚至没有听说过Sketch这个名字，我们很认真地修改过上一篇文章的每一句措辞，争取让内容更丰富有趣，当时还很担心不会被读者接受。

积木Sketch插件的“意外走红”，确实有些出乎我们的意料，但正是如此，才让我们知道UI一致性是绝大部分开发团队面临的共性问题，大家对落地设计规范，提高UI中台能力，提升产研效率都有着强烈的诉求。为了帮助更多团队提升产研效率，我们成立了袋鼠UI共建项目组，将门户建设、工具链建设以及组件建设统一管理统一规划，并将工具链的品牌确定为“积木”，而积木Sketch插件便是其中重要的一环。

我们通过建立包含相同设计元素的统一物料市场，PM通过Axure插件拾取物料市场中的组件产出原型稿；UI/UE通过Sketch插件落地物料市场中的设计规范，产出符合要求的设计稿；而物料市场中的组件又与RD代码仓库中的组件一一对应，从而形成了一个闭环。未来，我们希望通过高保真原型输出，可以给中后台项目、非依赖体验项目提供更好的服务体验，赋予产品同学直接向技术侧输出原型稿的能力。

![袋鼠UI工具链体系](https://p1.meituan.net/travelcube/887e98d01ab68110d3bcaa623e121d2a232962.png@1920w_941h_80q)

袋鼠UI工具链体系



### 2. 积木插件平台化

伴随着“积木”品牌的确立，越来越多的团队希望可以接入积木Sketch插件，其中部分团队也在和我们探讨技术合作的可能性。UI设计语言与自身业务关联性很强，不同业务的色彩系统、图形、栅格系统、投影系统、图文关系千差万别，其中任意一环的缺失都会导致一致性被破坏，业务方都希望通过积木插件实现设计规范的落地。为了帮助更多团队的UI同学提升设计效率，节约RD同学页面调整的时间，同时也让App界面具有一致性，从而更好地传达品牌主张和设计理念，我们决定对积木插件进行平台化改造。平台化是指积木插件可以接入各个业务团队的整套设计规范，通过平台化改造，可以使积木插件提供的设计元素与业务强关联，满足不同业务团队的设计需求。

![积木Skecth Plugin平台化示意](https://p1.meituan.net/travelcube/2145ccc358589a566876621f87e2250b101824.png@1920w_713h_80q)

积木Skecth Plugin平台化示意



积木插件原本只是外卖提升UI/RD协作效率的一次尝试，最初的目标仅是UI一致性，但是现在已经作为全面提升产研效率的媒介，承载了越来越多的功能。围绕设计日常工作，提供高效的设计元素获取方式，让工作变得更轻松，是积木的核心使命。如何推动设计规范落地，并且输出到各个业务系统灵活使用，是我们持续追寻的答案。而探寻研发和设计更为高效的协作模式也是我们一直努力的方向。

通过一段时间的平台化建设，目前美团已经有7个设计团队接入了积木插件，覆盖了美团到家事业部大部分设计同学，未来我们会持续推进积木插件的平台化建设，不断完善功能，期望能将积木插件打造成业界一流的品牌。

### 3. Sketch插件开发进阶

第一篇文章可能是为数不多的入门教程，而本篇可能是你能找到的唯一一篇进阶开发文章。进阶开发主要涉及如何切换业务方数据，即选择所属业务方后，对应的组件、颜色等设计素材切换为当前业务方在物料市场中上传的元素；将承载组件库的Library文件转化为插件可以识别的格式，并在插件上展示，以供设计师在绘制设计稿时选择使用；一些优化运行效率，提升用户体验的方法。

Sketch插件代码由于和业务强相关，且实现方式较为复杂，可能存在部分敏感信息，所以基本没有成熟的插件开源。在进行一些复杂功能开发时，我们也常常“丈二和尚摸不到头脑”，“要不这个功能算了吧”的想法也不止一次出现，可是每当开会看到旁边的设计同学在使用“积木”插件认真作图时，又一次次坚定了我们的信念，要不加班再试试吧，没准就能实现了呢？一次“委曲求全”，后面可能导致整个项目慢慢崩塌，所以我们一直以将积木插件打造成为业界领先的插件为信念。如果说看过了第一篇文章你已经知道了如何开发一款插件，那么通过本篇文章的学习你就可以真正实现一款可以与业务强关联且功能可定制的成熟工具，与其说是介绍如何开发一个进阶版的Sketch插件，不如说是分享给大家完成一个商业化项目的经验。

![img](https://p0.meituan.net/travelcube/ca4c4cbcfa27d8997e81690777cec9f11054954.gif@1200w_874h_80q)

## 支持多业务切换

为了当面对“我们可以接入积木插件吗”这种灵魂拷问时不再手足无措，平台化进程迅速启动。平台化的核心其实就是当发生业务线变更时，物料市场中的素材整体同步切换，因此我们需要进行如下几个操作：首先建立全局变量，存储当前用户所述业务方信息及鉴权信息；用户选择功能模块后，根据用户所述业务方，拉取对应素材；处理Library等素材并渲染页面展示；根据素材信息变更画板中的相关Layer。这部分主要介绍如何依靠持久化存储来实现业务切换的功能，就像在第一篇启蒙文档中说的那样，这里不会贴大段的代码，只会帮你梳理最核心的流程，相信你亲自实践一次之后，以后的困难都可以轻松解决。

![img](https://p1.meituan.net/travelcube/d89ca297723c9cdabe3fdf8de30bb281298154.png@1920w_972h_80q)

### 1. 定义通用变量

功能模块展示的素材与当前选择的业务相关，因此需要在每个功能模块的Redux初始化状态中增加一些全局状态变量。比如所有模块都需要使用businessType来确定当前选择的业务，使用theme进行主题切换，使用commonRequestParams获取用户鉴权信息等。

```JavaScript
export const ACTION_TYPE = 'roo/colorData/index';
const intialState = {
  id: 'color',
  title: '颜色库',
  theme: 'black',
  businessType: 'waimai-c',
  commonRequestParams: {
    userInfo: '',
  },
};
export default reducerCreator(intialState, ACTION_TYPE);
```

### 2. 实现数据交换

第一步：WebView侧获取用户选择，将所选的业务方数据通过window.postMessage方法传递至插件侧。

```JavaScript
window.postMessage('onBusinessSelected', item);
```

第二步：Plugin侧通过webViewContents.on( )方法接收从WebView侧传递过来的数据。Sketch官方通过Settings API提供了一些类的方法来处理用户的参数设置，这些设置在Sketch关闭后依然会保存，除了存储一段JSON数据外，Layer、Document甚至是Session variable都是支持的。

```JavaScript
 webViewContents.on('onBusinessSelected', item => {
    Settings.setSettingForKey('onBusinessSelected', JSON.stringify(item));
  });

// 除此之外，插件侧也可以通过localStorage向WebView注入数据
browserWindow.webContents.executeJavaScript(
      `localStorage.setItem("${key}",'${data}')`
);
```

第三步：当用户通过工具栏选择某一功能模块（例如“插画库”）时，会回调NSButton的点击事件监听，此时除了需要要让WebView展示（Show）以及获取焦点（Focus）外，还需要将第二步存储的业务方信息传入，并以此加载当前业务方的物料数据。

```JavaScript
//用户打开的功能模块
const button = NSButton.alloc().initWithFrame(rect) 
  button.setCOSJSTargetFunction(() => {
    initWebviewData(browserWindow);
    browserWindow.focus();
    browserWindow.show();
  });

// 注入全局初始化信息
function initWebviewData(browserWindow) {
  const businessItem = Settings.settingForKey('onBusinessSelected');
  browserWindow.webContents.executeJavaScript(`initBusinessData(${businessItem})`);
}
```

WebView侧功能模块收到初始化信息，开始进行页面渲染前的数据准备。Object.keys()方法会返回一个由给定对象的自身可枚举属性组成的数组，遍历这个数组即可拿到所有被注入的初始化数据，之后通过redux的store.dispatch方法更新state即可。至此实现业务切换功能的流程就全部结束了，是不是觉得非常简单，忍不住想亲自动手试一下呢？

```JavaScript
ReactDOM.render(<Provider store={store}><App /></Provider>,
  document.getElementById('root')
);

window.initBusinessData = data => {
  const businessItem = {};
  Object.keys(data).forEach(key => {
    businessItem[key] = { $set: initialData[key] };
  });
  store.dispatch(update(businessItem));
};

const update = payload => ({
  type: ACTION_TYPE,
  payload,
});
```

### 3. 小结

有小伙伴会问，为什么WebView与Plugin侧需要数据传递呢，它们不都属于插件的一部分么？根本原因是我们的界面是通过WebView展示的，但是对Layer的各种操作是通过Sketch的API实现的，WebView只是一个网页，本身与Sketch并无关系，因此必须使用bridge在两者之间进行数据传递。别担心，这里再带你把整个流程梳理一遍：①在插件启动后会从服务端拉取业务方列表；②用户在WebView中选择自己所属的业务方；③将业务方数据通过bridge传递至Plugin侧，并通过Sketch的Settings API进行持久化存储，这样就可以保证每次启动Sketch的时候无需再次选择所属业务方；④用户点击插件工具栏的按钮选择所需功能（例如色板库、组件库等），从持久化数据中读取当前所属业务方，并通知WebView侧拉取当前业务方数据。至此，整个流程结束。

![img](https://p0.meituan.net/travelcube/1669d16918e62863f636c9f5b1eb935a313007.png@1920w_1438h_80q)

## Library库文件自动化处理

这部分将介绍如何将Library库文件转化为插件可以识别的JSON格式，并在插件上展示。

如果要问Sketch插件最重要的功能是什么，组件库绝对是无可争议的C位。在长期的版本迭代中，随着功能的不断增加以及UI的持续改版，新旧样式混杂，维护极为困难。设计师通过将页面走查结果归纳梳理，制定设计规范，从而选取复用性高的组件进行组件库搭建。通过搭建组件库可以进行规范控制，避免控件的随意组合，减少页面差异；组件库中组件满足业务特色，同时具有云端动态调整能力，可以在规范更新时进行统一调整。

目前，我们将组件集成进Sketch供UI使用大致分为两个流派：一个是基于Sketch官方的Library库文件，设计师通过将业务中复用性高的Symbol组件归纳整理生成库文件（后缀.sketch），并上传至云端，插件拉取库文件转化为JSON并在操作面板展示供选取使用；另一个则是采用类似Airbnb开源的[React-Sketchapp](https://github.com/airbnb/react-sketchapp)这样的框架，它可以让你使用React代码来制作和管理视觉稿及相关设计资源，官方把它称作“用代码来绘画”，这种方案的实施难度较大，因为本质上设计是感性和理性的结合，设计师使用Sketch是画，而非带有逻辑和层级关系的写，他们对于页面的树形结构很难理解，上手成本较高，而且代码维护成本相对较大。我们不去评价哪种方案的好坏，只是第一种方案可以更好地满足我们的核心诉求。

![Sketch组件库处理效果示意](https://p0.meituan.net/travelcube/29873e6019b76c85e6118a3a8a837113326987.png@1920w_914h_80q)

Sketch组件库处理效果示意



### 1. 订阅远程组件库

Library库文件实际上是一个包含components的文档，components包括了Symbols、Text Styles以及Layer Styles三类，将Library存储在云端就可以在不同文档甚至不同团队间共享这些components。由于组件库实时指向最新，因此当其维护者更新库中的components时，使用了这些components的文档将会收到通知，这可以保证设计稿永远指向最新的设计规范。

订阅云端组件库的方式很简单，首先创建一个云端组件库，具体可以参照[上一篇文章](https://tech.meituan.com/2020/05/21/waimai-sketch-plugin.html)，如果需要服务多个设计部门，则需要创建多个库，每个库有唯一的RSS地址；在插件中获取到这些RSS地址后，可以通过Library.getRemoteLibraryWithRSS方法对其进行订阅。

```JavaScript
// 启动插件时添加远程组件库
export const addRemoteLibrary = context => {
  fetch(LibraryListURL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
  })
    .then(res => res.json())
    .then(response => response.data)
    .then(json => {
      const { remoteLibraryList } = json;
      _.forEach(remoteLibraryList, fileName => {
        Library.getRemoteLibraryWithRSS(fileName, (err, library) => {
        });
      });
      return list;
    });
};
```

### 2. Library库文件转换JSON数据

将Sketch的Library文件转换为JSON的过程，实际上就是转换为WebView可以识别格式的过程。因为积木插件是将组件按照一定分组展示在面板中供设计师选取，因此需要根据组件分类组织其结构。Sketch原生支持采用 “/” 符号对其进行分组：Group-name/Symbol-name，比如命名为Button/Normal和Button/Pressed的两个Symbols会成为Button Group的一部分。

![Symbol分组结构](https://p0.meituan.net/travelcube/c0dddf1d8e59703c41da344daa48caff113096.png@628w_348h_80q)

Symbol分组结构



实际中可以根据业务需要采用三级以上分组命名的方式，通过split方法将Symbol名称通过 “/” 符号拆分为数组，第一级名称、第二级名称等各级名称作为JSON结构的不同层级即可，具体操作可以参照如下示例代码：

```JavaScript
 const document = library.getDocument();
    const symbols = [];
    _.forEach(document.pages, page => {
      _.forEach(page.layers, l => {
        if (l.type && l.type === 'SymbolMaster') {
          symbols.push(l);
        }
      });
    });

 // 对symbol进行分组处理，并生成json数据
 for (let i = 0; i < symbols.length; i++) {
      const name = symbols[i].name;
      const subNames = name.split('/');
      // 去掉所有空格
      const groupName = subNames[0].replace(/\s/g, '');
      const typeName = subNames[1].replace(/\s/g, '');
      const symbolName = subNames.join('/').replace(/\s/g, '');
      result[groupName] = result[groupName] || {};
      result[groupName][typeName] = result[groupName][typeName] || [];
      result[groupName][typeName].push({
        symbolID:symbolID,
        name: symbolName,
      });
 }
```

经过以上操作后，一个简化版的JSON文件如下方所示：

```JavaScript
{
    "美团外卖C端组件库": {
        "icon": [{
            "symbolID": "E35D2CE8-4276-45A1-972D-E14A06B0CD23",
            "name": "28/问号"
        },{
            "symbolID": "E57D2CE8-4276-45A1-962D-E14A06B0CD61",
            "name": "27/花朵"
        }]
    }
}
```

### 3. Symbol缩略图处理

WebView默认是不支持直接显示Symbol供用户拖拽使用的，解决该问题的方案有两种：（1）通过dump分析Sketch的头文件发现，可以采用MSSymbolPreviewGenerator的imageForSymbolAncestry方法导出缩略图，该方法支持图片大小、颜色空间等多种属性设置，优势是较为灵活，可以根据需要进行任意配置，不过要承担后期API变更的风险；（2）直接采用sketchDOM提供的export方法，将Symbol组件导出为缩略图，之后在WebView中显示缩略图，当拖拽缩略图至画板时，再将其替换为Library中对应的Symbol即可。

```JavaScript
import sketchDOM from 'sketch/dom';

sketchDOM.export(symbolMaster, {
     overwriting: true,
     'use-id-for-name': true,
     output: path,
     scales: '1',
     formats: 'png',
     compression: 1,
});
```

### 4. 小结

以上就是实现平台化的一个基本流程了，不知道此时你有没有听得“云里雾里”。在这里，再把核心点带大家复习一下。本节主要讲了两件事情：第一，插件如何才能支持多个业务方，即在插件的业务方列表中选择相关业务方，就可以切换对应的设计资源；第二，如何处理Library文件，将其转换为JSON供WebView展示使用。具体流程如下：

1. 不同设计组的UI同学制作完成包含各种components的Library后，通过后台上传至云端。
2. RD同学根据当前使用者所属的设计团队拉取对应的包括Library在内的设计素材，颜色、图片，iconFont等设计素材可以直接展示，可是Library文件不支持在WebView中直接显示，需要进行处理。
3. 根据和UI同学约定组件的命名规则，通过使用“/”分割，将第一级名称、第二级名称等各级名称作为JSON结构的不同层级，再通过sketchDOM提供的export方法将Symbol转换为png格式的缩略图即可在插件中显示。
4. 将选中的缩略图拖拽至Sketch的画板时，再将缩略图替换为Library中的真实Symbol即可。

![Library库文件处理小结](https://p0.meituan.net/travelcube/91e7c4e32b88ffe971af31c41120f2d0271893.png@1920w_1013h_80q)

Library库文件处理小结



## 操作体验优化

完成了上述步骤后，就可以完成一款支持多业务方的插件了。但是随着积木Sketch插件接入的业务方越来越多，除了听到可以显著提升效率的褒奖外，对插件的吐槽声也常常传入我们的耳边。市面上成熟的插件也有很多，我们无法限制别人的选择，所以只能让积木变得更好用，本节主要介绍插件的优化方法。为了更好地倾听大家意见，积木插件通过各种措施了解用户的真实想法。首先积木插件接入美团内部的TT（Trouble Tracker）系统，相比公司很多专业系统，TT不带任何专业流程和定制化，只做纯流转，是一套适用于公司内部的、通用的问题发起、响应和追踪系统，用户反馈的问题自动创建工单并与对应RD关联，Bug可以最快速修复；插件内部增加反馈渠道，用户反馈及时发送给相关PM，作为下次功能排期的权重指标；插件内部增加多维度埋点统计，从设计渗透到高频使用两个方面了解UI同学的核心诉求。以下介绍了根据反馈整理的部分高优先级问题的解决方案。

### 1. 操作界面优化

很多RD在开发过程中，对界面美化往往嗤之以鼻，“这个功能能用就可以了”常常被挂在嘴边。难道UI的需求真的是中看不中用？一个产品设计师说过，最早的产品仅依靠功能就可以在竞品中脱颖而出，能不能用就成为了一个产品是否合格的标准。后来在越来越成熟的互联网环境中，易用性成了一个新的且更重要的标准，这时同类产品间的功能已经非常接近，无法通过不断堆叠功能产生明显差异。而当同类产品的易用性也趋于相近时，如何解决产品竞争力的问题就再一次摆在面前，这时就要赋予产品情感，好的产品关注功能，优秀的产品关注情感，可以让用户在使用中感受到温暖。

**WebView优化**

当我们经过了仔细的功能验证，兴致勃勃的把第一版积木插件给用户体验时，本来满心欢喜准备迎接夸奖，没想到却得到了很多交互体验不好的反馈，“仅仅实现功能就及格了”这个理论在一像素都不肯放过的设计师眼中肯定行不通。原生WebView给用户的体验往往不够优秀，其实只要一些很简单的设置就可以解决，但是这并不代表它不重要。

![WebView视图优化点举例](https://p0.meituan.net/travelcube/92facd7586e1124a796f7a9d0f696b48350315.png@1920w_911h_80q)

WebView视图优化点举例



禁止触摸板拖拽造成页面整体“橡皮筋”效果，禁止用户选择（user-select），溢出隐藏等操作，使WebView具有“类原生”效果，都会提升用户的实际使用体验。

```CSS
html,
body,
#root {
  height: 100%;
  overflow: hidden;
  user-select: none;
  background: transparent;
  -webkit-user-select: none;
}
```

除此之外在正式环境中（NODE_ENV为production），我们并不希望当前界面响应右键菜单，需要通过给document添加EventListener监听将相关事件处理方法屏蔽。

```JavaScript
document.addEventListener('contextmenu', e => {
  if (process.env.NODE_ENV === 'production') {
    e.preventDefault();
  }
});
```

**工具栏优化**

Sketch对于设计师的意义，就像代码编辑器对于程序员一样，工作中几乎无时无刻也离不开。在积木Sketch插件走出美团外卖，被越来越多的设计团队采用后，为了让它更加赏心悦目，UI同学决定对工具条进行一次全新的视觉升级。原生界面开发指的是通过macOS的AppKit进行用户界面开发，在插件开发中一些需要嵌入Sketch面板的UI模块就需要进行原生界面开发，比如吸附式工具条就属于通过macOS原生API开发的界面。

原生开发既可以使用Objective-C语言，也可以使用CocoaScript通过写JavaScript的方式进行开发。CocoaScript 通过Mocha实现JS到Objective-C的映射，可以让我们通过JS调用Sketch内部API以及macOS的Framework。在通过CocoaScript原生开发前需要了解一些基础知识：

1. 在使用相关框架前需要通过framework()方法进行引入，而Foundation以及CoreGraphics是默认内置的，无需再单独操作。
2. 一些Objective-C的selectors选择器需要指针参数，由于JavaScript不支持通过引用传递对象，因此CocoaScript提供了MOPointer作为变量引用的代理对象。

UI调整一般分为三个部分：布局调整、动效调整、图片替换。下面的章节会进行逐一介绍。

![新版积木工具栏效果图](https://p0.meituan.net/travelcube/8762b7d0f3366cff5dde29280fa3be87145229.png@1920w_932h_80q)

新版积木工具栏效果图



**布局调整**

这里UI的需求是NSButton的宽度填充满整个NSStackView，高度自定义。由于此功能看起来过于简单，当时认为估时0.5天绰绰有余，可是没想到搭进去了1个工作日加上2天周末的时间，因为无论如何设置NSStackView中子View尺寸都无法生效。

在顶住了周围人“UI问题不影响功能使用，以后有时间再优化吧”的“舆论压力”后，终于在官方文档里面发现了线索：“NSStackView A stack view employs Auto Layout (the system’s constraint-based layout feature) to arrange and align an array of views according to your specification. To use a stack view effectively, you need to understand the basics of Auto Layout constraints as described in Auto Layout Guide.”简而言之，NSStackView使用constraints的方式进行自动布局（可以类比Android中的ConstraintLayout），在进行尺寸修改时，是需要添加锚点的，因此需要通过Anchor的方式进行尺寸修改。

```JavaScript
// 创建工具条
const toolbar = NSStackView.alloc().initWithFrame(NSMakeRect(0, 0, 45, 400));
toolbar.setSpacing(7);
// 创建NSButton
const button = NSButton.alloc().initWithFrame(rect)
// 设置NSButton宽高
button
    .widthAnchor()
    .constraintEqualToConstant(rect.size.width)
    .setActive(1);
button
    .heightAnchor()
    .constraintEqualToConstant(rect.size.height)
    .setActive(1);
button.setBordered(false);
// 设置回调点击事件
button.setCOSJSTargetFunction(onClickListener);
button.setAction('onClickListener:');
// 添加NSButton至NSStackView中
toolbar.addView_inGravity(button, inGravityType);
```

**动效调整**

NSButton内置的点击效果大约15种，可以通过NSBezelStyle进行设置。积木插件工具栏并没有采用点击后icon反色的通用处理方式，而是点击后将背景色置为浅灰。如果想要自定义一些点击效果，只需在NSButton点击事件的回调中设置即可。

```JavaScript
onClickListener:sender => {
  const threadDictionary = NSThread.mainThread().threadDictionary();
  const currentButton = threadDictionary[identifier];
   if (currentButton.state() === NSOnState) {
      currentButton.setBackgroundColor(NSColor.colorWithHex('#E3E3E3'));
   } else {
      currentButton.setBackgroundColor(NSColor.windowBackgroundColor());
   }
}
```

**图片加载**

Sketch插件既支持加载本地图片，也支持加载网络图片。加载本地图片时，可以通过context.plugin的方法获取一个MSPluginBundle对象，即当前插件bundle文件，它的url()方法会返回当前插件的路径信息，进而帮助我们找到存储在插件中的本地文件；而加载网络图片则更加简单，通过NSURL.URLWithString( )可以获得一个使用图片网址初始化得到的NSURL对象，这里要格外注意的是，对于网络图片请使用https域名。

```JavaScript
//本地图片加载
const localImageUrl = 
       context.plugin.url()
      .URLByAppendingPathComponent('Contents')
      .URLByAppendingPathComponent('Resources')
      .URLByAppendingPathComponent(`${imageurl}.png`);

//网络图片加载
const remoteImageUrl = NSURL.URLWithString(imageUrl);

//根据ImageUrl获取NSImage对象
const nsImage = NSImage.alloc().initWithContentsOfURL(imageURL);
nsImage.setSize(size);
nsImage.setScalesWhenResized(true);
```

### 2. 执行效率优化

只有在设计稿中尽可能多地使用组件进行设计，并且将已有页面中的内容通过设计师的走查梳理逐渐替换成组件，才能真正通过建设组件库来进行提效。随着设计团队逐步将设计语言沉淀为设计规范，并将其量化内置于积木插件中，组件的数量越来越多，积木插件组件库作为UI同学使用最频繁的功能，需要格外关注其运行效率。

**前置组件库加载**

将组件库的加载逻辑前置，在打开文档时对远程组件库进行订阅操作。Sketch所提供的了Action API可以使插件对应用程序中的事件做出反应，监听回调只需在插件的manifest.json文件中添加一个handler即可，添加了对于“OpenDocument”的监听，也就是告诉插件在新文档被打开时要去执行addRemoteLibrary这个function。

```JSON
{
      "script": "./libraryProcessor.js",
      "identifier": "libraryProcessor",
      "handlers": {
        "actions": {
          "OpenDocument": "addRemoteLibrary"
        }
      }
}
```

**增加缓存逻辑**

组件库的处理需要将Library文件转换为带有层级信息的JSON文件，并且需要将Symbol导出为缩略图显示。由于这个步骤较为耗时，因此可以将经过处理的Library信息缓存起来，并通过持久化存储记录已缓存的Library版本。若已缓存的版本与最新版本一致，且缩略图与JSON文件均完整，则可以直接使用缓存信息，极大的提高Library的加载速度。以下非完整代码，仅作示例：

```JavaScript
verifyLibraryCache(businessType, libraryVersion) {
    const temp = Settings.settingForKey('libraryJsonCache');
    const libraryJsonCache = temp ? JSON.parse(temp) : null;

    // 1.验证缓存版本信息
    if (libraryJsonCache.version.businessType !== libraryVersion) {
      return null;
    }

    // 2.验证缩略图完整性
    const home = getAssertURL(this.mContext, 'libraryImage');
    const path = join(home, businessType);
    if (!fs.existsSync(path) || !fs.readdirSync(path)) {
      return null;
    }

    // 3.验证业务库Json文件完整性
    if (libraryJsonCache[businessType]) {
      console.info(`当前${businessType}命中缓存`);
      return libraryJsonCache;
    } else {
      return null;
    }
  }
}
```

### 3. 自定义Inspector属性面板

**与Objective-C工程混合开发**

随着各个设计组的组件库建设不断完善，抽离的组件数量不断增多，不少UI同学反馈Sketch原生组件样式修改面板操作不够便捷，无法约束选择范围，希望可以提供一种更有效的组件overrides修改方式，并且当修改“图片”、“图标”、“文字”等图层时，可以和积木插件的这些功能模块进行联动选择。实现自定义Inspector面板功能既可以使操作更便捷，又可以对修改项进行约束。

自定义属性面板功能的基本思想，是将组件从组件库拖至Sketch画板中时，组件的可修改属性可以显示在Sketch本身的属性面板上。我们引入了Objective-C原生开发以实现对Sketch界面的修改，为什么要使用原生开发？虽然官方提供了JS API并承诺持续维护，但这项工作一直处于Doing状态，而且官方文档更新缓慢，没有明确的时间节点，因此对于自定义Native Inspector Panel这种需要Hook API的功能，使用原生开发较为便捷，而且对于iOS开发者也更加友好，无需再学习前端界面开发知识。

![Sketch Inspector面板操作区优化](https://p0.meituan.net/travelcube/b4e858a01a9336affd1d3b4d8d9764bc1126400.png@1920w_1106h_80q)

Sketch Inspector面板操作区优化



**Xcode工程配置**

通过Xcode工程构建自定义属性面板，最终生成一个可以供JS侧调用的Framework。可以参考上一篇文章介绍的方法创建Xcode工程，该工程在每次构建后会自动生成测试Sketch插件并放入对应的文件夹中。需要注意的一点是，这里生成的插件只是为了方便开发和调试，后面会介绍如何将XCode工程构建的Framework集成至JS主工程中。

![Xcode工程配置示意](https://p0.meituan.net/travelcube/0f5d9d8167ca2eb856851e9c9e401a60116737.png@1920w_506h_80q)

Xcode工程配置示意



积木插件的主体功能使用JS代码实现，但是自定义属性选择面板使用Objective-C代码实现。为了实现积木插件的JS侧功能模块与OC侧模块之间的通信和桥接，这里借助了Mocha框架来实现相关的功能，Mocha框架也被Sketch官方所使用，将原生侧的方法封装为官方API后暴露给JS侧。

![Sketch与插件Framework通信原理](https://p0.meituan.net/travelcube/3ec26b83ef0278c0e284bdea04452455145356.png@1920w_920h_80q)

Sketch与插件Framework通信原理



组件选中时，Sketch软件会回调onSelectionChanged方法给JS侧，JS侧借助Mocha框架可以实现对OC侧的调用，同时将参数以OC对象的方式传递。JS侧传递给OC侧的Context内容很丰富，包含了选中的组件、相关图层还有Sketch软件本身的信息。虽然Sketch没有提供API，但是Objective-C语言本身具备KVO监听对象属性的能力，我们通过读取对应的属性值，就可以获取需要的对象数据。

```Objective-C
+ (instancetype)onSelectionChanged:(id)context {
 
    [self setSharedCommand:[context valueForKeyPath:@"command"]]; 
   
    NSString *key = [NSString stringWithFormat:@"%@-RooSketchPluginNativeUI", [document description]];
    __block RooSketchPluginNativeUI *instance = [[Mocha sharedRuntime] valueForKey:key];

    NSArray *selection = [context valueForKeyPath:@"actionContext.document.selectedLayers"];
    [instance onSelectionChange:selection];
    return instance;
}
```

Sketch官方没有将属性面板的修改能力暴露给插件侧，通过查询Sketch头文件发现通过reloadWithViewControllers:方法可以实现属性面板刷新，但是在实际开发过程中发现在某些版本的Sketch上会出现面板闪动的问题，这里借助Objective-C的[Method Swizzle](https://juejin.im/post/6844903856497754126)特性，直接修改reloadWithViewControllers:的运行时行为解决。

```Objective-C
[NSClassFromString(@"MSInspectorStackView") swizzleMethod:@selector(reloadWithViewControllers:)                                        withMethod:@selector(roo_reloadWithViewControllers:)                                                        error:nil];
```

Swizzle方法会修改原始方法的行为，实际操作中只有在满足特定条件的情况下才应触发Swizzle后的方法。

![Swizzle方法触发条件](https://p0.meituan.net/travelcube/d4a4ee150c57edd073a0d261bee5d8e0122926.png@1650w_1194h_80q)

Swizzle方法触发条件



**组件属性修改与替换原理**

通过自定义面板可以修改组件的可覆盖项（即override），目前可以应用可覆盖项的affectedLayer有Text/Image/Symbol Instance三种。设计师与开发者在此前对图层的格式进行了约定，保证我们可以按照统一的方式读取并替换图层的属性值。

**替换文本**

基于class-dump，我们可以找出Sketch中声明的所有类的属性和方法，文本处理的策略是，找到图层中的所有MSAvailableOverride对象，这些对象即表示可用的覆盖项，对文本信息的修改实际上是通过修改MSAvailableOverride对象的overridePoint来实现的。

```Objective-C
id overridePoint = [availableOverride valueForKeyPath:@"overridePoint"];
[symbolInstance setValue:text forOverridePoint:overridePoint];
```

**更改样式**

样式设置的策略，是找到当前选中组件对应的Library中相关样式的组件。由于所有的组件都遵循统一的命名格式，因此只要根据组件命名就能筛选出符合要求的组件。

```Objective-C
// 命名方式：一级分类/二级分类/组件名称，基于图层获取对应library
id library = [self getLibraryBySymbol:layer];
// 读取组件名称
NSString *layerName = [symbol valueForKeyPath:@"name"];
// 配置符合当前业务的Predicate
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name BEGINSWITH [cd] %@", prefix];
// 筛选符合Predicate的所有组件
NSArray *filterResult = [allSybmols filteredArrayUsingPredicate:predicate];
```

当使用者选中某一个样式后，插件会将设计稿上的组件替换为选中的组件，这里需要使用MSSymbolInstance中的changeInstanceToSymbol方法来实现。需要注意的是，changeInstanceToSymbol仅仅替换了图层中的组件，但是并没有修改图层上组件的属性，对于位置和大小等信息需要单独进行处理。

```Objective-C
// 在更新图层上的组件之前，我们需要把组件导入到当前document对象中
id foreignSymbol = [libraryController importShareableObjectReference:sharedObject intoDocument:documentData];

//更新图层上的组件 
[symbolInstance changeInstanceToSymbol:localSymbol];
```

**调试技巧**

OC侧开发的最大问题，在于没有官方API的支持。因此调试器就显得非常重要，单步调试可以让我们非常方便地深入到Sketch内部了解Document内部的数据结构。调试环境需要配置，但足够简单，并且对于开发效率的提升是指数级的。

1.对构建Scheme的配置。

![img](https://p0.meituan.net/travelcube/ab11fc1fc91c568fe12e8bfe5ec978b990897.png@1716w_500h_80q)

2.Attach到Sketch软件上，这样就可以实现断点调试。

![img](https://p0.meituan.net/travelcube/e12e78cf0236842a64a0455ce4d55f23390168.png@1562w_994h_80q)

**与当前JS工程混合编译**

1.通过skpm中内置的@skpm/xcodeproj-loader编译XCode工程，并将产物framework拷贝至插件文件夹。

```Objective-C
const framework = require('../../RooSketchPluginXCodeProject/RooSketchPluginXCodeProject.xcworkspace/contents.xcworkspacedata');
```

2.通过Mocha提供的loadFrameworkWithName_inDirectory方法，设置Framework的名称及路径即可进行加载。

```JavaScript
function() {
    var mocha = Mocha.sharedRuntime();
    var frameworkName = 'RooSketchPluginXCodeProject';
    var directory = frameworkPath;

    if (mocha.valueForKey(frameworkName)) {
      console.info('JSloadFramework: `' + frameworkName + '` has loaded.');
      return true;
    } else if (mocha.loadFrameworkWithName_inDirectory(frameworkName, directory)) {
      console.info('JSloadFramework: `' + frameworkName + '` success!');
      mocha.setValue_forKey_(true, frameworkName);
      return true;
    } else {
      console.error('JSloadFramework load failed');
      return false;
    }
  }
```

3.调用framework中的方法。

```Objective-C
// 找到已经被加载的framework
 const frameworkClass = NSClassFromString('RooSketchPluginNativeUI');
// 调用暴露的方法
frameworkClass.onSelectionChanged(context);
```

## 一起拼积木

目前，积木插件已经在美团到家事业部遍地开花，我们希望未来积木品牌产品可以在更大范围内得到应用，帮助更多团队落地设计规范，提升产研效率，也欢迎更多团队接入积木工具链。“不忘初心，方得始终”，就像第一篇启蒙文章中说的那样，我们除了希望制作一流的产品，也希望积木插件可以让大家在繁忙的工作中得以喘息。我们会继续以设计语言为依托，以积木工具链为抓手，不断完善优化，拓展插件的使用场景，让设计与开发变得更轻松。

总有人在问，积木插件现在好用吗？我想说，还不够好用。但是每次评审需求时看到旁边的设计师在认真地使用我们的插件作图，看到积木插件爱好者为我们制作表情包帮助我们推广，我们深知唯有交付最棒的产品，才能不辜负大家的期待。

