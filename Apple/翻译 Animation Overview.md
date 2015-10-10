# Introduction to Animation Overview
这篇文档目的是给 OS X 中的程序员基本介绍下 OS X 中提供的动画能力。

这篇文档是为所有有兴趣在 Cocoa 应用中使用动画的开发者准备的。如果你没有动画的背景知识，它是一个不错的开始点，但是你应该对 Cocoa 应用使用视图有基本的概念。通常你不需要对图形编程有任何理解，尽管有这些理解将会很有用。

如果你是一个高级 Cocoa 开发者，并已经对于动画技术很熟悉，你应该读这本书来获得关于在 Cocoa 应用中使高效用动画的一些指示，和稚嫩杨保证你的动画和 OS X 的用户体验高效的集成。

## Organization of This Document
这篇文档包含以下章节：

* **What Is Animation?** 描述了动画是什么和怎样在 OS X 中使用它们。
* **Animation Basics** 描述了动画的基本组成
* **OS X Animation Technologies** 描述了 OS X 为动画提供的技术
* **Choosing the Animation Technology for Your Application** 探讨了 Core Animation Layer 提供的强项和 layer-backed views，并提供了关于选择哪种技术的指导。

## See Also
关于 Core Animation 和 Cocoa 动画类的更多信息，可以查看以下资源：

* *Core Animation Programming Guide* 提供了集成 Core Animation 到你的应用的具体信息。
* *Animation Programming Guide for Cocoa* 描述了 Application kit 提供的动画类和协议。

----
# What Is Animation?
动画是一个视觉技术，通过以快速的次序展示一系列的图片来提供运动的视觉。每张图片包含一个很小的变化，例如一条腿微微的移动，或一辆汽车的轮子转动。当图片快速的被查看的时候，你的眼睛充满细节，运动的视觉就完成了。

当在应用的用户界面中合适的使用动画，动画可以在提供一个更动态的视觉和感受的同时加强用户体验。平滑的在屏幕上移动 UI 元素，渐现或渐隐，创建新的有特别视觉效果的自定义控件，这些组合起来可以为用户创建一个剧院式计算体验。

## Animation in OS X
OS X 的 UI 中有需要样例动画：

* Dock 图片来回跳动表示一个应用需要用户的注意。
* Sheets and drawers use animation as they are displayed and hidden.
* 进度条动画暗示一个长的操作正在进行
* Dragging icons to the dock causes icons to “part” to allow the new icon to be inserted.
* The default button in a panel pulses.

应用也可以使用动画来提供一个丰富的，动态的用户体验。例如，iChat 播放一个声音，然后使用动画来显示一个好友不再有空。声音提醒你这个状态变化，动画给你机会专注于应用来分析这个事件。

Front Row 在它的用户界面里到处都在使用动画。例如当 Front Row 被激活时，一个 transition 被用来显示模式由 Mac OS Aqua 界面转换到 Front Row 界面。当你导航到更深的菜单层次，新的菜单从右边压入，老的菜单被压入左边。当你返回上层菜单时，transition 是相反的，同样帮你维持了上下文。

在 Time Machine 中，视觉上的距离为时间上的线性过程提供了一个比喻。旧的系统快照显示得远些，允许你移动穿过它们来发现你想要恢复的版本。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationOverview/time_machine_user_interface.png" width=80% />

iTunes 7.0 CoverFlow 界面以一种吸引人的方式显示了专辑和电影封面。随着你浏览这些内容图片动画到直面用户。这是一个 cinematic computing exprience 的不错例子。相对于图片一张挨着一张显示来说，CoverFlow 用户界面允许更多的内容在更小的区域显示。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationOverview/itunes_7_0_coverflow_user_interface.png" width=80% />

即使很少的使用动画也可以有很好的交互作用。`iSync` 菜单额外的动画显示了同步正在进行。在所有的情况下应用通过使用动画向用户提供了额外的信息和上下文。

## Using Animation in Your Applications
你怎么使用动画到你自己的应用很大程度上依赖于你的应用提供的接口类型。使用 Aqua UI 的应用可以通过创建自定义控件或视图来集成动画。创建它们自己的 UI 的应用，在决定多少动画对你的用户合适的时候有更多的余地。

明智的使用动画和视觉效果，对于用户而言最平凡的系统工具都可以成为一个丰富的视觉体验，对于你的应用而言这提供了一个引人入胜的竞争优势。

---
# Animation Basics
对于所有的动画有些必须的基本属性：它们必须有一个关联的对象来做动画，它们必须定义将要进行什么样的类型的动画和动画将要持续多久。

这章以抽象的方式讨论了对于 OS X 动画技术而言很普遍的基本动画技术。

## Animation Target Object
每个动画必须会动画将要影响的一个视觉元素相关联。你可以把这个对象认为看作是动画的目标对象。动画目标对象提供了展示给用户的内容。一个动画对象要作用在整个对象上，要么在目标对象的某个属性上。例如，它在坐标系中的坐标或它绘制时所用的颜色。

动画是与一个动画目标对象相关联的。你不用显式的启动一个动画；动画的目标对象启动和停止动画。

## Types of Animation
OS X 动画支持 3 种不同类型的的动画：基本动画，关键帧动画，和转场动画。

### Basic Animation
**Basic animation**——也被称作简单动画或单帧动画——从开始值逐步到目标值途径一系列中间值。这些中间值被计算出，插入，以便动画可以在指定时间内发生。基本动画要求你制定一个目标对象要做动画的属性。

基本动画适用于任何可以做插值运算的值类型。包括：

* 整形和 doubles
* `CGRect`, `CGPoint`, `CGSize`, `CGAffineTransform` 结构
* `CATransform3D` 结构
* `CGColor` 和 `CGImage` 引用

### Keyframe Animation
**Keyframe animation** 和基本动画相似；然后它允许你指定一组目标值。在动画的过程中这组值中每个值依次到达。帧动画的关键帧可以是任何基本动画所支持的类型，或一个 Core Graphics 路径，这个路径会被分解一些列的 `CGPoint` 对象。像一个基本动画，一个关键帧动画要求动画作用在一个动画目标对象的指定属性上。

### Transition Animation
转场动画指定了一种视觉效果，这种视觉效果决定了在动画目标对象变得可见或隐藏的时候，它是怎么样被显示的。例如，使得一个动画目标对象可见可能使得它被从显示的一边压入视图。转场动画是通过使用 **Core Image** filters 实现的。

因为转场动画影响整个动画目标对象，没有必要指定一个目标对象的属性。

## Animation Timing
一个动画的总时长由好几个因素决定：duration，pacing，和 repeating 行为。

**Duration** 是一个动画从起点或当前状态到目标状态所花时长，以秒为单位。

**Pacing** 决定了插值怎么在动画的时长内分布。例如，一个单关键帧动画从 0 到 5， 时长 5s，线性 pacing 使得插值在时长内均匀分布。同样的动画使用一个 "EaseIn" 的话，开始的时候会慢些，然后随着动画的进行而加速。动画花费相同的时长并达到相同的值，但是每个插值到达的时间不同。

动画重复可以通过两种方式指定： 通过一个简单的计数来指定应该重复多少次，或设置一个时长来指定应该重复多久。例如，指定一个  5s 时长的动画重复 15s 时长使得动画重复 3 次。你也可以指定一个动画在重复时先前进，后倒退。

----
# OS X Animation Technologies
在一个应用中加入老道的动画和视觉效果是比较困难的，通常要求开发者使用底层的图像相关的 API，如 OpenGL 来获得可接受的动画性能。OS X 提供了一些技术使得通过一些高层 API 给 Cocoa 和 Carbon 应用创建丰富的，平滑过渡的应用 UI 界面变得更容易。

## Core Animation
Core Animation 是一个 Objective-C 框架，最初在 OS X v10.5 引入，它允许你通过添加实时动画和视觉变换来巨大的增加你的应用的生产价值，而不需要知道秘密的图形和数学技巧。使用 Core Animation 你可以同时动态的渲染和给 text, 2D graphics, OpenGL, Quartz Composoer 组合和 QuickTime 视频，兼有透明效果和 Core Image filters 和 effects。

在它的核心，Core Animation 是一个高速的 2D layering engine。你通过分解用户界面成单独的 layers 来创建一个 Core Animation 接口。通过组合这些 layers 到一起你就创建了最终的用户界面。(这里原文有个视频，但在原文的链接中没有找到，所以这里这部分就不译了)

Layers 是以嵌套的树的形式组织的——树对于在 Cocoa 应用中使用过视图的开发者而言应该很熟悉。每个可见的 layer 树都有一个相应的 render tree，这个 render tree 负责缓存 layer 的内容并根据要求尽快的组合 layer 到屏幕。你的应用不需要进行昂贵的重绘除非内容有所改变。

除了 layer 的基本位置和几何，每个 layer 也提供可选的视觉属性，这些属性在 layer 的内容渲染时被应用上，它们包括：

* 一个可选的背景色
* 一个可选的圆角半径，允许 layer 显示圆角
* 一个可选的 Core Image filters 数组，在 layer 的内容被组合之前应用到在 layer 内容之后的内容
* 一个 Core Image filter 应用来组合 layer 的背景和内容
* 一个可选的 Core Image filters 被应用到 layer 和它的 sublayers 的内容
* 控制一个 layer 的透明度
* 用来绘制一个可选阴影的参数，包括：颜色，offset，opacity，blur radius
* 使用指定的 line width 和 color 绘制一个可选的边框

尽管 Core Animation 是一个 2D layerying engine，但它提供了使用 projective transformations 来传递 3D 场景的支持。每个 layer 有一个 3D transform matrix 被应用于它的内容，和一个 3D transfrom matrix 用于它的 sublayers。使用这些 projections，你可以创建传递深度给用户的界面。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationOverview/core_animation_3d_interface.png" width=80% />

### Animating Layers
Core Animation 提供了两种方式支持对一个 layer 的可视属性做动画：显示的和隐式的。

隐式动画会自动发生以响应给一个支持动画的属性设置新值。Core Animation 全权负责在一个单独线程以一定的帧率跑一个动画，使得应用可以自由的处理其他事件。例如，通过赋 `newFrame` 值到 `frame` 属性，`textLayer` 对象平滑的动画到新的位置。

```objc
textLayer.frame=newFrame;
```

每个 layer 的可做动画属性都有一个相关联的隐式动画。你可以通过覆盖一个默认动画来给一个 layer 属性提供你的自定义动画。你也可以暂时或用就关闭任何 layer 属性的动画。

显式动画是通过创建一个 Core Animation 动画类的实例并指定一个目标 layer 和一个可选的属性完成的。显式给一个 layer 的属性做动画只会影响这个属性的 presentation 值。这个 layer 属性的真实值不会改变。例如，要吸引注意力到一个 layer，你可以通过给 transformation matrix 做动画来使它旋转 360°。这个动画会影响这个 layer 的显示，但它的 transform matrix 保持不变。

### Layout Management
Core Animation layers 支持相对于它们 superlayer 定位 layer 的经典的 Cocoa 视图模型——一种被称为 "springs and struts" 的样式。另外 Core Animation 也提供了一个更普遍的 layout manager 机制，它允许你编写你自己的 layout managers。一个自定义 layout manager 会负责提供关联 layer 的 sublayers 的布局。

使用一个自定义 layout manager，你的应用可以通过使用隐式动画创建复杂动画。更新一个 layer 的位置，大小，或 transform matrix 使得动画到新的设置。CoverFlow 这种样式的动画是通过一个自定义 layout manager 完成的。

Core Animation 提供了一个基于约束的 layout manager 类，这个类使用一系列的 constraints 来安排 layers。每个指定的 constraint 描述了一个 layer 的一个几何属性 (上下左右边，或水平或竖直中心) 与它的一个兄弟 layer 或 superlayer 的一个几何属性的关系。例如，使用 constraint layout manager 你可以定义 layerA 总是居中，并在 layerB 下面 5 points。移动 layerB 到一个新位置的话自动使得 layerA 相对于重新定位。

### NSView Integration with Core Animation
Cocoa `NSView` 类以两种方式与 Core Animation 和 layers 集成。第一种集成方式是 *layer hosting*。一个 layer-hosting  视图显示一个应用的 Core Animation layer 树。直接操作 layer 来与 layer 的内容交互是应用的责任。一个 layer hosting 的视图正是一个应用怎么在展示 Core Animation 用户接口。

你通过以下代码指定视图 `aView` 为 Core Animation layer `rootLayer` 的 layer host:

```objc
// aView is an existing view in a window// rootLayer is the root layer of a layer tree
[aView setLayer:rootLayer];[aView setWantsLayer:YES];
```

当使用一个 layer-hosting 视图时，你通常继承 `NSView` 并在子类中处理诸如按键和鼠标点击类的事件，根据需要操作 Core Animation layers。

`NSView` 和 Core Animation 集成的第二种类型是 *layer-backed* 视图。Layer-backed 视图使用 Core Animation layers 作为它们的 backing store，从而不需要视图来刷新屏幕。视图只在它的内容真的发生改变时重绘。为一个视图及它子类启用 layer-backing 可以使用以下代码：

```objc
// aView is an existing view in a window
[aView setWantsLayer:YES];
```

当为视图启用 layer backing 的时候，视图和它所有的姿势图会有一个 Core Animation layer tree 的镜像。视图可以是标准的 Aqua 空间或自定义视图。视图和它的子视图仍然参与到响应链，仍然接受事件，并且和其它视图行为一样。然而，当需要完成重绘并且内容并没有发生改变时，render tree 会处理重绘而不是应用自己。

除了提供缓存的重绘外，layer-backed 视图暴露了一些Core Animation layer 属性的高级视觉属性：

* 控制视图的 opacity
* 一个可选的阴影，通过 `NSShadow` 对象指定
* 一个可选的 Core Image filters 数组，这些 filters 会应用到视图背后的内容，在它的内容被组合之前
* 一个 Core Image filter 被用来组合视图的内容和背景
* 一个可选的 Core Image filter 数组被应用到视图和它的子视图的内容

> **注意：**当在 layer-backed 模式下使用一个视图的时候，你不应该直接操作这些 layers，相反使用 `NSView` 暴露的 layer 方法。

关于 layer-backed 视图的更多信息，可以查看 *View Programming Guide*

## Cocoa Animation Proxies
从 Core Animation 中获得提示，Application Kit 通过使用 **animation proxies** 来支持 the concept of a generalized animation facility。

支持 `NSAnimatablePropertyContainer` 协议的类对任何 KVC 兼容的属性提供了隐式动画支持。通过 animation proxy 的操作，你可以像你在 Core Animation 中做动画那样进行隐式动画。

例如，下面的代码片段使得 `aView` 显示到新的位置：

```objc
[aView setFrame:NSMakeRect(100.0,100.0,300.0,300.0)];
```

然而，使用一个 animation proxy 和同样的视图方法，`aView` 以动画的形式平滑的过度到它新的位置。

```objc
[[aView animator] setFrame:NSMakeRect(100.0,100.0,300.0,300.0)];
```

和 Core Animation 不像的是，Core Animation 只允许那些给能直接映射到 render tree 属性的属性做动画，Application Kit 允许给任何 KVC 兼容的属性做动画，只要它的类实现 `NSAnimatablePropertyContainer` 协议。给这些属性做动画不需要视图重绘它的内容，但是仍然比手动给这些属性做动画容易很多。和 Core Animation layer 一样，你可以改变这些可做动画属性的默认隐式动画。

> **注意**：视图的 `float`, `double`, `NSPoint`, `NSSize`, `NSRect` 类型属性使用 `NSAnimatablePropertyContainer` 方法 `animator` 返回的 proxy 做动画时不是必须需要视图是 layer-backing 的。

关于 Cocoa 动画的更多信息，可以查看 *Animation Programming Guide for Cocoa*。

### Transitioning to and from Full Screen Mode
当一个视图在全屏模式来回转动时，`NSView` 类提供方法允许你简单的提供一个 transition 动画。`enterFullScreenMode:withOptions:` 方法允许你指定视图将接管的目标屏幕。退出全屏是通过 `exitFullScreenModeWithOptions:` 方法完成的。你可以通过 `isInFullScreenMode` 测试一个视图是否在全屏模式。

### Additional Animation Support
`NSAnimation` 和 `NSViewAnimation` 类相对于 `NSAnimatablePropertyContainer`协议而言提供了一个更简单，但并不强大的动画能力。

`NSAnimation` 提供了基本计时功能和过程曲线计算，一个以简单方式影响动画行为的 delegate 机制，和将动画部分串起来的设施。

`NSViewAnimation` 是 `NSAnimation` 的具体子类，提供了使用预定义的四个过程曲线的之一来给视图的 frame 做动画，和进行 fade-in 和 fade-out 效果。 View animations may be blocking, asynchronous, or threaded asynchronous, but successive incremental changes as view frame coordinates are interpolated take effect immediately in the view instances themselves.

如果一个应用是为 OS X v 10.5 起的版本开发的，你会发现 `NSWindow` 和 `NSView` 类提供的 animation proxies 通常是方便使用，相对于直接使用 `NSAnimation` 而言。

`NSAnimation` 和 `NSViewAnimation` 在 OS X v10.4 才引入。关于 `NSAnimation` 和 `NSViewAnimation` 的更多信息，可以查看 *Animation Programming Guide for Cocoa*

---
# Choosing the Animation Technology for Your Application
你的应用应该选择哪个动画技术使用很具有挑战。

这章提供的引导将帮助你决定你的应用是否应该使用 Core Animation，non-layer-backed Cocoa 视图，还是 layer-backed Cocoa 视图，或这些技术的一个组合。它也总结了 Core Animation layers 和 layer-backed Cocoa 视图提供的能力。

## Guidance
当添加动画到你的应用的时候，下面的指导将帮助你选择合适的技术：

* 对于只需要静态、非旋转、Aqua 控件的 UI 部分，使用 Cocoa views
* 对于需要对视图或 window 的 frame 做动画的 UI 部分，考虑使用视图和 window 支持的 animator proxy。
* 对于用户中需要显示和与 rotated Aqua  控件交互的部分，使用 layer-backd Cocoa views
* 如果性能测试表明你的自定义视图是处理器或时间密集的，考虑使用 layer-backed  Cocoa 视图。记住 layer-backed 视图比 non-layer-backed 视图使用更多的内存。
* 如果你的自定义视图需要处理了很多的用户时间，考虑使用 layer-backed Cocoa 视图。
* 对于有相当图形和动画要求并且不依赖 Aqua 控件的 UI 部分，考虑直接使用 Core Animation layers。通过覆盖 view host 的 root layer直接使用 layers 不要求你处理所有用户事件 (鼠标和键盘交互)。

另一方面 Core Animation layers 相对于 Cocoa 视图实例计算上更轻量级。这允许你在使用 Core Animation layers 时从小的组成部分来创建接口。当你的应用需要显示由几千个 layer 对象组成的层次时，这也给了 Core Animaiont 另一个优势。

## Hybrid View/Core Animation Applications
当你在一个用户界面中混合使用 layer-backed 视图，non-layer-backed 视图，和 layer-hosting 视图时需要注意两点：

* layer-backed 视图的所有子视图自动 layer backed。为了提升性能，如果你不需要动画或缓存的绘制的话，你应该避免使视图成为 layer-backed 视图子视图。
* 你必须 host 你的自定义 Core Animation layer 到一个单一 layer-hosting 视图而不是插入它们到一个 layer-backed 视图管理的 layer 层次中。

### Example: A Hybrid Application
`CocoaSlides` 是一个 hybrid view/layer-backed 视图 UI 样例。下图展示视图层次的相关部分。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationOverview/hybrid_layer-backed_view_and_conventional_view_user_interface.png" width=80% />

`AssetCollectionView` 视图是一个 layer-backed 视图，因此 `SlideCarrierView` 的所有子视图都是 layer-backed。Layer-backed 视图被用来作为这部分 UI 的原因是：

* The slides require animation that includes rotation; 这就排除了 non-layer-backed 视图，因为视图视图动画 proxies 不支持 rotation，除非视图是 layer-backed。
* Slides 使用了标准的 Aqua checkbox；这个事实直接排除了 Core Animation，因为 Core Animation 不支持 Aqua 控件。以一个 Core Animation layer 替换一个 Aqua 控件的围观和感觉是不推荐的；当然这在未来的 OS X 版本中可能改变。 <br />使用一个 non-layer-backed 视图不是一个选项； Aqua checkbox 必须可用，即使是当旋转了。
* slides 绘制起来很复杂；通过使用一个 layer-backed 视图；slide 内容只在它发生改变时才重新绘制。

window 底部 UI 部分是由标准的 Aqua 控件组成，它们不是 layer-backed。这些控件没有需要在一个 layer-backed 视图层次中；它们永远不会被旋转，也不会从 cached drawing 中获得任何好处。

## Summary of Layer and View Capabilities
下面的部分是 Core Animation layers 和 layer-backed 视图共有的一些能力的简单总结，包括彼此相对于彼此的优点。

### Common Capabilities
* layers 和 layer-backed 视图都支持一个嵌套层次模型，这个模型使用相对于父亲的坐标系统。
* layers 和 layer-backed 视图支持缓存的内容，重绘时不需要应用的交互
* layers 和 layer-backed 视图支持全类型的 media 类型
* layers 和 layer-backed 视图支持超过 layers/views 边界的 sublayer/subview 覆盖它自己。

### Layer Advantages
* layers 支持一些视觉属性，这些属性在 layer-backed 视图中没有暴露。例如，corder radius 和 borders。
* Sublayers 可以位于 layer 的边界外部
* Layers 支持复杂的 masking。
* Layers 支持复杂的 transform。
* Layout manager 对 layer 的定位支持更灵活的控制

### Advantages of Layer-Backed Views
* Layer-backed 视图会参与到 responder chain 中
* Layer-backed 视图支持 drag and drop
* Layer-backed 视图支持 accessibility
* Layer-backed 视图支持所有的标准 Aqua 控件和视图
* Layer-backed 视图支持 scrolling
* Layer-backed 视图支持文本输入