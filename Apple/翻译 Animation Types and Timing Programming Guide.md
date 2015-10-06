# Introduction to Animation Types and Timing Programming Guide
这篇文档描述了使用 Core Animation 时涉及到的计时 (timing) 和动画类等基本概念。Core Animation 是一个 Objective-C 框架，既包含一个高性能的 compositing engine，又提供了一个简单易用的动画编程接口。

> **注意**: 动画本质上是一个视觉媒介。*AnimationTypesandTiming Programming Guide* 的 HTML 版本包含 QuikTime 视频 (伴随静态图片) ，这些视频展示了样例动画来帮助说明概念。PDF 版本只包含静态图片。

你应该阅读这篇文档来获得对于在一个 Cocoa 应用使用 Core Animation 的理解。Objective-C 2.0 编程语言应该被认为是先决条件，因为 Core Animation 大量的使用 Objective-C 属性。你也应该熟悉 *Key-Value Coding Programming Guide* 中描述的 KVC。熟悉 *Quartz 2D Programming Guide* 中描述的 Quartz 2D 图像技术也是有帮助的，尽管没有要求。

## Organization of This Document 
*Animation Types and Timing* 由以下文章组成：

* **Animation Class Roadmap** 提供了动画类和计时协议的概览
* **Timing, Timespaces, and CAAnimation** 具体的描述了 Core Animation 的即使模型和抽象类 `CAAnimation`。
* **Property-Based Animations** 描述了 property-based 动画：`CABasicAnimation` 和 `CAKeyframeAnimation`
* **Transition Animation** 描述了转场动画类，`CATransition`

## See Also
这些编程指导讨论了 Core Animation 使用的一些技术：

* *Animation Overview* 描述了 OS X 中可用的动画技术
* *Core Animation Programming Guide* 包含一些片段，展示了普遍的 Core Animation 任务
* *Animation Programming Guide for Cocoa* 描述了 Cocoa 应用可用的动画能力

----
# Animation Class Roadmap
Core Animation 提供了富于表现动画类集合，你可以在你的应用中使用它们：

* `CAAnimation` 是一个抽象类，所有的动画类都继承它。`CAAnimation` 实现了 `CAMediaTiming` 协议，这个协议为动画提供了简单的时长，速度，重复次数。`CAAnimation` 同时也实现了 `CAAction` 协议。这个协议提供了标准的途径来启动一个动画以响应一个 layer 对象触发的 action。<br /> `CAAnimation` 类也定义了一个动画的计时——是一个 `CAMediaTimingFunction` 实例。这个计时函数通过一个 Bezier 曲线来描述一个动画的步速。一个线性函数指定了一个动画的节奏在它的过程中均衡的，但是一个 ease-in 的计时函数会使得一个动画在时长结束的时候加速
* `CAPropertyAnimation` 是 `CAAnimation` 的抽象子类，提供了对 key path 所指定的 layer 属性做动画的支持。
* `CABasicAnimation` 是一个 `CAPropertyAnimation` 的子类，为一个 layer 属性提供了简单的填充实现。
* `CAKeyframeAnimation` (一个 `CAPropertyAnimation` 的子类) 提供了关键帧动画的支持。你指定要做动画的 layer 属性的 key path，一组代表动画每个阶段值的数据，和一组关键帧时间点和计时函数。随着动画运行，每个值按照指定的插值按序的被设置。
* `CATransition` 提供了转场效果，这个转场效果会影响整个 layer 的内容。动画时会 fade、pushes、reveal layer 的内容。在 OS X 上，stock transition effects 可以通过提供你自定义的 Core Image filters 来扩展。
* `CAAnimationGroup` 允许一组动画对象组合在一起并行的运行。

下图展示动画对象的类层次，也总结了通过继承而可用的属性。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationTypesAndTiming/core_animation_classes_and_protocol.png" width=80% />

---
# Timing, Timespaces, and CAAnimation
当把一个动画分解到它最简单的定义的时候，它只是一个值随着时间简单的变化。Core Animation 为动画和 layers 提供了基本计时功能，这个计时功能提供了强大的 timeline capability。

这章讨论了对于所有动画子类很普遍的 timing protocol 和基本动画支持。

## Media Timing Protocol
Core Animation 计时模型是在 `CAMediaTiming` 协议中描述，被 `CAAnimation` 和它的子类实现。计时模型指定一个动画的 time offset，duration，speed，and repeating behavior。

`CAMediaTiming` protocol 也被 `CALayer` 类实现，这就允许一个 layer 定义一个相对于 super layer 的时间空间；跟描述相对坐标空间一样的。这种 layer 树的时间空间概念提供了一个从根 layer 开始通过它的后代的可扩展的时间线 (timeline)。由于一个动画必须要关联到一个 layer 才会有显示，一个动画的计时可以被 layer 定义的 timespace 伸缩。

一个动画或 layer 的 `speed` 属性指定了这个伸缩因子。例如，一个 10s 的动画附上一个 timespace 中 speed 值为 2 的 layer，动画只会花 5s 时间显示 (两倍速度)。如果这个 layer 的一个 sublayer 也定义了一个 speed factor 为 2 的时间空间，那么它的动画只会显示 1/4 时长 (superlayer 的 speed * sublayer 的 speed)。

同样的，一个 layer 的时间空间也会影响一个 dynamic layer media 的播放，如 Quartz Composer compositions。将 `QCCompositionLayer` 的速度加倍会使得这个 composition 以两倍的速度播放，跟将依附在 layer 上的动画加倍是一样的。再次说明，这种效果是层叠的，所以 `QCCompositionLayers` 的任何 sublayers 也会使用增加的速度来显示它们的内容。

`CAMediaTiming` 的 `duration` 属性被动画用来定义动画单次迭代显示所花时长。`CAAnimation` 子类所花默认时长是 0s，这意味着动画应该使用它所在 transaction 的时长，或 .25s 如果没有 transaction 被指定。

timing protocol 使用 `beginTime` 和 `timeOffset` 属性提供了延迟执行动画的途径。`beginTime` 指定了动画应该开始的时间点，这是相对于动画 layer 的时间空间而言的。`timeOffset` 指定的是额外的 offset，但是以本地时间陈述的。这两个值结合起来决定最终的启动 offset。

### Repeating Animations
一个动画的重复行为由 `CAMediaTiming` 协议决定，通过 `repeatCount` 和 `repeatDuration` 属性。`repeatCount` 指定了动画应该重复的次数，可以是个小数。设置一个 10s 的动画的 `repeatCount` 为 2.5 会导致这个动画总共跑 25s，在第三次的迭代中间结束。设置 `repeatCount` 为 1e100f 会导致动画重复到它从 layer 中移除。

`repeatDuration` 和 `repeatCount` 相似，尽管它是通过秒指定的而不是迭代次数。`repeatDuration` 也可以是一个小数值。

动画的 `autoreverses` 属性决定了一个动画在它完成播放之后是否倒播；假设多个重复被指定。

### Fill Mode
timing protocol 的 `fillMode` 属性定义了一个动画在结束时刻将会怎么显示。动画可以冻结在它的开始位置，它的结束位置，或同时，或从显示上完全移除。默认行为是当它结束时完全从 layer 上移除。

## Animation Pacing
一个动画的节奏决定了插入的值是怎么在动画的时长中分布的。对于一个特定的视觉效果使用合适的节奏可以很大的加强它对用用户的效果。

一个动画的节奏使用一个计时函数来表达的，而计时函数又是通过一个 cubic Bezier 曲线表示。这个函数将动画的单个周期时长 (通常是 [0.0, 1.0] 的范围) 映射到输出时间。

`CAAnimation` 的 `timingFunction` 属性指定一个 `CAMediaTimingFunction` 的实例，这个类负责封装计时功能。

`CAMediaTimingFunction` 提供了两个选择来指定映射函数：通用的 pacing curves 常量，和通过指定两个 control points 创建自定义函数的方法。

预定义的计时函数通过制定下列的常量之一给 `CAMediaTimingFunction` 的类方法 `functionWithName:` :

* `kCAMediaTimingFunctionLinear` 指定一个线性节奏。一个线性节奏使得一个动画在它的时长内均匀的发生。
* `kCAMediaTimingFunctionEaseIn` 指定一个 ease-in 节奏。ease-in 节奏使得一个动画很慢的开始，然后加速它的过程。
* `kCAMediaTimingFunctionEaseOut` 指定一个 ease-out 节奏。ease-out 节奏使得一个动画很快的开始，然后减速它的过程。
* `kCAMediaTimingFunctionEaseInEaseOut` 指定一个 ease-in ease-out 的节奏。一个 ease-in ease-out 的动画开始很慢，加速通过中间点，然后开始减速，直到结束。

下表通过各自的 cubic Bezier timing 曲线显示了预定义的计时函数。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/AnimationTypesAndTiming/cubic_bezier_curve_representations_of_the_predefined_timing_functions.png" width=80% />
 
 自定义的计时函数通过 `functionWithControlPoints::::` 类方法或 `initWithControlPoints::::` 实例方法来创建。Bezier 曲线的两端自动设置为 (0.0, 0.0) 和 (1.0, 1.0)，并且创建函数期待 c1x, c1y, c2x, c2y 等坐标参数。最终定义 bezier 曲线的 control points 是：[(0.0,0.0), (c1x,c1y), (c2x,c2y), (1.0,1.0)]。
 
 下面的样例代码片段使用 [(0.0,0.0), (0.25,0.1), (0.25,0.1), (1.0,1.0)] 创建了一个自定义计时函数。
 
 ```objc
 CAMediaTimingFunction *customTimingFunction;
 customTimingFunction=[CAMediaTimingFunction functionWithControlPoints:0.25f :0.1f :0.25f :1.0f];
 ```
 
 > **注意**：关键帧动画需要一个更细致入微的 pacing 和 timing model 而不是一个简单的 `CAMediaTimingFunction` 实例就可以满足的。更多信息可以查看本文的 *Keyframe Timing and Pacing Extensions*
 
## Animation Delegates
`CAAnimation` 类提供了途径来通知一个 delegate 对象一个动画什么时候开始和结束。

如果一个动画有一个指定的 delegate，它将收到 `animationDidStart:` 消息，并传入开始的动画对象。当一个动画结束的时候，delegate 收到 `animationDidStop:finished:` 消息，传入一个停止了的动画对象和一个代表动画是成功完成它的时长还是被手动停止的布尔值。

> **重要**: `CAAnimation` delegate 对象被消息接受者所 retain。这是 *Advanced Memory Management Programming Guide* 中描述的内存管理规则中少有的例外情况。

-----
# Property-Based Animations
基于属性的动画是插入值到一个 layer 的一个简单属性的动画，例如，一个 layer 的位置，背景色，边框等。

这章讨论了 Core Animation 是怎样抽象属性动画和它提供来支持 layer 属性的基本和关键帧动画的类。

## Property-Based Abstraction
`CAPropertyAnimation` 类是 `CAAnimation` 的抽象子类，提供了基本了功能来动画显示一个 layer 的一个特定属性。Property-based 动画支持所有可以通过数学计算插入的值类型，包括以下：

* 整型和浮点型 (double)
* `CGRect`, `CGPoint`, `CGSize`, and `CGAffineTransform` 结构体
* `CATransform3D` 数据结构
* `CGColor` 和 `CGImage` 引用

和所有的动画一样，一个属性动画必须跟一个 layer 关联上。被做动画的属性通过一个相对于 layer 的 KVC key path 指定。例如，要对 "layerA" 的 `position` 属性的 x 部分做动画的话，你使用 "position.x" 来创建一个动画，并添加这个动画到 "layerA"。

你绝不需要直接实例化一个 `CAPropertyAnimation`。相反你会创建它的子类的实例，`CABasicAnimation` 或 `CAKeyframeAnimation`。同样，你也绝不需要继承 `CAPropertyAnimation`，相反你会继承 `CABasicAnimation` 或 `CAKeyframeAnimation` 来添加功能和存储额外的属性。

## Basic Animations
`CABasicAnimation` 类为 layer 的属性提供了基本的单帧动画能力。你使用继承的方法 `animationWithKeyPath:` 创建一个 `CABasicAnimation` 的实例，指定需要做动画的 layer 属性。*Core Animation Programming Guide* 中的 **Animatable Properties** 总结了 `CALayer` 和它的 filter 可做动画的属性。

### Configuring a Basic Animation’s Interpolation Values
`CABasicAnimation` 的 `fromValue`, `byValue` 和 `toValue` 属性定义了插值的范围。所有的都是可选的，非空的值不能超过两个。设置的值的类型和被做动画属性类型匹配。

插值按照以下规则使用：

* `fromValue` 和 `toValue` 非空。插值在 `fromValue` 和 `toValue` 之间。
* `fromValue` 和 `byValue` 非空。插值在 `fromValue` 和 `fromValue + byValue` 之间。
* `byValue` 和 `toValue` 非空。插值在 `toValue - byValue` 和 `toValue` 之间。
* `fromValue` 非空。插值范围在 `fromValue` 和当前的这个属性的 presentation 值之间。
* `byValue` 非空。插值范围在这个 `keyPath` 指定的属性 presentation 值和这个值加上 `byValue` 之间。
* 所有的属性值都为空。插值范围目标 layer 的 presentation layer 的 keypath 所指定的属性的之前值和当前值之间。

> **注意**: 在 OS X 上传给这些属性的是 `NSPoint` 结构类型。然而在 iOS 上，类型是 `CGPoints`，除了这个小的区别，它们的使用是相同的。

### An Example Basic Animation
下面的代码展示了一个实现显示动画的代码片段：

```objc
CABasicAnimation *theAnimation;
// create the animation object, specifying the position property as the key path
// the key path is relative to the target animation object (in this case a CALayer) 
theAnimation = [CABasicAnimation animationWithKeyPath:@"position"];
// set the fromValue and toValue to the appropriate pointstheAnimation.fromValue=[NSValue valueWithPoint:NSMakePoint(74.0,74.0)];theAnimation.toValue=[NSValue valueWithPoint:NSMakePoint(566.0,406.0)];
// set the duration to 3.0 secondstheAnimation.duration=3.0;
// set a custom timing functiontheAnimation.timingFunction=[CAMediaTimingFunction functionWithControlPoints:0.25f :0.1f :0.25f :1.0f];
```

> **注意**: 这个例子是在 OS X 中的。当在 iOS 中编译的时候需要一些小的修改。`NSMakePoint` 函数不可用，相反应该使用 `CGPointMake` 函数。除了这个直接替换外，其他都一样。

## Keyframe Animations
关键帧动画是通过 Core Animation 中的 `CAKeyframeAnimation` 类支持的，它跟基本的动画相似；不过它允许你指定一组目标值。在动画的过程中，这些值一次被插入。

### Providing Keyframe Values
关键帧的值可以使用两种方式指定：一个 Core Graphics path 或一组对象值。

一个 Core Graphics path 很适合给 layer 的 `anchorPoint` 或 `position` 做动画，这也是说，类型为 `CGPoints` 的属性。path 中除了 `moveto` 点外的每个点为计时和插值定义了一个简单的关键帧。如果 `path` 有指定，那么 `values` 被忽略。

默认情况下，当 layer 随着一个 `CGPath` 做动画它会保持它被设置的 rotation。设置 layer 的 `rotationMode` 属性为 `kCAAnimationRotateAuto` 或 `kCAAnimationRotateAutoReverse` 会导致 layer 按照 path 的射线 rotate。

提供一组对象给 `values` 属性使得你可以对 layer 的任意类型属性做动画。例如：

* 提供一组 `CGImage` 对象，并设置动画的 key path 为 layer 的 `content` 属性。这会使得 layer 的内容动画经过所提供的图片。
* 提供一组 `CGRects` 并设置动画的 key path 为 `frame` 属性。这会使得 layer 的 frame 依次经过提供的矩形框。
* 提供一组 `CATransform3D` 值并设置动画的 key path 为 `transform` 属性。这使得每个 transform matrix 应用到 layer 的 transform 属性上。

### Keyframe Timing and Pacing Extensions
关键帧动画比一个基本动画要求一个更复杂的计时和 pacing 模型。继承的 `timingFunction` 属性被忽略。相反你赋一个可选的一组 `CAMediaTimingFunction` 实例给 `timingFunctions` 属性。每个 timing 函数描述了一个关键帧段的 pacing。

但是对于 `CAKeyframeAnimation` 继承的 `duration` 属性是有效的，你可以使用 `keyTimes` 属性获取更精细的控制。

`keyTimes` 属性指定一组 `NSNumber` 对象，其中的每个对象定义了帧动画每段的时长。数组中的每个值介于 0.0 到 1.0 之间，并对应到 `values` 中的每一个值。`keyTimes` 中的以动画总时长的一部分来定义相应的段的时长。每个元素值必须大于或等于前面的值。

`keyTimes` 合适的值是依赖于 `calculationMode` 属性的。

* 如果 `calculationMode` 被设置为 `kCAAnimationLinear`，那么数组中的第一个值必须是 0.0，最后的值必须是 1.0。其它值被插入指定的关键点之间。
* 如果 `calculationMode` 被设置为 `kCAAnimationDiscrete`，数组中的第一个值必须是 0.0
* 如果 `calculationMode` 被设置为 `kCAAnimationPaced`， keyTimes 被忽略。

### An Example Keyframe Animation

```objc
// create a CGPath that implements two arcs (a bounce)CGMutablePathRef thePath = CGPathCreateMutable();CGPathMoveToPoint(thePath,NULL,74.0,74.0);CGPathAddCurveToPoint(thePath,NULL,74.0,500.0,                                   320.0,500.0,                                   320.0,74.0);CGPathAddCurveToPoint(thePath,NULL,320.0,500.0,                                   566.0,500.0,                                   566.0,74.0);
                                   
CAKeyframeAnimation * theAnimation;
// create the animation object, specifying the position property as the key path 
// the key path is relative to the target animation object (in this case a CALayer) theAnimation=[CAKeyframeAnimation animationWithKeyPath:@"position"]; 
theAnimation.path=thePath;
// set the duration to 5.0 secondstheAnimation.duration=5.0;
// release the pathCFRelease(thePath);
```

----
# Transition Animation
Transition animation is used when it is impossible to mathematically interpolate the effect of changing the value of a layer property, or the state of a layer in the layer tree.

这章讨论了 Core Animation 提供的转场动画功能。

## Creating a Transition Animation
`CATransition` 提供了转场的功能。它是 `CAAnimation` 的直接子类，因为它整个 layer ，而不是一个 layer 的某个指定属性。

一个 `CATransition` 实例是通过使用继承的类方法 `animation` 创建。这会使用下表中的默认值创建一个 transition 动画。

Transition Property | Value
type | 使用一个 fade transition 。值为 `kCATransitionFade`
subType | 不可用
duration | 使用当前的 transaction 或 0.25s 如果 transaction 的 duration 还未设置的话。
timingFunction | 使用 linear pacing。这个值为 nil。
startProgress | 0.0
endProgress | 1.0

## Configuring a Transition
一个 transition 一旦创建，你可以使用预定义的 transition type 来配置它，或者在 OS X 上，使用一个 Core Image filter 创建自定义的 transition。

使用来设置 `type` 属性的预定义 transition 如下：

Transition Type | Description
---------------------- | ----------------
`kCATransitionFade` | The layer fades as it becomes visible or hidden.
`kCATransitionMoveIn` | The layer slides into place over any existing content.
`kCATransitionPush` | The layer pushes any existing content as it slides into place
`kCATransitionReveal` | The layer is gradually revealed in the direction specified by the transition subtype.

除了 `kCATransitionFade` 外，一个预定义的 transition 类型允许你为 transition 指定一个方向，通过设置 `subType` 为下表中的一个常量。

Transition Subtype constant | Description
--------------------------------------- | ---------------
`kCATransitionFromRight` | transition 从 layer 的右边开始
`kCATransitionFromLeft` | transition 从 layer 的左边开始
`kCATransitionFromTop` | transition 从 layer 的顶部开始
`kCATransitionFromBottom` | transition 从 layer 底部开始

`startProgress` 属性允许你改变一个 transition 的起始点，通过设置一个值来代表整个动画的一部分。例如，要从一个 transition 的中间开始可以设置 `startProgress` 值为 0.5。相似的，你也可以指定 `endProgress` 值。`endProgress` 是整个 transition 应该停止的点。这两个属性的默认值分别是 0.0 和 1.0。

如果一个预定义的 transition 没有你的应用想用的视觉效果，并且你的应用是面向 OS X 而不是 iOS，你可以指定一个自定义 Core Image filter 对象来展示这个 transition。一个自定义 filter 必须同时支持 `kCIInputImageKey` 和 `kCIInputTargetImageKey` input keys，和 `kCIOutputImageKey` output key。filter 可以选择性的支持 `kCIInputExtentKey` input key，这个 key 被设置为一个 transition 应该发生的矩形框。