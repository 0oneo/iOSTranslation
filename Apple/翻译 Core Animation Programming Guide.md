# 关于 Core Animation
Core Animation 是一个在 OS X 和 iOS 中同时可用的图形渲染和动画基础设施，你可以用来做视图或其它可视元素的动画。Core Animation 已经帮你做了动画的每一帧所需的大部分工作。你所需要做的是配置一些动画参数 (如开始和结束的位置) 并告诉 Core Animation 开始。Core Animation 处理剩下的工作，将大部分真实的绘制工作交付给现有的图形应用来加速渲染。这个自动的图形加速保证了高帧率和平滑的动画，不需要 CPU 有过多的负担，从而减缓应用。

如果你正在编写一个 iOS 应用，你已经在使用 Core Animation，不管你知道与否。如果你在编写 OS X 应用的话，你基本不费精力就可以使用 Core Animation。 Core Animation 坐落在 AppKit 和 UIKit 之下并高度的集成到 Cocoa 和 Cocoa Touch 的工作流之中。当然，Core Animation 也有扩展你的应用展现的能力的接口，并给了你对动画更精细的控制。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/core-animation-framework-overview.png" width=50% />

## 窥看
你也许永远都不需要直接使用 Core Animation，但是当你需要的时候，你应该理解 Core Animation 是应用基础设施的一部分。

### Core Animation 管理应用的内容
Core Animation 自身不是一个绘制系统。它是一个用于组合 (compositing) 和用硬件操作你应用内容的基础设施。在这个基础设施的核心是 *layer objects*，它们被用来管理和操作应用内容。一个 layer 捕捉你的内容到一个位图 (bitmap)，这个位图可以轻易地被图形硬件操作。大部分应用中，layers 被用来作为管理视图内容的一种方式，但是你可以根据你的需要单独的使用它。

### Layer 修改引发动画
大部分你使用 Core Animation 创建的动画涉及到修改 layer 的属性 (properties)。跟视图 (view) 相像，layer 对象有一个 bounds rectangle，在屏幕上的 position，一个 opacity，一个 transform，和许多其它视觉相关的属性，这些属性可被修改。对于大部分属性，修改这些属性涉及到创建一个隐式动画——layer 从老的值动画到新的值。在那些你需要在动画行为中有更多控制的场景，你也可以显式对这些属性做动画。

### Layers 可以被组织成 Hierarchies
Layers 可以被组织成层次的，从而创建了 parent-child 的关系。这种安排 layers 的方式影响了可视内容，这种安排方式跟视图的安排方式类似。附着在视图上的一系列 layers 的层次与相应的视图层次相似。你也可以添加单单的 layers 到一个 layer 层次上来扩展你应用的可以视觉内容，而不是仅仅是你的视图。

### Actions Let You Change a Layer’s Default Behavior
隐式 layer 动画是使用 *action objects* 达成的，*action objects* 是实现了预先定义接口的一般对象。Core Animation 使用 action objects 来实现通常与 layers 关联默认动画系列。你可以创建你自己的 action 对象来实现自定义动画或使用它们来其他类型的行为。你然后将你的 action 对象赋给 layer 的一个属性。当那个属性发生改变的时候，Core Animation 取出你的 action 对象，告诉它进行它的行为。

## 怎样使用这篇文档
这篇文档是为那些需要对应用的动画有更多控制或想要使用 layers 来提升应用绘制性能的开发者。这篇文档同样提供了 OS X 和 iOS 中关于 layers 和视图集成的信息。layers 和视图的集成在 iOS 和 OS X 中是不同的，明白这些不同对于创建高效的动画很重要。

## 前提
你应该已经理解你的目标平台的视图架构，并熟悉怎么创建基于视图的动画。如果还没有的话，你需要读以下文档之一：

* 对于 iOS 应用，你应该理解 *View Programming Guide for iOS*  中介绍的视图架构
* 对于 OS X 应用，你应该理解 *View Programming Guide* 中介绍的视图架构

----
# Core Animation 基础
Core Animation 提供了一个用来使应用中的视图和其它可视元素动起来的通用系统。Core Animation 并不是应用视图的替代品。而是一项与视图集成起来为给它们的内容动画提供更好的性能和支持的技术。它通过缓存视图的内容到位图来达到这种行为，这份位图可以直接被图形硬件操作。在一些情况下，这种缓存行为也许会要求你重新思考你怎么展现和管理应用的内容，但是大部分时候你使用 Core Animation 根本不需要知道缓存这事。除了缓存视图的内容，Core Animation 也定义了一种方式指定任意可视内容，与你的视图集成这些内容，和其它的一切做动画。

你使用 Core Animation 来以动画的形式改变你应用的视图和可视对象。大部分改变涉及到修改你可视对象的属性。例如，你也许使用 Core Animation 来动画显示一个视图位置、大小、透明度的变化。当你做出这些改变的时候，Core Animation 在属性的当前值和你指定的新值之间做动画。通常你不会使用每秒 60 次替换视图的内容，而不使用 Core Animation 。相反，你使用 Core Animation 来围绕屏幕移动一个视图的内容，fade in 或 out 视图的内容，应用任何图形变换到视图，或改变视图的可视属性。

## Layers 提供绘制和动画的基础
*Layer objects* 是组织在 3D 空间的 2D 平面，是你用 Core Animation 做的任何事的核心。跟视图一样，layers 管理着面的关于几何、内容和其他可视属性的信息。跟视图不像的是，layers 并不定义它们自己的外表。一个 layer 只管理围绕位图的状态信息。位图自身可以视图绘制自己的结果或一个你指定的固定图片。因为这个原因，你在你的应用中使用的主 layers 被认为是 model 对象，因为它们主要管理数据。这个概念很重要，需要记住，因为它影响着动画的行为。

### 基于 Layer 的 Drawing Model
应用大部分 layer 并做任何绘制。相反，一个 layer 拿到应用提供的内容后缓存到一个位图中，这个位图有时被指作 *backing store*。当你之后改变 layer 的一个属性时，你所做的只是改变与 layer 对象关联的状态信息。

当一个改变引起一个动画时，Core Animation 将 layer 的位图和状态信息传递给图形硬件，图形硬件使用新的信息来渲染位图，如下图所示。在硬件中操作位图要比软件中这么做提供了快很多的动画。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/how-core-animation-draws-content.png" width=90% />

因为它是操作静态的位图，基于 layer 的绘制跟传统的基于视图的绘制技术有很大的区别。对于基于视图的绘制技术，改变视图自身的往往会导致调用视图的 `drawRect:` 方法使用新的参数来重新绘制内容。但因为这种绘制工作是在主线程上使用 CPU 完成的，所以性能消耗很昂贵。Core Animation 通过在硬件中操作缓存的位图来达到相同或相似的想过来尽可能的避免这样的消耗。

尽管 Core Animation 尽可能多的使用缓存的内容，你的应用必须提供初始的内容，并时不时的更新它。你的应用有好几种方式来提供 layer 对象的内容。

### 基于Layer 的 Animations
一个 layer 的数据和状态信息和一个 layer 对象的内容在屏幕上视觉展现是解耦的。这种解耦给了 Core Animation 一个方式来使自己介于和动画展现从老的状态到新状态的改变。例如，改变一个 layer 的位置引起 Core Animation 把 layer 从它当前的位置移动到新指定的位置。相似的对其它属性的改变引起相应的动画。下图图示了你可以在 layers 上进行的一些动画类型。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/animations-type-core-animation-support.png" width=90% />

在这些动画的过程中，Core Animation 帮你在硬件中做了所有帧的绘制。你需要做的只是指定动画开始和结束的点，然后让 Core Animation 做剩下的。你也可以根据需要指定自定义的时间函数和动画参数；然而 Core Animation 提供合适的默认值如果你不提供的话。

## Layer 对象定义它们自己的 Geometry
Layer 的工作之一是管理它内容的视觉几何 (visual geometry)。视觉几何由关于内容的界限 (bounds)，屏幕上的位置，layer 是否被 rotated，scaled，或 tranformed 等信息组成。跟视图相像，一个 layer 有 frame 和 bounds rectangles，你可以使用它们来定位 layer 和它的内容。Layer 也有视图没有的属性，如一个 anchor point，anchor point 定义了操作围绕着哪一点发生。你指定 layer 几何信息的某些面也和你指定视图的信息不一样。

### Layers 使用两种不同类型的 Coordinate Systems
Layers 同时使用 *point-based coordinate systems* 和 *unit coordinate systems* 来指定内容的位置。具体使用那种坐标系统依赖于被传递的信息类型。Point-based coordinates 用来指定直接映射到屏幕坐标的值，或必须相对于另一个 layer 而言的值，如 layer 的 `position` 属性。Unit coordinates 被用在与屏幕无关的值，因为它与其它的值相关。例如，一个 layer 的 `anchorPoint` 属性指定一个相对于 layer 自身 bounds 的坐标，它的值可以改变。

Point-based coordinates 的常用场景是指定 layer 的大小和位置，你使用 layer 的 `bounds` 和 `position` 来指定。`bounds` 定义了 layer 自身的坐标系统，包含了它在屏幕上的大小。`position` 属性定义 layer 相对于 parent layer 的坐标系统的位置。尽管 layer 有一个 `frame` 属性，这个属性的值实际上是继承自 `bounds` 和 `position` 属性，并很少被用到。

layer 的 `bounds` 和 `frame` 矩形框的方向总是跟潜在系统的默认方向一致。下图显示了 iOS 和 OS X bounds 矩形框的默认方向。iOS 中 bounds 的 origin 默认在 layer 的左上角，OS X 中它在左下角。如果你在 iOS 和 OS X 中共用 Core Animation 的代码的话，你需要考虑这种不同。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/default-layer-gemoetries.png" width=80% />

上图中需要注意的一点是 `position` 属性是定位在 layer 的中间。这个属性几个定义会随着 layer 的 `anchorPoint` 而变的属性之一。这个 anchor point 代表了一些坐标起始的点。

anchor point 是几个你使用 unit coordinate system 来指定坐标的属性之一。Core Animation 使用 unit coordinate 来表示那些属性的值会随着 layer 的大小变化而改变的属性。你可以认为 unit coordinates 是指定一个所有可能值范围的一个百分比。在 unit coordinate system 中的每一个坐标有一个 0.0 到 1.0 范围。例如，沿着 x 轴，左边的边在坐标 0.0 处，右边的边在坐标 1.0 处。沿着 y 轴，unit coordinate 的值依赖于平台。如下图所示。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/unit-coordinate-systesm-for-ios-and-osx.png" width=80% />

> **注意**: 直到 OS X 10.8，`geometryFlipped` 属性需要改变 y 轴默认方向的一种方式。当涉及到翻转 (flip) 变换时使用这个属性来纠正 layer 的方法很有必要。例如，如果一个父视图使用了翻转变换，子视图的内容将是被反转。这种情况下，设置子 layer 的 `geometryFlipped` 属性成 `YES` 是纠正这个问题的简单方式。自 OS X 10.8 来，AppKit 替你管理这个属性，你不应该修改它。对于 iOS 应用，推荐你完全不用这个属性。

所有的坐标值，不管它们是点或单元坐标，都是用浮点数来表示的。浮点数的使用可能使得值落到正常坐标值的之间。使用浮点数是方便的饿，尤其是在打印或在一个点对应好几个像素的视网膜屏上绘制时。浮点数使得你可以忽略潜在设备的分辨率，只需按你需要的精度指定值。

### Anchor Points 影响 Geometric Manipulations
Layer 的几何相关的计算是相对于 layer 的 anchor point 的，你可以通过 layer 的 `anchorPoint` 属性来访问这个值。anchor point 的影响在操作 layer 的 `position` 和 `transform` 属性时最直观。`position` 属性总是相对于 layer 的 anchor point 而言的，而任何你应用到 layer 的转换 (transformation) 也总是相对于 anchor point 而言的。

下图展示了 anchor point 从默认位置改变到不同的值时是怎样影响 layer 的 `position` 属性的。即使 layer 在父 layer 的 bounds 中没有移动，将 anchor point 从 layer 的中心移到 layer bounds 的 origin 改变了它的 `position` 属性。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/how-anchor-point-affect-layer-position-property.png" width=80% />

下图展示了改变 anchor point 怎么影响应用到 layer 的 transform。当你应用一个旋转的 transform 到 layer 上时，旋转是围绕着 anchor point 的。因为 anchor poin 默认设置为 layer 的中间，这通常正是你所期望的旋转行为。然而，如果你改变 anchor point 的话，旋转的结果会不同。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/how-anchor-point-affect-layer-position-property.png" width=80% />

### Layers 可以在三维空间被操作
每个 layer 有两个变换矩阵，你可以使用来操作 layer 和它的内容。Layer 的 `transform` 属性用来指定你想应用到 layer 和它的子 layers 上的变换。通常当你想修改 layer 自身时使用这个属性。例如，你也许使用这个属性来 scale 或 rotate layer 或暂时改变它的位置。`sublayerTransform` 属性定义了额外的 transformations，这个属性只应用于 sublayers，最常用于给一个场景的内容添加透视效果。

Transforms 通过将坐标乘以一个矩阵得到一个新的坐标来工作，新的坐标代表了原来点被转换后的版本。因为 Core Animation 值是在三维空间的，每个坐标点有 4 个值，必须乘以一个 4x4 的矩阵，如下图所示。在 Core Animation 中，图中的 transform 的类型是 `CATransform3D`。幸运的是在进行标准的 transform 时你不需要直接修改这个结构的字段值。Core Animation 提供了广泛的函数来创建 scale、translation、rotation 矩阵。除了使用这些函数来操作 transform 外，Core Animation 扩展了 KVC 支持来允许你使用 key path 来修改一个 transform。具体的 key path 参见 *CATransform3D Key Paths*。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/converting-a-coordinate-using-matrix-math.png" width=80%>

下图展示了一些你可以创建的通用 transformations 的矩阵配置。任何坐标乘以 identity transform 返回同样的坐标。对于其他的 transformations，坐标怎么被修改完全依赖于你改变了矩阵的哪些部分。

例如，只沿着 x 轴移动的话，你只需要提供一个非 0 的值给 translation 矩阵的 tx 部分，保持 ty 和 tz 部分为 0。对于旋转，你需要提供目标旋转角度的 sine 和 cosine 值。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/matrix-configurations-for-common-transformations.png" width=80% />

## Layer Trees Reflect Different Aspects  of the Animation State
一个使用 Core Animation 的应用有三个 layer 对象的集合，每个 layer 对象的集合再组成应用屏幕上内容中担当不同的角色：

* *model layer tree* (简称 "layer tree") 中的对象是那些应用交互最多的。这个树中的对象是存储了任何动画目标值的 model 对象。无论什么时候你改变一个 layer 的属性，你使用这些对象中的一个。
* *presentation tree* 中的对象包含了任何正在进行的动画的当前值。也就是说 layer tree 对象包含了动画的重点值，presentation tree 中的对象反映了屏幕上显示的当前值。你绝不应该这个树中对象的值。相反，你使用这些对象来读取当前动画的值，也许创建一个从这些值开始的新动画。
* *render tree* 中的对象进行了真正的动画，它们是 Core Animation 私有的。

每个集合的 layer 对象都是以层状结构组织的，跟应用中的视图很像。实际上，对于一个所有视图都使用 layer 的应用，每个 tree 的初始结构和视图的层次结构一模一样。然而应用可以根据需要添加额外的 layer 对象 —— 也就是，与 view 不相关的 layer —— 到 layer 的层次结构中。对于那些不要视图的负载的内容，你也许这么做是为了优化应用的性能。下图分解了在一个简单的 iOS 应用中可以发现的 layers。这个例子中 window 有一个 content view，content view 包含了一个 button view 和两个单独的 layer 对象。每个 view 有一个相应的 layer 对象，这些 layer 对象组成了 layer 的层次结构。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layers-associated-with-a-window.png" width=80% />

对于 layer tree 中的每一个对象，在 presentation 和 render tree 中都一个对应的对象，如下图所示。像前文提及的一样，应用主要使用 layer tree 中的对象工作，但有时会访问 presentation tree 中的对象。具体的说，访问 layer tree 中一个对象的 `presentationLayer` 属性返回相应的在 presentation tree 中的对象。你也许想要访问这个对象来获取动画过程中一个属性的当前值。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-tree-for-a-window.png" width=80% />

> **重要**: 你应该在动画的过程中访问 presentation tree 中的对象。当一个动画在过程中时，presentation tree 包含了屏幕当前位置 layer 的属性。这个行为是有别于 layer tree 的。

## Layers 和 Views 间的关系
Layer 并不是应用视图的替换，也就是说，你不同只通过 layer 来创建应用的视觉接口。layer 为视图提供了基础设施。具体的说，layer 使得绘制和队视图内容做动画变得更简单和高效，并在做这些的时候保证了高帧率。然而，layer 有很多事是不做的。Layer 不处理事件，绘制内容，参与到响应链等等。因为这个原因，每个应用仍然有一个或多个视图来处理这类交互。

在 iOS 中，每个视图都有相应的 layer 对象，但 OS X 你必须决定哪些视图有 layer。自 OS X v10.8 开始，给你所有的视图添加 layer 更合理。然而，不要求你一定这么做，在负载无法保证或不需要的时候，你仍然可以关闭 layer。Layers 确实增加了应用内存的消耗，但是它们带来的好处往往更多，所以在关闭 layer 支持前测试应用的性能是最好的。

当你打开 view 的 layer 支持，你创建的 view 被称为 *layer-backed view*。在一个 layer-backed view 中，系统负责创建潜在的 layer 对象和保持 layer 与 view 同步。所有的 iOS view 都是 layer-backed，大部分 OS X view 也是一样。然而，在 OS X 中，你也可以创建一个 *layer-hosting view*，这种 view 由你来提供 layer 对象。对于一个 layer-hosting view，AppKit 采取了 hands off 的方式来管理 layer，不改变 layer 来响应 view 改变。

> **注意**: 对于 layer-backed views，可能的话建议你操作 view，而不是它的 layer。在 iOS 中，views 只是 layer 对象的一个简单包装，所以任何对 layer 的操作通常可以正常工作。但在 iOS 和 OS X 中有些情况下，操作 layer 而不是 view 可能引起不想要的结果。无论在哪儿，可能的话这篇文档总是会之处这些陷阱，并提供方法来帮你解决它们。

除了与 view 相关联的 layer 外，你也可以创建跟 view 没有关系的 layer 对象。你可以将这些单独的 layer 对象嵌入应用中任何其它 layer。通常你使用单独的 layer 来作为特定优化的一部分。例如，如果你想在多个地方使用相同的图片，你可以加载图片一次，然后把它关联到多个单独的 layer 对象上，然后把这些 layer 添加到 layer tree 上。这样每个 layer 指向相同 image source，而不是创建自己的 copy。

----
# 创建 Layer Objects
Layer 对象你使用 Core Animation 做的所有事情的核心。Layers 管理应用的可视内容，提供选项修改内容的样式和视觉外表。尽管 iOS 应用自动打开了 layer 支持，但OS X 的开发者在他们获得性能提升前必须显式打开。一旦启用，你需要理解怎么配置，和操作应用的 layer 来获得你想要的效果。

## 打开应用中的 Core Animation 支持
在 iOS 应用中，Core Animation 总是启用的，每一个 view 都有一个 layer 支持。OS X 中，应用必须显式通过以下方式启用 Core Animation 支持：

* 链接 `QuartzCore` framework. (iOS 只在显式使用 Core Animation 的接口时需要链接这个框架)
* 通过以下方式启用一个或多个 `NSView` 对象的 layer 支持：
	* 在你的 nib 文件中，使用 View Effects Inspector 来启用 layer support。这个 inspector 为选中的 view 和它的 subviews 显式了勾选框。无论在什么时候，可能的话建议为你的 content view 启用 layer support
	* 对于代码创建的 view，调用 view 的 `setWantsLayer:` 方法，传一个 `YES` 来表示 view 应该使用 layer

使用上述方法之一启用 layer support 总是创建一个 layer-backed view。使用 layer-backed view 的话，系统总是负责创建底层的 layer 对象和保持 layer 始终最新。在 OS X 中，同样可以创建 layer-hosting view，就是你的应用自己创建管理底层的 layer 对象。(iOS 中你创建不了 layer-hosting view) 

## 改变与 View 关联的 Layer 对象
Layer-backed view 默认创建一个 `CALayer` 类的实例，并且大部分时候你不需要不同类型的 layer 对象。然而，Core Animation 提供了不同的 layer 类，每一个都提供了特别的功能，你也许会发现很有用。选择一个不同的 layer 类也许能帮你提升性能，或更简单的支持特定类型的内容。例如，`CATiledLayer` 类被优化来以高效的方式展示大的图片。

### 改变 UIView 使用的 Layer 类
你可以通过重写 view 的 `layerClass` 方法返回一个不同的类对象来改改变 iOS view 使用的 layer 类型。大部分 iOS view 创建一个 `CALayer` 对象并使用这个对象作为它内容的 backing store。对于大部分你自己的视图而言，默认的选择是一个不错的选择，你应该不需要改变它。但是你也许发现在某些场合一个不同的 layer 类更适合。例如，在以下场合你也许想要改变 layer class：

* 你的视图使用 `Metal` 或 `OpenGL ES` 来绘制内容，在这种情况你将会使用一个 `CAMetalLayer` 或 `CAEAGLLayer` 对象
* 有一个特定的 layer class 提供了更好的性能
* 你想要利用一些特定的 Core Animation layer class，如 particle emitters 或 replicators

改变一个视图的 layer class 很直观；实例代码如下。你所需做的只是覆盖 `layerClass` 方法并返回你想要使用的 class 对象。在显示之前，view 会调用 `layerClass` 方法并使用返回的 class 创建一个新的 layer object。一旦创建，一个视图的 layer object 不可以改变。

```objc
+ (Class) layerClass {
    return [CAMetalLayer class];
}
```

### 改变 NSView 使用的 Layer Class
你可以通过覆盖 `makeBackingLayer` 方法来改变 `NSView` 使用的 layer class。在这个方法中，你的实现创建并返回一个 layer 对象，AppKit 使用这个对象来支持你的视图。你也许会在你想要使用一个自定义 layer，如一个 scrolling 或 tiled layer 时覆盖这个方法。

### 在 OS X 中 Layer Hosting 让你改变 Layer Object
一个 layer-hosting view 是一个 `NSView` 对象，这个对象你来创建和管理底层的 layer 对象。也许会在你想控制与视图关联的 layer 对象的类型时候，你会使用 layer hosting。例如，也许你创建一个 layer-hosting view 以便你可以指定一个 layer class 而不是默认的 CALayer 类。在你想要使用一个单一视图管理一个层状的独立 layer 对象时使用 layer hosting。

但你调用 `setLayer:` 方法并提供一个 layer 对象时，AppKit  对于这个 layer 采取 hands-off 的方式。通常 AppKit 会更新一个视图的 layer 对象，但在 layer-hosting 的情况下，对于大部分属性它不会这么做。

要创建一个 layer-hosting view，创建你的 layer 对象并在显示视图到屏幕前把它关联到视图上，如下面的代码。除了设置 layer 对象外，你仍然必须调用 `setWantsLayer:` 方法让视图知道它应该使用 layers。

```objc
// Create myView...

[myView setWantsLayer:YES];
CATiledLayer* hostedLayer = [CATiledLayer layer];
[myView setLayer:hostedLayer];

// Add myView to a view hierarchy.
```

如果你选择自己 host layers，你必须自己设置 `contentsScale` 属性，并在合适的时候提供高分辨率的内容。

### 不同的 Layer Classes 提供特定的行为
Core Animation 定义了许多标准的 layer 类，它们中的每一个都是为特定的场景所设计。CALayer 类所有 layer 对象的根类。它定义了所有 layer 对象必须支持的行为，并是 layer-backed view 的 layer 的默认类型。然而，你也可以使用下表中的 layer class

Class | Usage
-------- | --------
`CAEmitterLayer` | 使用来实现基于 Core Animation 的 particle emitter 系统。emitter layer 对象控制了 particles 的产生和它们的起点。
`CAGradientLayer` | 用来绘制一个填充 layer 对象的颜色渐变。
`CAMetalLayer` | 用来创建和提供 drawable textures，以使用 Metal 来渲染 layer 内容。
`CAEAGLLayer/CAOpenGLLayer` | 用来创建 backing store 和 context，以使用 OpenGL ES (iOS) 或 OpenGL (OS X) 渲染 layer 内容。
`CAReplicatorLayer` | 当你想要自动拷贝一个或多个 sublayers 时使用。replicator 会为你创建拷贝，并用你指定的属性来改变拷贝的外表或属性。
`CAScrollLayer` | 用来管理由多个 sublayers 组成的大的可滑动的区域。
`CAShapeLayer` | 用来绘制 cubic Bezier spline。Shape layers 对于绘制 path-based shapes 很有优势，因为它们总是导致一个 crip path，相对于你绘制到一个 layer 的backing store 中的 path，在伸缩的时候并不好看。然而 crisp 结果涉及到在主线程上渲染 the shape，并缓存结果。
`CATextLayer` | 用来渲染一个 plain 或 attributed 字符串。
`CATiledLayer` | 用来管理一个大的图片，这个大的图片可以分为很多小块，并支持单独的渲染，支持放大和缩小它的内容。
`CATransformLayer` | 用来渲染一个真 3D layer 层次，而不是其它 layer class 实现的 flattened layer 层次。
`QCCompositionLayer` | 用来村然一个  Quartz Composer composition (仅用于 OS X)

## 提供一个 Layer 的内容
Layers 管理应用提供的内容，是一些 data objects。一个 layer 的内容由包含你想要显示的视觉数据的位图组成。你可以以下面三种方式之一提供位图的内容：

* 直接赋一个 image 对象给 layer 对象的 `contents` 属性 (这个技术最适合于 layer 的内容从不或很少改变)
* 赋一个 delegate 对象给 layer，让 delegate 来绘制 layer 的内容。(这项技术最适用于 layer 内容会间隔的变化，并能由外部对象提供，如一个视图)
* 定义一个 layer 子类，覆盖它的绘制方法之一，自己提供 layer 的内容。(这种技术适用于你必须创建一个自定 layer 或 你想从根本上改变 layer 的绘制行为)

你唯一需要担心给 layer 提供内容的时候是你自己创建 layer 对象。如果你的应用只包含 layer-backed view，那么你不需要担心适用任何前面的技术来提供 layer 的内容。Layer-backed view 尽可能以最高效的方式自动为它关联的 layers 提供内容。

### 使用一个 image 作为 layer 的内容
因为一个 layer 只是一个管理位图的容器，你可以直接赋一个 image 到 layer 的 `contents` 属性。赋一个 image 到 layer 很简单，让你来指定你想在屏幕上显示的图片。layer 会使用你直接提供的 image 对象，并不去尝试创建它自己的拷贝。你的应用在多个地方使用同样的图片时，这种行为可以节省内容。

你赋给 layer 的图片类型必须是 `CGImageRef` 类型的。(自 OS X v10.6 起，你也可以赋一个 `NSImage` 对象) 当赋一个图片的时候，记得提供的图片的分辨率需要与设备的分辨率匹配。对于视网膜屏，你可以需要调整图片的 `contenScale` 属性。

### 使用一个 delegate 来提供 Layer 的内容
如果你的 layer 的内容能动态的变化，你可以使用一个 delegate 对象来提供内容，并在需要的时候更新内容。在显示的时候，layer 调用 delegate 对象提供需要的内容：

* 如果你的 delegate 实现了 `displayLayer:` 方法，这个实现负责创建一个位图，并赋给 layer 的 `contents` 属性
* 如果你的 delegate 实现了 `drawLayer:inContext:` 方法，Core Animation 创建一个位图，创建一个 graphics context 以便绘制到位图中，然后调用 delegate 的这个方法填充位图。delegate 方法需要做的只是绘制到提供的 graphics context。

delegate 对象必须实现上述方法两者之一。如果 delegate 同时都实现的话，layer 只会调用 `displayLayer:` 方法。

当你的应用更喜欢加载或创建它想要显示的位图时重写 `displayLayer:` 方法是最合适的。下面的代码展示了 `displaysLayer:` 的样例实现。在这个例子中，delegate 使用一个 helper 对象来加载和显示它需要的图片。delegate 根据它内部的状态选择要显示的图片，这里它的内部状态由属性 `displayYesImage` 表示。

```objc
- (void)displayLayer:(CALayer *)theLayer {
    // Check the value of some state property
    if (self.displayYesImage) {
        // Display the Yes image
        theLayer.contents = [someHelperObject loadStateYesImage];
    } else {
        // Display the No image
        theLayer.contents = [someHelperObject loadStateNoImage];
    }
}
```

如果你没有提前渲染好的图片或一个 helper 对象来给你创建位图，你的 delegate 可以使用 `drawLayer:inContext:` 方法动态的绘制你的内容。下面的代码给出了 `drawLayer:inContext:` 的样例实现。在这个例子中，delegate 使用定宽和当前的渲染颜色画了一个简单曲线。

```objc
- (void)drawLayer:(CALayer *)theLayer inContext:(CGContextRef)theContext {
    CGMutablePathRef thePath = CGPathCreateMutable();
    
    CGPathMoveToPoint(thePath,NULL,15.0f,15.f);
    CGPathAddCurveToPoint(thePath, NULL, 15.f, 250.0f, 295.0f, 250.0f, 295.0f,15.0f);
    
    CGContextBeginPath(theContext);
    CGContextAddPath(theContext, thePath);
    
    CGContextSetLineWidth(theContext, 5);
    CGContextStrokePath(theContext);
    
    // Release the path
    CFRelease(thePath);
}
```

对于一个有自定义内容的 layer-based views，你应该继续覆盖视图的方法来做你的绘制。一个 layer-based view 自动使它自己成为 layer 的 delegate，并实现所需的 delegate 方法，你不应该去改变这些配置。相反，你应该去实现 view 的 `drawRect:` 方法来绘制你的内容。

自 OS X v10.8 起，绘制的另一个选择是通过覆盖视图的 `wantsUpdateLayer` 和 `updateLayer` 方法。覆盖 `wantsUpdateLayer` 返回 YES 导致 `NSView` 类走另一条渲染路径。view 调用 `updateLayer` 方法，而不是`drawRect:`，`updateLayer` 的实现必须直接赋一个位图给 layer 的 `contents` 属性。这正是 AppKit 期望你直接设置一个 layer 对象的内容。 

### 通过子类来提供 Layer Content
如果你在实现一个自定义 layer，你可以覆盖 layer class 的绘制方法来做任何绘制。对于一个 layer 对象而言自己产生内容并不通用，但是 layer 当然可以管理显示的内容。例如，`CATiledLayer` 类通过分解一个大的图片成可被单独管理和渲染的块，来管理一个张大图片。因为只有这个 layer 知道哪些块需要被渲染，它直接管理绘制行为。

当继承的时候，你可以使用以下技术之一绘制 layer 的内容：

* 覆盖 layer 的 `display` 方法，并使用它直接设置 layer 的 `contents` 属性
* 覆盖 layer 的 `drawInContext:` 方法，使用它绘制内容到提供的 graphics context

覆盖哪一个方法依赖于你对绘制过程需要多少控制。`display` 方法是更新 layer 内容的主要入口，所以覆盖它给了你完全的控制。覆盖 `display` 方法同样意味着你负责创建 `CGImageRef` 对象来赋给 `contents` 属性。如果你只是想要绘制内容 (或让 layer 管理绘制操作)，你可以覆盖 `drawInContext:` 方法，让 layer 给你创建 backing store。

### 微调你提供的内容
当你赋一个图片给 layer 的 `contents` 属性时，layer 的 `contentsGravity` 属性决定了怎么操作图片来适应当前的 bounds。默认，如果一个图片大于或小于当前的 bounds，layer 对象伸缩图片来适应到可用空间内。如果 layer‘s bounds 的长宽比和图片的长宽比不同，这会导致图片变形。你可以使用 `contentsGravity` 属性来保证以最好的方式展现。

你可以赋给 `contentsGravity` 属性的值可以被分为以下两类：

* position-based gravity constants 允许你 pin your image 到特定的边或角，不对图片进行伸缩
* scaling-based gravity constants 允许使用一些选项拉伸图片，其中一些保持了图片的长宽比，一些没有

下图展示了 position-based gravity 设置是怎么影响你的图片。除 `kCAGravityCenter` 常量外，每个常量将图片 pin 到 layer 的 bounds 矩形框特定的边或角。`kCAGravityCenter` 常量居中你的图片。这些常量都不伸缩图片，所以图片总是以它原始大小渲染。如果你图片大于 layer 的 bounds，这回导致图片的一部分被裁减，如果图片更小，不被图片覆盖的部分 layer 背景会显示出来，如果设置了的话。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/position-based-gravity-constants-for-layer.png" width=80% />

下图展示了 scaling-based gravity 常量是怎么影响你的图片的。如果图片不是刚好和 layers 的 bounds 大小一致的话，这些常量会伸缩你的图片。它们的不同是怎样处理原始图片的长宽比。一些保留它，一些不保留。默认 layer 的 `contentsGravity` 属性被设置为 `kCAGravityResize`，这个模式不保留原始图片的长宽比。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/scaling-based-gravity-constants.png" width=80% />

### 与高分辨率的图片工作
Layers 内在并不知道潜在设备的分辨率。一个 layer 只是简单存储位图的指针，然后基于可用的像素尽可能以最好的方式显示它。如果你赋一个图片给 layer 的 `contents` 属性，你必须设置 layer 的 `contentScale` 属性来告诉 Core Animation 图片的分辨率。这个属性的默认值是 1.0，这个值适合于标准分辨率的屏幕上显示的图片。如果你的图片是为视网膜屏准备的话，设置这个属性的值为 2.0。

修改 `contentsScale` 属性只在你直接赋一个位图到你的 layer 上时有必要。一个 layer-backed view 会基于屏幕的分辨率之地哦那个设置 layer 和 view 管理的内容 的 scale factor。例如，如果你在 OS X 中赋一个 `NSImage` 对象给 `contents` 属性，AppKit 会查看是否同时有同一张图片的标准和高分辨率的图片。如果有的话，AppKit 会根据当前的分辨率使用正确的图片，并设置 `contentsScale` 属性来匹配。

In OS X, the position-based gravity constants affect the way image representations are chosen from an NSImage object assigned to the layer. (愚笨，不知咋译) 因为这些变量不会使得图片伸缩，Core Animation 依赖 `contentsScale` 属性来选择最合适的像素密度的图片。

OS X 中，layer 的delegate 可以实现 `layer:shouldInheritContentsScale:fromWindow:` 方法，并使用它来响应 scale factor 的改变。当给定的 window 的分辨率发生改变 (可能是 window 在标准分辨率和高分辨率屏幕间切换) 的时候，AppKit 自动调用这个方法。你的实现可以返回 YES 表示支持改变 layer 的图片的分辨率。这个方法然后应该更新 layer 的内容来反映新的分辨率。

## 调节 layer 的 Visual Style 和 Appearance
Layer 对象有内置的视觉修饰，如边框，用来填充 layer 主内容的背景色。因为这些视觉修饰不需要你这部分的渲染，它们使得某些时候需要使用单独的 layer 实体。你所做的是设置 layer 的一个属性，layer 负责必须的绘制，包括任何动画。

### Layer 有自己的 background 和 border
一个 layer 除了显示它内容外还可以显示一个背景和边框。背景渲染在 layer 的内容之后，border 在内容之上，如下图。如果 layer 包含 sublayers，它们也在 border 下面。因为背景色在内容之下，背景色会穿过内容任何透明的部分。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/add-border-and-background-to-a-layer.png" width=80% />

下面的代码展示了怎么设置背景色和 border，所有这些属性可做动画。

```objc
myLayer.backgroundColor = [NSColor greenColor].CGColor;
myLayer.borderColor = [NSColor blackColor].CGColor;
myLayer.borderWidth = 3.0;
```

> **注意**: 你可以为 layer 的背景使用任意颜色的背景，包括有透明度的颜色或使用一个 pattern image。当使用 pattern image 的时候，要知道是 Core Graphics 来负责渲染 pattern image，并使用它自己的坐标系，而它的坐标系和 iOS 默认的不一致。因此，iOS 中渲染的图片是倒立的，除非你 flip 坐标系。

如果你设置 layer 的背景色为不透明的颜色，考虑设置 layer 的 `opaque` 属性为 YES。这么做可以提高 compositing 屏幕上的 layer 时的性能，消除 layer 的 backing store 管理 alpha channel 的需要。如果你有圆角的话，一定不要标记一个 layer 为 opaque。

### Layers 支持圆角
你通过给 layer 添加 corner radius 来创建一个矩形圆角。一个 corner radius 是一个视觉修饰，它会隐蔽掉 layer 的 bounds 矩形框的角。如下图所示。因为涉及到应用一个透明的 mask，corner radius 不会影响 layer 的 `contents` 中的图片，除非 `masksToBounds` 属性被设为 `YES`。然而，corner radius 总是影响 layer 的背景和 order。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/a-corner-radius-on-a-layer.png" width=80% />

应用 corner radius 到你的 layer 上，你只需指定 `cornerRadius` 属性的值。radius 值的单位是 points，在显示前应用到四个角。

### Layers Support Built-In Shadows
Layer 对象有几个属性是来配置阴影效果的。一个 Shadow 通过让 layer 看上去像浮在下面的内容上给 layer 添加了深度。在某些场合你会发现这种视觉修饰很有用。使用 layer，你可以控制 shadow 的颜色，相对于 layer 内容的位置，opacity，shape。

layer shadows 的 opacity 默认是 0，也就是影藏 shadow。任何非 0 值会使得 Core Animation 绘制 shadow。因为 shadow 被直接放在 layer 下面，你也许需要改变 shadow 的 offset 来看见它。需要记住的是，你为 shadow 指定的 offsets 是相对于 layer 的本地坐标系而言的，这在 iOS 和 OS X 中是不同的。下图展示了一个延伸到 layer 内容右下的 shadow。iOS 中 y 轴的值要是正的，OS X 中需要是负的

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/apply-a-shadow-to-layer.png" width=80% />

当添加 shadow 到一个 layer 时，shadow 是 layer 内容的一部分，但实际上超出了 layer 的 bounds rectangle。结果是，如果你启用了 `masksToBounds` 属性，shadow 效果会被沿着边裁减。如果你的 layer 包含透明的内容，这会导致奇怪的效果——部分在 layer 内容之下的 shadow 直接可见，但是超过 layer 边界的不可见。如果你想同时使用 shaow 和使用 bounds masking。你可以两个 layers 而不是一个。mask 应用到包含 layer 内容的 layer，然后将这个 layer 添加到有 shadow 效果同样大小的的 layer 中。

### Filters Add Visual Effects to OS X Views
在 OS X 应用中，你可以直接应用 Core Image filters 到 layer 的内容上。你也许使用它们来 blur 或 sharpen layer 的内容，改变颜色，或进行许多其它类型的操作。例如，一个图像处理程序可能使用这些 filters 来不损伤的修改图片，一个视频编辑程序也许使用它们来实现不同类型的视频效果变换。因为 filters 使用硬件来操作 layer 的内容，渲染得快且平滑。

> **注意**: 你不能添加 filters 到 iOS 中的 layer 对象。

对于一个给定的 layer，你可以同时应用 filers 到 layer 的 foreground 和 background 内容。foreground 内容由 layer 自身包含的所有信息组成，包括它的内容，背景色，border，内容的 sublayers。background content 在 layer 之下但不属于 layer 自身一部分。大部分 layer 的 background content 就是它直接 superlayer 的内容，superlayer 的内容要么全部要么部分被挡住。例如，当你想要用户专注于 foreground content 时，你也许会应用一个 blur filter 到 background content。

你可以通过添加 `CIFilter` 对象到 layer 的下面属性来指定 filters:

* `filters` 属性包含一组适用于 layer 的 foreground 内容的 filters
* `backgroundFilters` 属性包含适用于 layer 的 background content 的 filters
* compositingFilter 属性定义了 layer 的 foreground 和 background 内容怎么组合到一起

要添加一个 filter 到 layer，你必须首先创建并配置 `CIFilter` 对象，再添加到 layer 上。`CIFilter` 类包含好几个方法来定位可用的 Core Image Filter，如 `filterWithName:` 方法。创建 filter 只是第一步。许多 filter 有参数来决定 filter 怎么处理图片。例如，a box blur filter 有一个 radius 参数，这个参数会影响 filter 应用的 blur 量是多少。你总是应该为这些参数提供值，作为配置过程的一部分。然而，一个普遍的参数你不用指定，就是输入的图片，这个由 layer 提供。

当添加 filter 到 layer 时，最好先配置 filter 的参数。这么做的主要原因是一旦添加到 layer，你就不能修改这个 `CIFilter` 对象了。然而，你可以使用 layer 的 `setValue:forKeyPath:` 方法来改变 filter 的值。

下例展示了怎么创建和应用 啊pinch distortion filter 到一个 layer 对象. This filter pinches the source pixels of the layer inward, distorting those pixels closest to the specified center point the most. 可以看到不需要指定 filter 的 input image，因为 layer 会提供。

```objc
CIFilter* aFilter = [CIFilter filterWithName:@"CIPinchDistortion"];[aFilter setValue:[NSNumber numberWithFloat:500.0] forKey:@"inputRadius"];[aFilter setValue:[NSNumber numberWithFloat:1.25] forKey:@"inputScale"];[aFilter setValue:[CIVector vectorWithX:250.0 Y:150.0] forKey:@"inputCenter"];

myLayer.filters = [NSArray arrayWithObject:aFilter];
```

## The Layer Redraw Policy for OS X Views Affects Performance
在 OS X 中，layer-backed view 支持好几种不同策略来决定什么更新潜在 layer 的内容。因为在 native AppKit drawing model 和 Core Animation 引入的不同，这些策略使得老的代码可以轻松的迁移到 Core Animation。你可以配置一个接一个的配置这些视图的策略来保证你的每个视图的最优性能。

每个 view 定义了 `layerContentsRedrawPolicy` 方法，并返回视图的 layer 重绘的策略。你可以通过调用 `setLayerContentsRedrawPolicy:` 方法来设置策略。为了与原有的 drawing model 保持兼容，AppKit 默认设置重绘策略为 `NSViewLayerContentsRedrawDuringViewResize`。然而，你可以改变策略为下表中的任意一个。

Policy | Usage
-------- | ---------
NSViewLayerContentsRedrawOnSetNeedsDisplay | 这是推荐的策略。使用这个策略，view 的几何变化不会自动引起 view 更新 layer 的内容。相反，layer 现有的内容被拉升和操作来适应几何变化。要强制视图重绘和更新它的内容的话，你必须显式调用视图的 `setNeedsDisplay:` 方法。<br />这个策略最接近 Core Animation layer 的标准行为。虽然它不是默认策略，并必须显式设置。
NSViewLayerContentsRedrawDuringViewResize | 这是默认的策略。这个策略保持了与传统的 AppKit 最大的兼容性，每次 view 的几何发生变化的时候，AppKit 都会重新缓存 layer 的内容。这种行为导致了视图的 `drawRect:` 方法在应用的主线程上被多次调用。
NSViewLayerContentsRedrawBeforeViewResize | 使用这个策略，AppKit 在任何 resize 操作之前以 layer 最终大小绘制 layer 并缓存它的内容。resize 操作使用缓存的位图作为开始图片，伸缩它来适应老的 bounds rectangle。然后动画显式到 bitmap 的最终大小。这种可能会导致 view 的内容在动画开始看上去是被拉伸或压缩，这在开始的样子不重要或不可注意的时候很好
NSViewLayerContentsRedrawNever | 使用这个策略，AppKit 根本不更新 layer，即使你调用 `setNeedsDisplay:` 方法。这个策略对于 view 的内容从不改变和 view 的大小基本不改变的地方很适用。例如，你可以使用这个策略来显式固定内容或背景的元素。

视图重绘策略消除了使用独立 sublayers 来提升绘制性能的需要。在引入视图重绘机制前，有些 layer-backed views 绘制的频率比它需要的还多，因此引起了性能问题。这些性能问题的方法是使用 sublyer 来显示 view 中不需要频繁绘制的内容。重绘策略的引入，现在更推荐使用 layer-backed view 的重回机制而不是显式创建 sublayer hierachies。

## Adding Custom Properties to a Layer
`CAAnimation` 和 `CALayer` 类扩展了 key-value coding conventions 来支持自定义属性。你可以使用一个自定义的 key 来添加数据到一个 layer，并获取它。你甚至可以关联 actions 到自定义的属性上，以便当你改变这个属性的值时，一个相应的动画被进行。

## Printing the Contents of a Layer-Backed View
在打印过程中，layers 会根据需要重绘它的内容以使用打印环境。尽管 Core Animation 通常依赖缓存的 bitmap 当渲染到屏幕时，它会重绘内容当打印时。具体的说，如果一个 layer-backed view 使用 `drawRect:` 方法来提供 layer contents，Core Animation 调用 `drawRect:` 来产生可供打印的 layer contents。

---
# Animating Layer Content
Core Animation 和拥有 layers 的 view 的扩展提供的基础设施使得创建应用 layer 的复杂动画变得简单。例子包括改变一个 layer 的 frame rectangle 的大小，改变它屏幕上的位置，应用一个旋转变换，或改变它的 opacity。使用 Core Animation，初始化一个动画和改变属性一样简单，但你也可以创建动画，并显式设置动画的属性。

## Animating Simple Changes to a Layer's Properties
你可以隐式的进行简单动画，或根据你的需要显式的进行。隐式动画使用默认的计时和动画属性来进行一个动画，然而显式动画需要你自己使用一个 animation 对象配置这些属性，所以隐式动画在那些想通过很少的代码做修改并且默认的计时对你而言刚好可以的时候很受欢迎。

简单的动画涉及改变 layer 的属性和让 Core Animation 动画显式这些变化。Layers 定义了很多属性会影响 layer 的视觉外表。改变这些属性中的一个是一种动画显示这种变化的方式。例如，将 layer 的 opacity 从 1.0 变为 0.0 会导致 layer fade out 变成透明的。

> **重要**: 尽管你有时可以直接使用 Core Animation 给 layer-backed view 做动画，但是这么做通常需要额外的步骤。

要引起隐式动画，你需要做的是更新 layer 对象的属性。当修改 layer tree 中的 layer 对象时，你的改变立即被这些对象反应出来。然而这些 layer 对象的视觉外表并不立即发生改变。相反，Core Animation 使用你的修改作为一个导火索，创建和规划一个或多个隐式动画执行。因此，做一个像下例中的改变会导致 Core Animation 创建一个 animation 对象，规划这个动画在下一个更新周期中运行。

```objc
theLayer.opacity = 0.0;
```

要使用 animation 对象创建同样的显式动画的话，需要创建一个 `CABasicAnimation` 对象，使用这个对象配置动画的参数。你可以设置动画开始和结束的值，改变发生的时长，或改变其它的动画参数，这些都得在添加 animation 对象到 layer 之前。下例展示了怎么使用 animation 对象 fade out 一个 layer。当创建这个对象的时候，你指定想要做动画的 key path，然后设置动画参数。要执行这个动画的话，你使用 `addAnimation:forKey:` 方法来添加它到你想做动画的 layers。

```objc
CABasicAnimation* fadeAnim = [CABasicAnimation animationWithKeyPath:@"opacity"];
fadeAnim.fromValue = [NSNumber numberWithFloat:1.0];
fadeAnim.toValue = [NSNumber numberWithFloat:0.0];
fadeAnim.duration = 1.0;
[theLayer addAnimation:fadeAnim forKey:@"opacity"];

// Change the actual data value in the layer to the final value.
theLayer.opacity = 0.0;
```

> **指示**: 当创建一个显式动画时，推荐你赋值给动画对象的 `fromValue`。如果你不指定这个属性的值，Core Animation 使用当前的值作为起始值。如果你已经更新属性到最终值，这也许不会出现你想要的结果。

不像隐式动画，会更新 layer 对象的数据值，显式动画不会修改 layer tree 中的数据。显式动画只会产生动画。动画的最后，Core Animation 会从 layer 上移除动画对象，然后使用 layer 的当前值重绘。如果动画的修改被保持住，你必须按上例那样去修改 layer 的属性。

隐式和显示动画通常在当前的 run loop cycle 结束后开始运行，并且当前线程必须有一个 run loop 以便动画能够运行。如果你修改了多个属性，或你添加了多个动画对象到一个 layer 的话，所有的这些属性改变会被同时做动画。

例如，你可以 fade 一个 layer 的同时从屏幕上移除它，通过同时配置两个动画。然而，你也可以配置动画对象在指定的时间开始。

## Using a Keyframe Animations to Change Layer Properties
一个 property-based 动画将属性的值从起始值变为结束值，而一个 CAKeyframeAnimation 对象让动画以线性或非线性的经过一系列的值。一个关键帧动画由一系列目标值和到达这些目标值的时刻醉成。在最简单的配置，你应该使用数组指定目标值和时间。对于一个 layer 的位置的改变，你也可以让改变沿着一个路径。动画对象通过你指定的关键帧，通过给定的时间函数来填充中间的值来做动画。

下面的代码创建了一个 bounce 的 keyframe 动画

```objc
// create a CGPath that implements two arcs (a bounce)
CGMutablePathRef thePath = CGPathCreateMutable();
CGPathMoveToPoint(thePath,NULL,74.0,74.0);
CGPathAddCurveToPoint(thePath,NULL,74.0,500.0, 320.0,500.0, 320.0,74.0);
CGPathAddCurveToPoint(thePath,NULL,320.0,500.0, 566.0,500.0,566.0,74.0);
CAKeyframeAnimation * theAnimation;

// Create the animation object, specifying the position property as the key path.
theAnimation=[CAKeyframeAnimation animationWithKeyPath:@"position"];
theAnimation.path=thePath;
theAnimation.duration=5.0;

// Add the animation to the layer.
[theLayer addAnimation:theAnimation forKey:@"position"];
```

### Specifying Keyframe Values
Key frame values 是 keyframe 动画最重要的部分。这些值定义了执行过程中动画的行为。指定 keyframe 值的主要方式是使用一组对象，但是对于值类型为 `CGPoint` 类型，你可以指定一个 `CGPathRef` 数据类型。

当你指定一个数组类型值，你放进数组中值的类型依赖于你的属性的要求。你可以直接添加一些对象到数据，但是，一些对象必须转换成 `id` 类型在被添加之前，并且所有的原子类型和结构必须包装在对象里。例如：

* 对于 `CGRect` 类型的属性，用 `NSValue` 包装
* 对于 layer 的 transform 属性，包装 `CATransform3D` 矩阵到 `NSValue` 对象。给这个属性做动画，使得帧动画按顺序的应用转换矩阵。
* 对于 `borderColor` 属性，转换 `CGColorRef` 类型到 `id` 类型，然后添加到数据
* 对于 `CGFloat` 值，包装到 `NSNumber` 对象中
* 当对 layer 的 contents 做动画时，设置一个包含 `CGImageRef` 类型的数组

对于 `CGPoint` 类型的属性，你可以创建一个包含点的数组 (CGPoint 包装在 NSValue 中)，也可以使用 `CGPathRef` 对象来指定。当你设置一组点的时候，keyframe 动画对象在点之间画直线。当你指定一个 `CGPathRef` 对象的时候，动画从 path 的开始点开始，沿着 path 做动画。path 可以是 open 或 closed。

### Specifying the Timing of a Keyframe Animation
keyframe 动画的计时和节奏比基本的动画要复杂，你有好几个属性可以控制它：

* `calculationMode` 属性定义了你是用来计算动画时间的算法。这个属性的值决定了其它时间相关的属性怎么被使用:
	* 线性和立方动画——也就是说，动画的 `calculationMode` 属性被设置为 `kCAAnimationLinear` 或 `kCAAnimationCubic` —— 使用提供的计时信息来产生动画。这些模式给了你对动画计时的最大控制。
	* Paced animation —— 也就是，动画的 `calculationMode` 属性值为 `kCAAnimationPaced` 或 `kCAAnimationCubicPaced` —— 不依赖于属性 `keyTimes` 或 `timingFunctions` 提供的外部计时。相反，时间值被隐式的计算来提供动画恒定的速度。
	* Discrete animations —— `calculationMode` 值设为 `kCAAnimationDiscrete` —— 动画从一个 keyframe 值跳到下一个值，没有中间态。这个计算模式使用 `keyTimes` 但忽略 `timingFunctions` 属性。
* `keyTimes` 属性指定时间标记点来应用这些 keyframe 值。这个属性只在计算模式为 `kCAAnimationLinear`，`kCAAnimationDiscrete`，`kCAAnimationCubic` 的时候
* `timingFunctions` 属性指定了用于每个 keyframe 段的时间曲线。(这个属性会替换掉继承的 `timingFunction` 属性)。

如果你想要自己处理计时，使用 `kCAAnimationLinear` 或 `kCAAnimationCubic` 模式和 `keyTimes` 和 `timingFunctions` 属性。`keyTimes` 定义了应用 keyframe 值的时间点。中间值的计时是由 timing functions 控制的，这个属性允许你使用 easy-in 或 easy-out 到每段。如果你不指定任何 timing functions，默认使用 linear timing。

## Stopping an Explicit Animation While It Is Running
动画通常跑到它们结束，但是如果需要的话你可以使用以下技术轻松的停止它们：

* 从 layer 中移除一个 animation 对象，调用 `removeAnimationForKey:` 方法移除 animation 对象。这个方法使用传递给 `addAnimation:forKey:` 方法的 key 来标识 animation 对象。你指定的 key 一定不要是 nil
* 从 layer 移除所有 animation 对象，调用 layer 的 `removeAllAnimations` 方法。这个方法移除所有正在进行的动画，使用 layer 的当前信息重绘。

> **注意**: 你不能直接从一个 layer 移除隐式动画

当你从一个 layer 中移除一个动画对象时，Core Animation 通过使用 layer 当前值重绘 layer 来响应。因为 layer 的当前值通常是动画的结束值，这会导致 layer 的外表会突然的跳跃。如果你想要 layer 的外表停留在动画最后帧的位置，你可以使用 presentation tree 中的对象来获取最终的值，然后设置到 layer tree 中的对象上。

## Animating Multiple Changes Together
如果你想要同时应用多个动画到一个 layer 上的话，你可以使用一个 `CAAnimationGroup` 对象来打包它们。使用一个 group 对象可以简化它们的管理，因为 group object 可以提供单一配置点。应用在 group 上的 timing 和 duration 会覆盖单个 animation object 上的配置。

下面的代码使用了一个 animation group 来同时进行相同时长 border 相关的动画，。

```objc
// Animation 1
CAKeyframeAnimation* widthAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderWidth"];

NSArray* widthValues = [NSArray arrayWithObjects:@1.0, @10.0, @5.0, @30.0, @0.5, @15.0, @2.0, @50.0, @0.0, nil];
widthAnim.values = widthValues;
widthAnim.calculationMode = kCAAnimationPaced;

// Animation 2
CAKeyframeAnimation* colorAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderColor"];
NSArray* colorValues = [NSArray arrayWithObjects:(id)[UIColor greenColor].CGColor, (id)[UIColor redColor].CGColor, (id)[UIColor blueColor].CGColor, nil];
colorAnim.values = colorValues;
colorAnim.calculationMode = kCAAnimationPaced;

// Animation group
CAAnimationGroup* group = [CAAnimationGroup animation];
group.animations = [NSArray arrayWithObjects:colorAnim, widthAnim, nil];
group.duration = 5.0;

[myLayer addAnimation:group forKey:@"BorderChanges"];
```

一个更高级的 group animations together 的方式是使用一个 transaction 对象。Transactions 通过允许你创建嵌套的动画集合并给每个配置不同的参数提供了更大的灵活性。

## Detecting the End of an Animation
Core Animation 提供了检测动画什么时候开始和结束的支持。这些通知是记录与动画相关任务的好时机。例如，你也许使用一个开始通知来设置状态相关的信息，使用相应的结束通知来清除相应的状态。

有两种不同的方式来通知动画的状态：

* 添加一个 completion block 到当前的 transaction 上，调用 `setCompletionBlock:` 方法。当前所有的动画结束后，transaction 执行你的 completion block。
* 赋一个 delegate 到 CAAnimation 对象，实现 ` animationDidStart:` 和 `animationDidStop:finished:` 方法。

如果你想把两个动画链在一起，以便让一个动画在另一个动画结束之后，不要使用 animation notifications。相反，使用 `beginTime` 属性来在指定的时间开始。链接两个动画，设置第二个动画的开始时间为第一个的结束时间。

## How to Animate Layer-Backed Views
如果一个 layer 属性 layer-backed view，创建动画的方式是使用 UIKit 或 AppKit 提供的 view-based animation interface。有方法直接使用 Core Animation 接口来给这个 layer 做动画，但是怎么创建这些动画依赖于具体的平台。

### Rules for Modifying Layers in iOS
因为 iOS 视图总是有一个潜在的 layer，`UIView` 类自身的很多数据直接来自 layer。结果是，对 layer 的改变会自动直接反应在 view 上。这种行为以为着你可以使用 Core Animation 或 UIView 接口来做这些改变。

如果你想使用 Core Animation 类来初始化这些动画，你必须在 view-based animation block 中发出这些 Core Animation 调用。`UIView` 类默认关闭了 layer 的动画，但在 block 中重新打开了。所以你在 block 外的任何改变是没有动画的。下面的代码展示怎么隐式修改 layer 的 opacity 和显式修改它的位置。例子中，`myNewPosition` 被提前计算好，并被 block 捕获。两个动画同时进行，但是 opacity 动画使用默认的计时，position 动画使用 animation object 中制定的计时。

```objc
[UIView animateWithDuration:1.0 animations:^{
    // Change the opacity implicitly.
    myView.layer.opacity = 0.0;
    
    // Change the position explicitly.
    CABasicAnimation* theAnim = [CABasicAnimation animationWithKeyPath:@"position"];
    theAnim.fromValue = [NSValue valueWithCGPoint:myView.layer.position];
    theAnim.toValue = [NSValue valueWithCGPoint:myNewPosition];
    theAnim.duration = 3.0;
    [myView.layer addAnimation:theAnim forKey:@"AnimateFrame"];
}];
```

### Rules for Modifying Layers in OS X
要在 OS 中动画显式 layer-backed view 的改变，最好使用 view 自身的接口。你应该很少会直接附着在 layer-backed NSView 对象上的 layer。AppKit 负载创建和配置这些 layer 对象和管理它们。修改 layer 会导致它与 view 对象不同步并导致意想不到的结果。对于 layer-backed views，你的代码绝对不要修改 layer 的以下属性：

* anchorPoint
* bounds
* compositingFilter
* filters
* frame
* geometryFlipped
* hidden
* position
* shadowColor
* shadowOffset
* shadowOpacity
* shadowRadius
* transform

> **重要**: 上面的这些限制不适用于 layer-hosting views。如果你手动创建 layer 对象并手动与 view 关联，你负责修改 layer 的这些属性和保持 view 的同步。

AppKit 默认关闭了 layer-backed view 的隐式动画。view‘s animator proxy 对象自动为你启用隐式动画。如果你想要直接给 layer 的属性做动画，你也可以通过改变 `NSAnimationContext` 对象的 `allowsImplicitAnimation` 属性的值来启用隐式动画。再次申明，你只应该给不在上述列表的属性这么做。

### Remember to Update View Constraints as Part of Your Animation
如果你使用 constraint-based 布局规则来管理视图的位置，你必须移除配置动画过程中任何可能影响动画 的 constants。Constraints 会影响你对 view 的位置和大小做的修改。它们也影响 view 和 child views 之间的关系。如果你给这些视图做动画，你可以移除 constraint，做你想做的改变，然后应用需要的新的 constraint。

---
# Building a Layer Hierachy
大部分时候，应用中使用 layers 的方式是与一个 view 对象一起使用。然而，有时你需要通过添加额外的 layer 对象到你的 view 来增加它的层次。你也许使用 layer 来提升性能，或让你实现一个 view 很难实现的 feature。在这些场合，你需要知道怎么管理你创建的 layer 层次。

> **重要**:  自 OS X v10.8 起，推荐你最小化使用 layer hierarchies 并只使用 layer-backed views。OS X 引入的 layer 重绘机制让你自定义 layer-backed views 的行为并同时获得之前单独使用 layer 时可能获得的性能。

## Arranging Layers into a Layer Hierarchy
Layer hierarchies 在许多方面和 view hierarchies 相像。你嵌入一个 layer 到另一个创建了 parent-child 的关系。这种 parent-child 关系 sublayer 的很多方面。例如，它的内容会在 parent 之上，它的位置是相对于 parent 的坐标而言的，并且它被任何应用到 parent 的 transform 影响。

### Adding, Inserting, and Removing Sublayers
每一个 layer 对象有添加，插入和移除 sublayers 的方法。下表总觉了这些方法和它们的行为

Behavior | Methods | Description
------------ | ------------- | ---------------
Adding Layers | `addSublayer:` | 添加一个新 sublayer 到当前 layer。sublayer 被添加到 layer 的 sublayers 列表的最后。这导致 sublayer 显式在有相同 `zPosition` 属性值的兄弟 layer 上面。
Inserting layers | `insertSublayer:above:`<br />`insertSublayer:atIndex:`<br />`insertSublayer:below:` | 插入 sublayer 到指定的 index 或相对于另一个 layer 的位置。当你在一个 layer 之上或之下插入一个 layer 时，你只需指定 sublayer 在 sublayers 数据中的位置。layers 的可见性主要由它们的 `zPosition` 决定，其次才是由它们在 sublayers 数组中的位置决定。
Removing layers | `removeFromSuperlayer` | 从 parent layer 中移除 sublayer。
Exchanging layers | `replaceSublayer:with:` | 交换两个 layer。如果你插入的 sublayer 已经另一个 layer 的层次中，它首先被移除。

当你与 layer 工作的时候你使用上述方法。你不会使用这些方法来安排属于 layer-backed view 的 layer。然而，一个 layer-backed view 可以作为你独立创建的 layer 的 parent。

### Positioning and Sizing Sublayers
当添加或插入 sublayers 的时候，你必须在显示到屏幕之前设置它的大小和位置。在 sublayer 被添加到 layer hierarchy 中你可以修改它的位置和大小，但是你应该养成创建它们的时候设置这些值。

你可以通过 `bounds` 属性来修改 sublayer 的大小，使用 `position` 属性修改它在 superlayer 中的位置。bounds 矩形框的 origin 几乎总是 (0, 0) ，大小可以是任何你想的大小，以点为单位。`position` 的值是相对于 layer's anchor point 而言的，默认是在中心。如果你不指定这些值，Core Animation 设置 layer 的初始长宽为0，开始位置为 (0, 0)。

```objc
myLayer.bounds = CGRectMake(0, 0, 100, 100);
myLayer.position = CGPointMake(200, 200);
```

> **注意**: 对于 layer 的长宽总是使用整数。

### How Layer Hierarchies Affect Animations
一些 superlayer 的属性会影响应用到 sublayers 上动画。其中一个属性是 `speed` 属性，它是动画的 speed 的乘数。这个属性的默认值是 1.0，但将它的值变为 2.0 会导致动画以原来两倍的速度运行，因此在一半的时间内完成。这个属性不仅影响你所设置的 layer，也影响 layer 的 sublayers。这样的修改是相乘的。如果一个 sublayer 和它的 superlayer 同时有速度 2.0，那么这个 layer 上的速度将是原来速度的 4 倍。

大部分 layer 的修改都会按预定的方式影响它的 sublayers。例如，应用一个 rotation transform 到一个 layer 会 rotate 这个 layer 和它的所有 sublayers。同样，改变一个 layer 的 opacity 会改变它的 sublayers 的 opacity。

## Adjusting the Layout of Your Layer Hierarchies
Core Animation 支持好几种方式来调整 sublayers 的位置和大小来响应它们的 superlayer 的改变。在 iOS 中，layer-backed view 的普遍使用使得创建 layer hierarchy 变得不重要；只支持手动的 layout updates。对于 OS X，存在几种其他选择使得管理 layer hierarchies 变得简单。

如果你使用单独的 layer 对象来创建 layer hierarchies 的话  Layer-level layout  是唯一合适的。如果你的 layer 是跟 view 相关联的话，使用 view-based layout 来更新你的 view 的位置和大小。

### Using Constraints to Manage Your Layer Hierarchies in OS X
Constraint 允许你设定一系列 layer 与 superlayer、layer 与 sibling layer 之间的关系来指定它的位置和大小。定义 constraint 需要以下步骤：

1. 创建一个或多个 `CAConstraint` 对象。使用这些对象来定义 constraint 参数
2. 添加你的 constraint 对象到它们修改属性的 layer
3. 获取共享的 `CAConstraintLayoutManager` 对象，并赋给最近的 superlayer

下图展示了你可以使用来定义 constraint 的属性，和它们影响的边。你可以使用 constraint 来修改 layer 的位置，基于它的边的中点相对于另一个 layer 的位置。你也可以使用它们改变 layer 的大小。大小的改变可以是跟 superlayer 成比例的或相对于另一个 layer。你也可以添加一个伸缩因子或常量。这种额外的灵活性使得通过一系列简单的规则来精准的控制一个 layer 的大小和位置变得可能。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/constraint-layout-manager-attributes.png" width=80% />

每个 constraint 对象封装了 layers 之间在同一轴上的一个几何关系。每个轴最多两个 constraint 对象，并且是这两个 constraint 决定了哪个属性是可变的。例如，如果你指定 layer 左右边的 constraint，layer 的大小随之改变。如果你指定左边和宽的 constraints，那么 layer 的右边位置改变。如果你指定 layer 某条边的 constraint，Core Animation 创建一个隐式的 constraint 来保证 layer 的大小在给定的方向是固定的。

当你创建 constraint 时，你总是指定三个信息：

* The aspect of the layer that you want to constrain
* The layer to use as a reference
* The aspect of the reference layer to use in the comparison

下例中的 constraint 将 layer 的竖向中点与它的 superlayer 的竖向中点 pin 一起了。当你引用 superlayer 的时候，使用字符串 "superlayer"。这个字符串是预留来应用 superlayer 的。使用它消除了 superlayer 的指针引用或知道 layer 的名字。这同样允许你改变 superlayer 然后让这个 constraint 自动的应用到 new parent 上。(当创建相对于 sibling layer 的 constraint 时，你必须使用它的 `name` 属性标识)

```objc
[myLayer addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidY relativeTo:@"superlayer" relativeTo:@"superlayer"];
```

运行时应用 constraint ，你必须将共享的 `CAConstraintLayoutManager` 对象附着到直接的 superlayer 上。每个 layer 负责管理它的 sublayers 的布局。将 layout manager 赋给 parent layer 告诉 Core Animation 应用定义好的 constraints。layout manager 对象自动应用 constraints。在赋给 parent layer 之后，你不需要告诉它更新 layout。

要看看在更具体的场景中 constraints 是怎么工作的，可以看下例。设计要求 layerA 的长和宽不变同时 layerA 始终在 superlayer 的中心。另外，layerB 必须和 layerA 对齐，layerB 在 layerA 下面 10 points，layerB 的底边必须保持在 superlayer 底边 10 point 之上。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/constraint-based-layout-png“ width=80% />

```objc
// Create and set a constraint layout manager for the parent layer.
theLayer.layoutManager=[CAConstraintLayoutManager layoutManager];

// Create the first sublayer.
CALayer *layerA = [CALayer layer];
layerA.name = @"layerA";
layerA.bounds = CGRectMake(0.0,0.0,100.0,25.0);
layerA.borderWidth = 2.0;

// Keep layerA centered by pinning its midpoint to its parent's midpoint.
[layerA addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidY relativeTo:@"superlayer" attribute:kCAConstraintMidY]];
[layerA addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidX relativeTo:@"superlayer" attribute:kCAConstraintMidX]];
[theLayer addSublayer:layerA];

// Create the second sublayer
CALayer *layerB = [CALayer layer];
layerB.name = @"layerB";layerB.borderWidth = 2.0;

// Make the width of layerB match the width of layerA.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintWidth relativeTo:@"layerA" relativeTo:@"layerA"

// Make the horizontal midpoint of layerB match that of layerA
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidX  relativeTo:@"layerA" attribute:kCAConstraintMidX]];

// Position the top edge of layerB 10 points from the bottom edge of layerA.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMaxY relativeTo:@"layerA" attribute:kCAConstraintMinY offset:-10.0]];

// Position the bottom edge of layerB 10 points
//  from the bottom edge of the parent layer.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMinY relativeTo:@"superlayer" attribute:kCAConstraintMinY offset:+10.0]];
[theLayer addSublayer:layerB];
```

上例中值得注意的一个有趣点是代码没有显式的设置 layerB 的大小。因为定义的 constraints，layerB 的 width 和 height 在每次 layout 更新的时候自动更新。因此，使用 bounds 设置大小没有必要。

> **警告**: 当创建 constraints 时，不要在 constraints 间创建循环引用。循环引用会使得计算所需的布局信息变得不可能。当遇到这样的循环引用的时候，布局行为是不确定的。

### Setting Up Autoresizing Rules for Your OS X Layer Hierarchies
Autoresizing rules 是 OS X 中另一种调节 layer 大小和位置的方式。使用 autoresizing rules，你指定 layer 的边与 superlayer 相应的边间的距离是固定的还是可变的。同样你可以指定 layer 的 width 和 height 是否是可变的。你指定的关系总是在 layer 与 superlayer 之间。你不能使用 autoresizing rules 来指定 sibling layers 间的关系。

要设置一个 layer 的 autoresizing rules，你必须设置合适的常量值到 `autoresizingMask` 属性上。默认，layer 被配置成有固定的 width 和 height。在 layout 的过程中，Core Animation 自动的帮你计算 layer 准确的大小和位置，这过程涉及到一系列基于很多因子的运算。Core Animation 在要求你的 delegate 做手动 layout 更新之前应用 autoresizing 行为，所以你可以使用 delegate 来调节 autoresizing 的结果。

### Manually Laying Out Your Layer Hierarchies
在 iOS 和 OS X 中，你可以通过实现 superlayer delegate 对象的 `layoutSublayersOfLayer:` 方法。你使用这个方法来调节 sublayers 的位置和大小。当你手动做这些更新的时候，由你来进行一些计算来得出 sublayers 的位置。

如果你在实现一个自定义的 layer 子类，你的子类可以覆盖 `layoutSublayers` 方法，并使用这个方法来处理布局任务。你只应该在你需要完全控制你的 layer 中的 sublayer 位置时覆盖上述方法。在 OS X 中覆盖这个方法阻止了 Core Animation 应用 constraints 或 autoresizing rules。

## Sublayers and Clipping
跟 view 不一样，一个 superlayer 不会自动裁剪超过它 bounds 的 sublayers 的内容。相反，superlayer 默认允许它的 sublayers 完整的展示自己。然而，你可以通过设置 `masksToBounds` 为 `YES` 来启用裁剪的行为。

一个 layer 的 clipping mask 包括 layer 的 corner radius，如果有一个指定的话。下图展示了一个 layer 的 `masksToBounds` 是怎么影响有 corner radius 的。当该属性为 `NO` 时，sublayers 完整展示，即使它们超过了 parent layer 的边界。改变属性为 `YES` 时，导致它们的内容被裁剪。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/clipping-sublayers-in-parent.png" width=80% />

## Converting Coordinate Values Between Layers
有时，你可能需要在不同 layer 间转换坐标。 `CALayer` 类提供了一系列的方法来供你使用：

* `convertPoint:fromLayer:`
* `convertPoint:fromLayer:`
* `convertPoint:toLayer:`
* `convertRect:toLayer:`

除了转换 point 和 rectangle 值外，你也可以使用 `convertTime:fromLayer:` 和 `convertTime:toLayer:` 方法在不同的 layer 间转换时间的值。每个 layer 定义了它自己的时间空间，并使用这个空间来与系统的其它部分来同步动画的开始和结束。这些时间空间默认是同步的；然而，如果你改变一系列 layer 的动画速度的话，这些 layer 的时间空间会相应的改变。你可以使用时间转换函数来考虑这些因素，并保证 layers 间的计时是同步的。

---
# Advanced Animation Tricks
有许多方式配置 property-based 或 keyframe 动画来为你做得更多。需要同时或串行进行多个动画可以使用高级的行为来同步时这些动画的计时或把它们串在一起。你也可以使用其它类型的动画对象来创建视觉 transition 或其它有趣的动画效果。

## Transition Animations Support Changes to Layer Visibility
和名字暗示的一样，一个 transition animation 对象给一个 layer 创建了一个 visual transition。transition 最常用的场景是以协同的方式动画显示一个 layer 和消失一个 layer。跟 property-based 动画不同的是，property-based 动画改变一个 layer 的属性，一个 transition 动画操作一个 layer 缓存的图片，创建通过改变 property 不可能实现的动画。transition 让你进行的标准动画类型有 reveal，push，move，或 crossfade 动画。在 OS X 中，你也可以使用 Core Image filters 来创建使用其它效果的 transition，如 wipes，page curls，ripples，或你自定义的效果。

进行一个 transition 动画，你创建一个 `CATransition` 对象，并把它添加到 transition 涉及的 layer 对象。你使用 transition 对象来指定你需要进行的 transition，transition 动画的起点和终点。你也不需要使用整个 transition animatior。transition 让你指定 start 和 end progress values to use when animating。these values let you do things like start or end an animation at its midpoint.

下面的代码两个 view 间的 push animatiion。例子中 myView1 和 myView2 在同一个 parent 中的同样位置，但是只有 myView1当前是可见的。push 动画使得 myView1 从屏幕移除到左边，fade until it is hidden，同时 myView2 从屏幕右侧滑入，become visible。更新两个 view 的 hidden 属性保证了它们的可见性在动画结束时候是正确的。

```objcCATransition* transition = [CATransition animation];
transition.startProgress = 0; 
transition.endProgress = 1.0;
transition.type = kCATransitionPush;
transition.subtype = kCATransitionFromRight;
transition.duration = 1.0;

// Add the transition animation to both layers
[myView1.layer addAnimation:transition forKey:@"transition"];
[myView2.layer addAnimation:transition forKey:@"transition"];

// Finally, change the visibility of the layers.
myView1.hidden = YES;
myView2.hidden = NO;
```

当两个 layers 涉及到想到的 transition，它们可以使用相同的 transition 对象。使用相同的 transition 对象简化了你需要写的代码。然而，你可以使用不同的 transition 对象，而且当 transition 的参数不同的时候，你必须这么做。

下面的代码展示了在 OS X 中怎么使用一个 Core Image filter 来实现一个 transition 效果。在使用你想要的参数配置好你的 filter 之后，将它赋给 transition 对象的 filter 属性。在这之后，使用动画的过程和使用其它动画对象是一样的

```objc
// Create the Core Image filter, setting several key parameters.
CIFilter* aFilter = [CIFilter filterWithName:@"CIBarsSwipeTransition"];[aFilter setValue:[NSNumber numberWithFloat:3.14] forKey:@"inputAngle"];[aFilter setValue:[NSNumber numberWithFloat:30.0] forKey:@"inputWidth"];[aFilter setValue:[NSNumber numberWithFloat:10.0] forKey:@"inputBarOffset"];

// Create the transition objectCATransition* transition = [CATransition animation];
transition.startProgress = 0;transition.endProgress = 1.0;transition.filter = aFilter;transition.duration = 1.0;

[self.imageView2 setHidden:NO];
[self.imageView.layer addAnimation:transition forKey:@"transition"];
[self.imageView2.layer addAnimation:transition forKey:@"transition"];
[self.imageView setHidden:YES];
```

> **注意**: 当在动画中使用 Core Image Filters 时，配置 filter 的过程最麻烦的。例如，对于 bar swipe transition，指定一个过大或过小的 input angle 可能导致没有 transitiion 在发生。如果你没有看见你期望的动画，试着调整 filter parameters to 不同的参数来结果是否是你期望的。

## Customizing the Timing of an Animation
Timing 是动画的重要的一部分，对于 Core Animation，你通过 `CAMediaTiming` protocol 中的方法和属性为你的动画指定精确的 timing 信息。两个 Core Animation 类实现了这个协议。`CAAnimation` 实现了这个协议以便你可以指定动画的时间。`CALayer` 实现了这个协议以便你可以配置一些 timing 相关的特性给你的隐式动画，尽管包装这些动画的隐式 transaction 对象提供的默认 timing 信息会优先。

当讨论 timing 和动画时，理解 layer 对象是怎么与 time 工作的很重要。每个 layer 有它自己的 local time，它用来管理动画计时。通常，两个不同 layer 的 local time 很接近所以你可以给每个 layer 指定相同的时间，用户不会注意到什么。然而，一个 layer 的 local time 是被它的 parent layers 或它自己的 timing parameters 所改变。例如，改变一个 layer 的 `speed` 属性导致这个 layer (包含它的 sublayers) 上的动画时间按比例的改变。

未来帮助你保证对于一个给定的 layer 时间值是合适的，`CALayer` 类定义了 `convertTime:fromLayer:` 和 `convertTime:toLayer:` 方法。你可以使用这些方法来转换一个固定时间值到一个 layer 的 local time 或转换一个 layer 的时间值到另一个 layer 中。这些方法会考虑到可能影响 local time 的 media timing property，并返回一个你可以在另一个 layer 中使用的时间。下面的代码展示了你通常用来获取一个 layer 的本地时间的逻辑。`CACurrentMediaTime` 是一个返回电脑当前 clock time 的方便函数，方法会将这个值转换到 layer 的local time。

```objc
CFTimeInterval localLayerTime = [myLayer convertTime:CACurrentMediaTime() fromLayer:nil];
```

一旦你有一个 layer local time 的时间值，你可以使用用来更新一个动画对象或 layer 对象 timing 相关的属性，你可以实现有趣的动画行为，包括：

* 使用 `beginTime` 属性来设置一个动画开始的时间。通常，动画在下一个 updating cycle 开始。你可以使用 `beginTime` 参数来延迟动画开始时间几秒钟。将动画串在一起的方式设置一个动画的开始时间与另一个动画结束时间相匹配。<br />如果你延迟一个动画，你也许想要设置动画的 `fillMode` 为 `kCAFillModeBackwards`。这个 fill mode 导致 layer 显式动画的开始值，即使在 layer tree 中的 layer 对象包含一个不同值。没有这个值的话，你会看到在动画开始前 layer 直接 jump 到最终的位置。
* `autoreverses` 导致一个动画执行指定的时长，然后返回到动画开始的位置。你可以和 `repeatCount` 属性一起使用来动画来回显示。设置 repeat count 为一个 whole number (如 1.0) 导致动画停留在开始的位置。添加一个 extra half step (如 1.5) 导致动画停在结束的位置。
* 和 group animations 一起使用 `timeOffset` 属性来晚点儿启动一些动画
 
 ## Pausing and Resuming Animations
 要暂停一个动画，你可以利用动画实现 `CAMediaTiming` 的事实，并设置动画的 speed 为 0.0。设置动画的 speed 为 0 暂停动画直到你将它改为非 0 的值。下面的代码展示了一个简单的例子，关于怎么暂停和恢复一个动画。
 
 ```objc
-(void)pauseLayer:(CALayer*)layer {
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

- (void)resumeLayer:(CALayer*)layer {
    CFTimeInterval pausedTime = [layer timeOffset];
    layer.speed = 1.0;layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}
```

## Explicit Transactions Let You Change Animation Parameters
每个你对 layer 所做的改变必须是一个 transaction 的一部分。CATransaction 类管理动画的创建，组合和在合适的时机执行它们。在大部分时候，你不需要创建你自己的 transaction。无论什么时候你添加一个显式动画或隐式动画到你的一个 layer，Core Animation 自动创建一个隐式 transaction。然而，你也可以显式创建 transactions 来更精确的管理这些动画。

你使用 `CATransaction` 类的方法来创建和管理 transactions。要开始 (和隐式创建) 一个新的 transaction，调用 `begin` 类方法；要结束这个 transaction，调用 `commit` 类方法。在这两个调用间的改变是这个 transaction 的一部分。例如，改变一个 layer 的两个属性。

```objc
[CATransaction begin];
theLayer.zPosition=200.0;
theLayer.opacity=0.0;
[CATransaction commit];
```

使用 transaction 的一个主要原因是，在一个显式 transaction 的返回内，你可以改变动画时长，timing function，和其它参数。你也可以赋一个 completion block 到这个 transaction 上，以便你的应用可以在一组动画结束后可以被通知。改变动画参数需要使用 transaction dictionary 中合适的 key 来调用 ` setValue:forKey:`。例如，改变默认值时长为 10s，你会使用 `kCATransactionAnimationDuration` key.

```objc
[CATransaction begin];
[CATransaction setValue: @10.0 forKey:kCATransactionAnimationDuration];
// Perform the animations
[CATransaction commit];
```

你可以在你想给不同动画组提供不同的默认值的场合使用嵌套 transactions。将一个 transation 嵌套到另一个，只需要再调用 `begin` 方法。每个 `begin` 调用必须与相应的 `commit` 方法调用是匹配的。只有在你 commit 最外层的 transaction时，Core Animation 才开始关联的动画。

下面的代码展示嵌套 transacation 的一个例子。例子中内部的 transaction 使用与 outer transaction 不同的动画参数。

```objc
[CATransaction begin]; // Outer transaction

// Change the animation duration to two seconds
[CATransaction setValue:@2.0f forKey:kCATransactionAnimationDuration];
// Move the layer to a new position
theLayer.position = CGPointMake(0.0,0.0);

[CATransaction begin]; // Inner transaction
// Change the animation duration to five seconds
[CATransaction setValue:@5.0f forKey:kCATransactionAnimationDuration];

// Change the zPosition and opacity
theLayer.zPosition=200.0;
theLayer.opacity=0.0;

[CATransaction commit]; // Inner transaction
[CATransaction commit]; // Outer transaction
```

## Adding Perspective to Your Animations
应用可以在三维空间中操作 layers，但是为了简单，Core Animation 使用 paralle projection，paralle projection essenctially flattens the scene into a two-dimensional plane. 这种默认行为使得相同大小的 layer 但是不同 `zPosition` 看起来大小相同，即使它们 z 轴上距离较远。The perspective that you normally have viewing such a scene in three dimensions is gone. 然而，你可以通过修改 layers 的 transformation matrix 来改变这种行为来包含 perspective 信息。

当你修改一个场景的 perspective 时，你需要修改 superlayer 的 `sublayerTransform` 矩阵。修改 superlayer 简化了你修改所有 child layer 以应用同样的 perspective 信息的代码。 It also ensures that the perspective is applied correctly to sibling sublayers that overlap each other in different planes.

下面的代码展示了为一个 parent layer 创建一个简单的 perspective transform 的方式。`eyePosition` 变量定义了 z 轴上眼睛与 layers 间的距离。通常你指定一个正值以保证 layers 朝向期望的方向。Larger values result in a flatter scene while smaller values cause more dramatic visual differences between the layers.

```objc
CATransform3D perspective = CATransform3DIdentity;
perspective.m34 = -1.0/eyePosition;

// Apply the transform to a parent layer.
myParentLayer.sublayerTransform = perspective;
```

---
# Changing a Layer’s Default Behavior
Core Animation 使用 action 对象为 layers 实现了隐式动画。一个 action 对象是一个实现了 `CAAction` 协议的对象，定义了在 layer 上执行的相应行为。所有 `CAAnimation` 对象实现了这个协议，通常是这些对象被指定执行当一个 layer 的 property 改变的时候。

动画显示 properties 只是一种类型的 action，但是你可以定义使用几乎所有你想要的行为的 actions。要这么做的话，你必须要定义你的 action 对象，并把它们关联到应用的 layer 对象上。

## Custom Action Objects Adopt the CAAction Protocol
要创建你自己的 action 对象，在你的类中实现 `CAAction` 协议，并实现 `runActionForKey:object:arguments:` 方法。在那个方法中，使用可用的信息来进行任何你想对 layer 进行的 actions。你也许会使用这个方法来添加一个 animation 对象到一个 layer，或使用它来进行其他的任务。

当你定义一个 action 对象的时候，你必须决定你想要你的 actions 对象怎么被 triggered。一个 action 的 trigger 定义了稍后你注册这个 action 所使用的 key。Action 对象可以通过以下情况 trigger：

* layer 的一个属性发生改变。这可以是 layer 的任何一个属性，不仅仅是 animatable 属性。(你可以关联 actions 对象到你自定义的属性上) 标识这个 action 对象的 key 是这个属性的 name。
* layer became visible，或添加到一个 layer hierarchy。标识这个 action 的 key 是 `kCAOnOrderIn`。
* layer 从一个 layer hierarchy 中移除。标识这个 action 的 key 是 `kCAOnOrderOut`
* layer 将要涉及到一个 transition 动画。标识这个 action 的 key 是 `kCATransition`

## Action Objects Must Be Installed On a Layer to Have an Effect
在一个 action 可以被执行前，layer 需要找到相应的 action 对象来执行。layer 相关 actions 的 key 要么是 property 的 name 要么是标识这个 action 的特定字符串。当一个合适的事件发生在 layer 上时，layer 调用 `actionForKey:` 方法来搜索与 key 关联的 action 对象。你的应用可以在搜索过程中的几个不同点介入，并提供一个相关的 action 对象

Core Animation 按以下顺序搜索 action 对象：

1. 如果 layer 有一个 delegate，并且 delegate 实现了 `actionForLayer:forKey:` 方法，layer 会调用这个方法。delegate 必须做以下事之一：
	* 基于给定的 key 返回一个 action 对象
	* 如果它不处理这个 action 返回 nil，这种情况下搜索继续
	* 返回 `NSNull` 对象，这种情况下搜索立即停止
2. layer 对象在它的  `actions` 字典中查找给定的 key
3. layer 查找 `style` 字典，看是否有一个 actions 字典包含给定的 key。(换句话说，`style` 字典包含一个 `actions` key，这个 key 的值也是一个字典。layer 接着在这个字典中查找)。
4. layer 调用它的 `defaultActionForKey:` 方法
5. layer 执行 Core Animation 定义的隐式 action，如果有的话。

如果你在任何合适的搜索点提供一个 action 对象，layer 会停止它的搜索，并执行返回的 action object。当它找到一个 action 对象时，layer 调用它的 `runActionForKey:object:arguments:` 方法来进行这个 action。如果你为给定的 key 定义的 action 已经是一个 `CAAnimation` 类实例，你可以使用这个方法的默认实现来执行动画。如果你在定义你自己的 action 对象，你必须使用你自己的对象的实现来采取合适的行为。

在什么时候插入你的 action 对象依赖于你想要怎样修改你的 layer。

* 对于你只会特定环境下使用的 actions，或对于已经使用 delegate ，提供一个 delegate 并实现它的 `actionForLayer:forKey:` 方法
* 对于不常使用 layer 对象，添加 action 到 layer 的 `actions` 字典中
* 对于自定义属性相关的 action 对象，添加 action 到 layer 的 `style` 字典中
* 对于 layer 行为的基础的 actions，继承 laye，覆盖方法 `defaultActionForKey:`

下面的代码展示了使用 delegate 的方法来提供 action 对象。这个例子中，delegate 查找对 layer 的 `contents` 属性的修改并使用一个 transition 动画用新的内容替换老的内容

```objc
- (id<CAAction>)actionForLayer:(CALayer *)theLayer  forKey:(NSString *)theKey {
    CATransition *theAnimation=nil;
    
    if ([theKey isEqualToString:@"contents"]) {
        theAnimation = [[CATransition alloc] init];
        theAnimation.duration = 1.0;
        theAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
        theAnimation.type = kCATransitionPush;
        theAnimation.subtype = kCATransitionFromRight;
    }
    return theAnimation;
}
```

## Disable Actions Temporarily Using the CATransaction Class
你可以使用 CATransaction 类暂时停掉 layer actions。当你改变一个的属性时，Core Animation 通常创建一个 transaction 对象来动画显式变化。如果你不想要这个动画，你可以通过创建一个显式 transaction 并设置它的 `kCATransactionDisableActions` 属性为 `YES` 来实现。下面的代码显式了关闭移除 layer 动画的代码片段。

```objc
[CATransaction begin];
[CATransaction setValue:(id)kCFBooleanTrue forKey:kCATransactionDisableActions];
[aLayer removeFromSuperlayer];
[CATransaction commit];
```

---
# Improving Animation Performance
对于基于 app 的动画而言，Core Animation 是提升帧率的不错方式，但它的使用并不保证提升性能。在 OS X 中，关于使用 Core Animation 行为最有效方式你必须要做出选择。和所有性能相关的问题一样，你应该使用 Instruments 来长时间测量和追踪应用的性能，以保证性能是在提升而不是降低。

## Choose the Best Redraw Policy for Your OS X Views
 `NSView` 类的默认 redraw policy 保持了它原有的 drawing behavior，即使这个 view 是 layer-backed。如果你在应用中使用 layer-backed views，你应该检查 redraw policy 选项，并选择为你的应用提供最好性能的那个，默认的 policy 并不能提供最好的性能。相反 `NSViewLayerContentsRedrawOnSetNeedsDisplay` policy 更可能减少 app 所做的 drawing，并提升性能。其它的 policies 对于特定类型的 view 也许能提供更好的性能。
 
## Update Layers in OS X to Optimize Your Rendering Path
自 OS X v10.8 起，views 有两个选择来更新它潜在的 layer 的内容。当你在  OS X v10.7 和之前的系统中更新一个 layer-backed view 时， the layer captures the drawing commands from the view’s drawRect: method into the backing bitmap image. Caching the drawing commands is effective but is not the most efficient option in all cases. If you know how to provide the layer’s contents directly without actually rendering them, you can use the updateLayer method to do so.
 
## General Tips and Tricks
有好几种方法使得你的 layer 实现高效。和任何类似的优化一样，你总是应该测量你的代码当前的性能在优化前。这给了你测量优化是否有效的测量基准。

### Use Opaque Layers Whenever Possible
设置 layer 的 `opacity` 属性为 `YES`，让 Core Animation 知道它不需要维护 layer 的 alpha channel。没有 alpha channel 意味着 compositor 不需要混合这个 layer 后面的内容，这在渲染的时候可以节约时间。然而，这个属性主要是对于 layer-backed view 的 layer 和 Core Animation 负责创建 layer 潜在内容的 layer 而言的，对于 layer 的 `contents` 是图片的是情况，图片的 alpha channel 总是被保存的，不论 `opacity` 属性的值是多少。

### Use Simpler Paths for CAShapeLayer Objects
在 composite 的时候，`CAShapeLayer` 类通过你提供的 path 来渲染它的内容到一个位图中。这样做的好处是 layer 总是以最好的分辨率来绘制 the path，但是这样消耗了渲染时间。如果你提供的 path 很复杂，rasterizing 这个 path 也许会损耗太多时间。如果 layer 的大小频繁的变化的话，用来绘制总时间可能会成为一个性能瓶颈。

一种最小化 shape layer 绘制时间的方法是分解复杂的 shapes 成简单的 shapes。使用简单 path 然后 layering multiple `CAShapeLayer` on top of one another in the compositor can be much faster than one large complex path。这是因为绘制操作发生在 CPU 上，然而 compositing 发生在 GPU 上。和任何这种类型的简化一样，潜在的性能提升依赖于你的内容。因此，优化前测试你的性能很重要，以便你有比较的基准。 

### Set the Layer Contents Explicitly for Identical Layers
如果你在多个 layer 中使用相同的图片的话，手动加载这张图片，直接将它赋给这些 layer 对象的 `contents` 属性。赋一个图片给 `contents` 属性阻止 layer 给 backing store 分配内存。相反，layer 使用你提供的 image 作为它的 backing store。当多个 layers  使用相同的图片时，这种方式使得这些 layers 对象使用相同的内存而不是它们每个一个 copy。

### Always Set a Layer’s Size to Integral Values
为了最好的结果，始终设置 layer 的 width 和 height 为整数值。尽管你可以指定 layer 的 width 和 height 为浮点数，layer bounds 最终被用来创建一个 bitmap image。为 layer 的 width 和 height 指定整数简化了 Core Animation 创建和管理 backing store 和其他 layer 信息等必须做的工作。

### Use Asynchronous Layer Rendering As Needed
任何你在 layer 的 delegate 方法 `drawLayer:inContext:` 或你视图的 `drawRect:` 方法中做的绘制通常同步的在主线程上发生。某些场合，同步绘制内容可能并不提供最好的性能。如果你注意到动画有卡顿，你也许会启用 `drawsAsynchronously` 属性来将这些绘制操作移至后台线程。如果你这么做的话，保证你的绘制代码是线程安全的。同样，请测量你的性能改变。

### Specify a Shadow Path When Adding a Shadow to Your Layer
让 Core Animation 决定一个 shadow 的 shape 很昂贵并影响你应用的性能。与其让 Core Animation 决定 shadow 的 shape，不如使用 layer 的 `shadowPath` 属性显式指定 shadow 的 shape。当你为这个属性指定一个 path 对象时，Core Animation 使用这个 shape 来绘制并缓存 shadow effect。对于这些 shape 从不改变的 layers，这样做会大大提升性能，因为这样做减少了 Core Animation 所需做的渲染。

---
# Appendix: Layer Style Property Animations
在绘制过程中，Core Animation 使用 layer 的几个属性，并按指定的顺序渲染它们。这个顺序决定了 layer 最终的外表。这章将说明通过设置 layer 不同的样式属性所达到的效果。

## Geometry Properties
一个 layer 的 geometry properties 指定了它相对于它的 parent layer 怎样展示。这些 geometry 也指定了 layer 的圆角半径，和应用到 layer 和 sublayers 的 transform。

下面的属性定义了一个 layer 的 geometry：

* `bounds`
* `position`
* `frame` (从 bounds 和 position 算出来的)
* `anchorPoint`
* `cornerRadius`
* `transform`

## Background Properties
Core Animation 第一个渲染的是 layer 的背景。你可以指定一个颜色作为 layer 的背景。OS X 中，你也可以指定一个 Core Image Filter 应用到 layer 的背景内容上。下图显示了同一个 layer 两个版本。左边的 layer 设置了它的背景色，右边的没有背景色，但有一个边框，一些内容和一个 pinch distortion filter 赋到 `backgroundFilters` 属性上。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-with-background-color.png" width=80% />

background filters 应用到 layer 之后的内容，主要是有 parent layer 的内容组成。你也许使用一个 background filter 来使 foreground layer 内容凸显。例如，使用一个 blur filter。

`CALayer` 的下面属性影响 layer 的背景：

* `backgroundColor`
* `backgroundFilters` (not supported in iOS)

> **PlatformNote:** IniOS, the `backgroundFilters` property is exposed in the CALayer class but the filters you assign to this property are ignored.

## Layer Content
如果 layer 有任何内容的话，内容在背景上渲染。你可以通过直接设置一个位图给 layer 提供内容，或使用一个 delegate 来提供内容，或继承 layer 然后直接绘制内容。你可以使用许多不同的绘制技术 (包括 Quartz，Metal，OpenGL，Quatz Composer) 来提供内容。下图显示一个直接通过位图设置内容的 layer。位图内容由大部分透明空间和一个 Automator icon 组成。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-display-a-bitmap-image.png" width=80% />

Corner radius 并不会自动裁剪 layer 的内容。然后，设置 layer 的 `masksToBounds` 属性为 `YES` 之后会导致 layer 裁剪它的 corner radius。

下面的属性会影响 layer 展示的内容：

* `contents`
* `contentsGravity`
* `masksToBounds`

## Sublayers Content
任何 layer 可以包含一个或多个 child layers，被称作 sublayers。sublayers 递归的渲染，相对于 parent layer 的 bounds 定位。另外，Core Animation 应用 parent layer 的 `sublayerTransform` 属性到每个 sublayer 上，且这个 transform 是相对于 parent layer 的 `anchorPoint` 而言的。你可以使用 sublayer tranform 来应用 perspective 和其它效果到所有的 layers 上。下图显式了一个有个两个 sublayer 的 layer。左边的有背景色，右边的没有。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-display-the-sublayers-content.png" width=80% />

设置 `masksToBounds` 属性为 `YES` 会导致超出边界的 sublayers 被裁剪

影响 sublayers 显示的属性如下：

* `sublayers`
* `masksToBounds`
* `sublayerTransform`

## Border Attributes
一个 layer 可以使用指定的颜色和宽度来显示可选的 border。border 沿着 layer rectangle 的边框并会考虑任何 core radius。下图显示了一个有 border 的 layer。注意在 layer 的 bounds 外的内容在 layer 的 border 之下。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-display-border.png" width=80% />

下面的属性影响 layer 的 border 显示：

* `borderColor`
* `borderWidth`

## Filters Property
在 OS X 中，你也许会应用一个或多个 filter 到 layer 的内容上，使用一个自定义 compositing filter 来指定 layer 的内容怎么和 layer 下面的内容混合。下例显式了一个使用 Core Image posterize filter 的 layer

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-displaying-filter-properties.png" width=80% />

通过下面属性指定 layer 的 content filter:

* `filters`
* `compositingFilter`

> **Platform Note**: iOS 中，layers 忽略这些属性的值。

## Shadow Properties
Layers 可以显示 shadow effects，并配置它们的 shape，opacity，color，offset，和 blur radius。如果你不指定一个自定义 shadow shape，shadow 的 shape 会基于 layer 不全透明的部分。下图显示了一个有 red shadow 的 layer 的不同版本。

左边和中间的版本包含一个背景色，所有 shadow 只出现在 layer 边框附近。然而，右边的版本不包含背景色。这种情况，shadow 会应用到 layer 的内容，边框，sublayers。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-displaying-shadow-properties.png" width=80% />

影响的 shadow 的属性有：

* `shadowColor`
* `shadowOffset`
* `shadowOpacity`
* `shadowRadius`
* `shadowPath`

## Opacity Property
opacity 属性决定了有多少 background 内容穿过 layer 显示出来。下图显式了一个 opacity 为 0.5 的 layer

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-displaying-shadow-properties.png" width=80% />

下面的属性影响 layer 的 opacity:

* `opacity`

## Mask Properties
你可以使用 mask 掉部分或全部 layer 的内容。mask 自身是一个 layer 对象，它的 alpha channel 被用来决定什么被 blocked 掉，什么被 transmitted。mask layer 的 opaque 部分允许允许下面的内容显示，透明的部分 partially 或 fully obscure the underlying content。下图展示了一个有 mask layer 和不同背景的 layer。左边的版本的 opacity 被设置为 1.0，右边的版本的 opacity 被设置为 0.5。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/CoreAnimation/layer-composited-with-mask-property.png" width=80% />

下面的属性指定了 layer 的 mask layer：

* `mask`

---
# Appendix B: Animatable Properties
`CALayer` 和 `CIFilter` 中的很多属性是可以做动画的。这个 appendix 列出了这些属性，和与这些属性相随的默认动画。

## CALayer Animatable Properties
下表列出了你可能考虑做动画的 CALayer 属性。对于每个属性，列表也给出了默认动画类型，这些默认类型被创建并执行，这正是我们看到的隐式动画。

Property | Default Animation
----------- | -------------------------
`anchorPoint` | 使用默认隐式的 `CABasicAnimation` 对象。
`backgroundColor` | 使用默认隐式的 `CABasicAnimation` 对象
`backgroundFilters` | 使用默认的 `CATransition` 对象。filter 的 sub-properties 使用默认的 `CABasicAnimation` 对象
`borderColor` | 使用默认隐式的 `CABasicAnimation` 对象
`borderWidth` | 使用默认隐式的 `CABasicAnimation` 对象
`bounds` | 使用默认隐式的 `CABasicAnimation` 对象
`compositingFilter` | 
`contents` | 使用默认隐式的 `CABasicAnimation` 对象
`contentsRect` | 使用默认隐式的 `CABasicAnimation` 对象
`cornerRadius` | 使用默认隐式的 `CABasicAnimation` 对象
`doubleSided` | 没有默认隐式动画
`filters` | 使用默认 `CABasicAnimation` 对象。filter 的 sub-properties 使用 `CABasicAnimation` 对象。
`frame` | 这个属性 is not animatable。你可以通过使用 `bounds` 和 `position` 达到同样的目的
`hidden` | 使用默认隐式的 `CABasicAnimation` 对象
`mask` | 使用默认隐式的 `CABasicAnimation` 对象
`masksToBounds` | 使用默认隐式的 `CABasicAnimation` 对象
`opacity` | 使用默认隐式的 `CABasicAnimation` 对象
`position` | 使用默认隐式的 `CABasicAnimation` 对象
`shadowColor` | 使用默认隐式的 `CABasicAnimation` 对象
`shadowOffset` | 使用默认隐式的 `CABasicAnimation` 对象
`shadowOpacity` | 使用默认隐式的 `CABasicAnimation` 对象
`shadowPath` | 使用默认隐式的 `CABasicAnimation` 对象
`shadowRadius` | 使用默认隐式的 `CABasicAnimation` 对象
`sublayers` | 使用默认隐式的 `CABasicAnimation` 对象
`sublayerTransform` | 使用默认隐式的 `CABasicAnimation` 对象
`transform` | 使用默认隐式的 `CABasicAnimation` 对象
`zPosition` | 使用默认隐式的 `CABasicAnimation` 对象

下表列出了默认的 property-based 动画的默认参数：

Description | Value
---------------- | --------
Class | `CABasicAnimation`
Duration | 0.25s, 或当前 transaction 的时长
Key path | 设置为 layer 的 property name

下表列出了默认 transition-based 动画的配置信息

Description | Value
---------------- | ---------
Class | `CATransition`
Duration | 0.25s，或当前 transaction 的时长
Type | Fade (`kCATransitionFade`)
Start progress | 0.0
End progress | 1.0

## CIFilter Animatable Properties
Core Animation 添加了以下动画属性到 Core Image 的 `CIFilter` 类。这些属性只在 OS X 中可用。

* name
* enabled

更多的信息可参见 `CIFilter Core Animation Additions`

----
# Appendix C: Key-Value Coding Extensions
Core Animation 扩展了 `NSKeyValueCoding` protocal，因为这跟 `CAAnimation` 和 `CALayer` 相关。这个扩展给一些 key 添加了默认值，扩展了 wrapping conventions，添加了对 `CGPoint`, `CGRect`, `CGSize`, `CATransform3D` 类型的支持。

## Key-Value Coding Compliant Container Classes
`CAAnimation` 和 `CALayer` 类是 key-value coding compliant container classes，这意味着你可以设置任意的 keys。即使某些 key 不是 `CALayer` 的申明属性，你仍然可以像下面一样设置一个值：

```objc
[theLayer setValue:[NSNumber numberWithInteger:50] forKey:@"someKey"];
```

你也可以试着获取任意 key 的 value。例如，你想获取之前设置的 key path 的值，如下:

```objc
someKeyValue=[theLayer valueForKey:@"someKey"];
```

## Default Value Support
Core Animation 添加一个 convention 到 key value coding，一个类可以给未设置的 key 提供默认值。`CAAnimation` 和 `CALayer` 类使用 `defaultValueForKey:` 类方法来支持这种 convention。

要给一个 key 提供默认值，创建指定 class 的子类，重写 `defaultValueForKey:` 方法。这个方法的实现应该检测 key 参数，并返回合适的值。下面的代码中给出了一个样例实现。

```objc
+ (id)defaultValueForKey:(NSString *)key {
    if ([key isEqualToString:@"masksToBounds"]) {
        return [NSNumber numberWithBool:YES];
    }
    return [super defaultValueForKey:key];
}
```

## Wrapping Conventions
当一个 key 的数据由原子类型或 C 数据结构组成的时候，你必须 wrap that type in an object type before assign it to the layer. Similarly, when accessing that type, you must retrieve an object and then unwrap the appropriate values using the extensions to the appropriate class. 下表列出了通常用的 C 类型和相应的 wrap 方法。

C type | Wrapping Class
--------- | ---------------------
`CGPoint` | `NSValue`
`CGSize` | `NSValue`
`CGRect` | `NSValue`
`CATransform3D` | `NSValue`
`CGAffineTransform` | `NSAffineTransform` (OS X only)

## Key Path Support for Structures
`CAAnimation` 和 `CALayer` 类让你使用 key path 来访问一些数据结构的字段。这个特性是一个动画显式指定数据结构字段的方便方式。你可以和 `setValue:forKeyPath:` `valueForKeyPath:` 方法一起使用来获取和设置这些字段。

### CATransform3D Key Paths
你可以使用 enhanced key path support 从一个包含 `CATransform3D` 数据结构的属性中获取指定的 tranformation value。你可以在字符串 `transform` 或 `sublayerTransform` 后跟上表中的 filed key path 来指定 layer 的 tranfroms key path。例如，要指定围绕 z 轴旋转因子的话，你会使用 key path `transform.rotation.z`

Filed Key Path | Description
------------------- | ----------------
rotation.x | 设置一个 `NSNumber` 对象，值的单位是弧度，围绕 x 轴
rotation.y | 设置一个 `NSNumber` 对象，值的单位是弧度，围绕 y 轴
rotation.z | 设置一个 `NSNumber` 对象，值的单位是弧度，围绕 z 轴
rotation | 设置一个 `NSNumber` 对象，值的单位是弧度，围绕 z 轴。这个字段和 `rotation.z` 一样
scale.x | 设置一个 `NSNumber` 对象，值为 x 轴上的伸缩因子
scale.y | 设置一个 `NSNumber` 对象，值为 y 轴上的伸缩因子
scale.z | 设置一个 `NSNumber` 对象，值为 z 轴上的伸缩因子
scale | 设置一个 `NSNumber` 对象，值为三个轴上的伸缩因子
translation.x | 设置一个 `NSNumber` 对象，值为 x 轴上的移动因子
translation.y | 设置一个 `NSNumber` 对象，值为 y 轴上的移动因子
translation.z | 设置一个 `NSNumber` 对象，值为 z 轴上的移动因子
translation | 设置一个 `NSValue` 对象，这个对象包含一个 `NSSize` 或 `CGSize` 数据类型。这个数据类型表示了 x 轴和 y 轴的移动距离

下面的列子显示了你怎么使用 `setValue:forKeyPath: ` 方法修改一个 layer。例子设置 x 轴上的 tranlation factor 为 10 points，导致 layer 沿着指定的轴移动 10 points。

```objc
[myLayer setValue:[NSNumber numberWithFloat:10.0] forKeyPath:@"transform.translation.x"];
``` 

> **注意** 使用 key path 设置值个使用 Objective-C 属性设置值不一样。你不能使用 property 来设置 transfrom 值。你必须要使用 `setValue:forKeyPath:` 来设置上表中的 key path。

### CGPoint Key Paths
如果一个给定的属性的类型是 `CGPoint`，你可以下表中的字段名来设置或获取这个属性。例如，要改变一个 layer 的 `position` 属性的 `x`，你可以使用 key path `position.x`。

Structure Filed | Description
------------------- | ---------------
x | 点的 x 部分
y | 点的 y 部分

### CGSize Key Paths
如果属性的数据类型是 `CGSize` 类型，你可以将下表中的字段名追加到属性名后面来获取或设置这个值。

Structure Field | Description
--------------------- | ---------------
width | size 的 width 部分
height | size 的 height 部分

### CGRect Key Paths
如果你一个给定的属性值类型是 `CGRect`，你可以将下表中的字段名追加到属性名后面来获取或设置这个值。例如，要改变 layer‘s bounds 的 width 部分，你可以使用 key path `bounds.size.width`。

Structure Field | Description
--------------------- | ----------------
origin | rectangle 的 origin，是一个 `CGPoint`
origin.x | The x component of the rectangle origin.
origin.y | The y component of the rectangle origin.
size | The size of the rectangle as a CGSize.
size.width | The width component of the rectangle size.
size.height |  The height component of the rectangle size.