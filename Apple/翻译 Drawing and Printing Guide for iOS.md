# About Drawing and Printing in iOS
这篇文档会覆盖以下主题：

* 绘制自定义 UI 视图. 自定义 UI 视图允许你绘制那些标准 UI 元素不能轻易绘制的内容. 例如一个绘制程序可能为用户的绘制使用一个自定义视图, 或一个 arcade 游戏使用一个自定义视图来绘制 sprites
* 绘制到 offscreen 位图和 PDF 内容. 无论你是否计划稍后显示图片, 将它们导出到文件, 或打印图片到一个 AirPrint-enabled 打印机, offscreen绘制让你在不干扰用户的工作流的同时做这些. 
* 增加你的应用增加 AirPrint 的支持. iOS 打印系统让你绘制不同的内容来适应页面. 

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/custom-view-and-standard-view.png" width=80% />

## At a Glance**重要**: 一个视图由 `CAEAGLL
iOS 本地的图形系统主要有三种技术：UIKit,  Core Graphics 和 Core Animation. UIKit 提供了视图和一些在这些视图中绘制的高级功能, Core Graphics 提供了额外的在 UIKit 视图中 (底层) 绘制支持, Core Animation 提供了应用 transformations 和动画到 UIKit 视图的能力. Core Animation 同样也负责视图组合 (view compositing)。

## Custom UI Views Allow Greater Drawing Flexibility
这份文档描述了使用本地绘制技术绘制到自定义视图。这些技术包括 Core Graphics 和 UIKit 框架都支持 2D 绘制。

在你考虑使用自定义 UI 视图前，你应该确定是否真需要这么做。本地绘制适合于处理更复杂的 2D 布局需求。然而，因为自定义视图是处理器密集的，你应该限制使用本地绘制技术绘制的总量。

作为自定义绘制的替换方案，一个 iOS 应用可以使用以下技术在屏幕上绘制内容：

* **使用标准的视图**. 标准的视图让你绘制通用的用户交互原语，这些交互原语包括列表，collections，alerts，images，progress bars，tables等等，使用它们不需要显示的绘制任何内容。使用内建的视图不仅保证了 iOS 应用间一致的用户体验，也节约了编程付出。如果内建的视图满足你的需求，你应该读读 *View Programming Guide for iOS*
* **使用 Core Animation layers**. Core Animation 允许你创建复杂的，层状的 2D 视图，并使用它们来做动画和 tranformations. Core Animation 是以下工作的不错选择，给标准的视图做动画，以复杂的方式组合视图来展示视觉的深度，和这篇文档中描述的自定义视图一起使用。更多的信息可以查看 *Core Animation Overview*
* **在一个 GLKit 视图中使用 OpenGL ES 或使用自定义视图**. OpenGL ES 框架提供一系列标准图形库，这个图形库主要是为游戏开发或需要高帧率的应用准备的，如 virtual prototyping apps, mechanical and architectural design apps。这个框架符合 OpenGL ES 2.0 和 OpenGL ES 1.1 规范。关于 OpenGL 绘制的更多信息，可以参见 *OpenGL ES Programming Guide for iOS *.
* **使用 web 内容**. `UIWebView` 允许你在一个 iOS 应用中显示基于 web 的内容。要了解关于在 web view 显示 web 内容的更多信息，可以参见 *Using UIWebView to display select document types* 和 *UIWebView Class Reference*.

依赖于你所创建应用的内容，使用很少的自定义视图或根本不适用是可能的。尽管那种沉浸式的应用通常大量的使用自定义绘制代码，工具和效率应用通常可以使用标准视图和控件来展示它们的内容。

自定义绘制代码的使用应该限制在你显示的内容能够动态的变化。例如，一个画图应用通常需要使用自定义绘制代码来追踪用户的绘制命令，一个 arcade-style 的游戏也许持续的更新屏幕来反映游戏环境的变化。在这些情况下，你应该选择一个适合的绘制技术，创建一个自定义视图类处理时间和合理的更新显示。

另一方面，如果应用的大部分界面是固定的，你可以提前渲染到一个或多个图片中，在运行时使用 `UIImageView` 类显示这些图片。你可以将 image view 和其他的内容层叠起来来构建你的界面。你也可以使用 `UILabel` 来显示可配置的文字，使用按钮或其他控件来提供交互。例如，一个电子版的 board game 可以通过使用很少或不用自定义绘制代码来实现。

因为自定义视图通常是处理器密集型的 (很少用到 GPU)，如果你可以通过使用标准视图来完成你想做的，你应该总是这么做。同样，你应该尽可能少的创建自定义视图，你的自定义视图应该包含你通过其他方式不能绘制的内容。如果你需要组合标准 UI 组件和自定绘制，考虑使用一个 Core Animation layer 来叠加一个自定义视图和一个标准视图以便尽可能少的绘制。

### A Few Key Concepts Underpin Drawing With the Native Technologies
当你使用 UIKit 和 Core Graphics 绘制内容的时候，你应该熟悉视图绘制周期外的一些概念。

* 对于 `drawRect:` 方法，UIKit 创建一个 **graphics context** 供渲染显示用。这个 graphics context 包含绘制系统进行绘制所需的信息，包括 fill 和 stroke 颜色，字体，clipping area 和 line width 等信息。你也可以为位图和 PDF 内容创建并绘制到自定义 graphics context。
* UIKit 有一个 **default coordinate system**，这个坐标系绘制的原点在左上角。正方向是向下和向右。你可以改变大小，你可以通过修改当前的 transformation 矩阵来改变相对于视图或窗口的默认坐标系的大小，方向和位置。
* 在 iOS 中，**logical coordinate space** 以 point 为单位来度量距离，这个坐标空间是不等同于 device coordinate space 的，后者以像素为单位。为了更准确的原因，points 是通过浮点数表示的。

### UIKit, Core Graphics, and Core Animation Give Your App Many Tools For Drawing
UIKit 和 Core Graphics 有许多围绕 graphics context 的互补图形功能，Bézier paths, images，bitmaps，transparency layers，colors，fonts，PDF content，and drawing rectangles and clipping areas. 另外，Core Graphics 提供了与 line attributes, color spaces, pattern colors, gradients, shadings, and image masks 相关的功能. Core Animation 框架使你能够通过操作和展示其他技术创建的内容来创建流畅的动画。

## Apps Can Draw Into Offscreen Bitmaps or PDFs
下面的情况下创建 offscreen 内容很有用：

* offscreen bitmap context 通常被用在缩减待上传的图片，渲染一个图片以便保存到文件，或使用 Core Graphics 来产生复杂的图片以便显示。
* offscreen PDF context 通常用在绘制用户产生的内容，以便打印。

在你创建一个 offscreen context 后，你可以绘制内容到它里面，就像你在自定义视图的 `drawRect:` 中一样绘制。

## Apps Have a Range of Options for Printing Content
自 iOS 4.2 开始，应用可以使用 AirPrint 无线打印内容到打印机。当完成一个打印工作时，你有三种方式来提供 UIKit 内容打印：

* 可以提供框架一个或多个直接支持打印的对象，这样的对象需要应用最少的牵连。包含 image data 或 PDF 内容的 `NSData`，`NSURL`, `UIImage`，`ALAsset` 类实例属于这样的对象。
* 可以指定一个 print formatter 到指定的 print job。一个 print formatter 是一个可以在多个页面布局某种内容的对象。
* 可以指定一个 page renderer 给指定 print job。一个 page renderer 通常是 `UIPrintPageRenderer` 的自定义子类，部分或全部的绘制可被打印的内容。一个 page renderer 可以使用一个或多个 print formatter 来帮助它绘制和格式化可打印的内容。

## It’s Easy to Update Your App for High-Resolution Screens
一些 iOS 设备有高分辨率的特点，所以你的应用必须准备好在这样的设备上和低分比率的设备上运行。iOS 会处理大部分与不同分辨率相关的工作，但是你的应用必须处理剩余的。你的任务包括提供特定名称的高分辨率图片，根据当前的 scale 因子来修改你的 layer 和 image 相关的代码。

## See Also
关于打印的完整例子，可以参见 *PrintPhoto*, *Sample Print Page Renderer*,  and *UIKit Printing with UIPrintInteractionController and UIViewPrintFormatter* 样例代码。

---
# iOS Drawing Concepts
高质量的图形是你应用界面重要的一部分。提供高质量图形不仅仅使你的应用好看，也让你的应用看起来是系统其它部分的自然延伸。iOS 提供了两种路径来创建系统中的高质量图形 : OpenGL 或 使用 Quartz, Core Animation, UIKit 来本地渲染。这篇文档描述本地渲染. (想要了解 OpenGL 绘制的话，请参见 *OpenGL ES Programming Guide for iOS*)

Quartz 是主要的绘制接口，提供了基于 base 的绘制，anti-aliased 渲染，gradient fill patterns，images，colors，coordinate-space transformations，PDF document creation，display，parsing。UIKit 为 line art，Quartz images, color 操作提供了 Objective-C 的封装。Core Animation 提供动画显示 UIKit 视图属性变化的潜在支持，可被用来实现自定义动画。

这章提供了 iOS 应用绘制过程的概览，顺便提及了每种绘制技术中的特定绘制技巧。同时你也会看到一些关于怎么为 iOS 平台优化你的绘制代码的 tips 和 guidance。

> **重要**: 并不是所有的 UIKit 类是线程安全的。在非主线程绘制相关的操作时主意检查文档

## The UIKit Graphics System
在 iOS 中，所有绘制到屏幕的操作——不管它是设计到 OpenGL，Quartz，UIKit，或 Core Animation——发生在一个 `UIView` 类或其子类实例的范围内。Views 定义了屏幕上绘制所发生的区域。如果你使用系统提供的视图，这种绘制会自动帮你处理。如果你自定义视图，你必须自己提供绘制代码。如果你使用 Quartz，Core Animation，UIKit 来绘制，你会使用到下面描述的绘制概念。

除了直接绘制到屏幕上外，UIKit 同样也允许你绘制到 offscreen 位图和 PDF graphics context 中。当你绘制到一个 offscreen context 中时，你不是会绘制到一个 view，这意味着类似于 drawing cycle 的概念并不适用。 (除非你获取这个图片，并绘制它到屏幕上)

### The View Drawing Cycle
对于 UIView 的子类基本的绘制模型涉及到按需的更新内容。然而`UIView` 类使得更新过程简单和高效，这是通过收集你做的更新请求，并在核实的时候把它们传送给绘制代码的。

当一个视图初次显示或当视图的一部分需要重绘，iOS 通过调用视图的 `drawRect:` 方法来要求视图绘制它的内容。

有好几中行为会导致一个视图更新：

* 移动或移除部分遮挡你视图的视图
* 使一个之前隐藏的视图再次显示，通过设置它的 `hidden` 属性为 `NO`
* 滑动一个视图出屏幕在滑回屏幕上
* 显式调用视图的 `setNeedsDisplay` 和 `setNeedsDisplayInRect:` 方法

系统视图会自动重绘。对于自定义视图，你必须重写 `drawRect:` 方法并在里面进行你所有的绘制。在你的 `drawRect:` 方法内部，使用本地绘制技术来绘制 shape, text, images, gradients, 或其它你想要的可是内容。你的视图初次可见的时候，iOS 传递一个 rectangle 到视图的 `drawRect:` 方法，这个 rectangle 包含视图的整个可视区域。在接下来的调用中，rectangle 只包含视图需要重绘的那部分。为了最大的性能，你只应该重绘被影响的内容。

在你调用 `drawRect:` 方法后， 视图标识自己为已更新，等待新行为到达并引发另一个 update cycle。如果视图显示静态内容，你所有需要做的是响应视图的滑动引起的可见性变化和其他视图的显示。

如果你想要改变视图的内容，你必须告诉你的视图来重绘它的内容。要这么做的话，调用 `setNeedsDisplay` 或 `setNeedsDisplayInRect:` 方法来触发一次更新。例如，如果你一秒中多次更新视图内容，你也许想要设置一个 timer 来更新你的视图。你也许会更新视图来来响应用户交互或在你的视图中创建新内容。

> **重要**: 你自己不要调用视图的 `drawRect:` 方法。这个方法应该只被 iOS 自带代码在屏幕重绘的时候调用。在其它的时候，没有 graphics context 存在，所以绘制是不可能的。

### Coordinate Systems and Drawing in iOS
当 iOS 中的一个应用绘制些东西的时候，它必须在坐标系定义的二维空间中定位被绘制的内容。这个概念初看起来可能很直接，但并不是。iOS 中的应用在绘制的时候有时需要处理不同的坐标系。

在 iOS 中，所有的绘制都发生在一个 graphics context。概念上，一个 graphics context 是一个描述绘制应该在哪里和怎样发生的对象，它包含了基本绘制属性如绘制时使用的颜色，clipping area，line width 和样式信息，字体信息，compositing 选项等等。

另外像下图中显示的那样，每个 graphics context 有一个坐标系统。更准确的说，每个 graphics context 有三个坐标系：

* The drawing (user) coordinate system. 这个坐标系是绘制时你用来发送绘制命令的。
* The view coordinate system (base space). 这个坐标系是相对于视图的一个固定坐标系。
* The (physical) device coordinate system. 这个坐标系代表屏幕上的物理屏幕。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/relationship-between-different-coordinate-system.png" width=80% />

iOS 的绘制框架创建 graphics context 供绘制到指定的目的地——屏幕，bitmaps，PDF content 等等——这些 graphics context 为这个目的地建立初始的 drawing coordinate system。这个初始的 drawing coordinate system 被称为 **default coordinate system**，并与视图的潜在坐标系是 1:1 的关系。

每个视图有一个 *current transformation matrix (CTM)*，一个数学矩阵将当前的 drawing coordinate system 映射到 view coordinate system。应用可以修改这个矩阵来改变将来的绘制操作。

iOS 的每个绘制框架会基于当前的 graphics context 建立一个默认的 coordinate system. 在 iOS 中，有两种类型的坐标系：

* 一个左上角原点坐标系 (upper-left-origin coordinate system, ULO)，在这个坐标系里面绘制操作的原点在绘制区域的左上角，正值方向是向下和向右。UIKit 和 Core Animation 框架使用的默认坐标系统是基于 ULO 的。
* 一个左下角原点坐标系 (lower-left-origin coordinate system, LLO)，在这个坐标系绘制操作的原点在绘制区域的左下方，正值方向是向上和向右。Core Graphics 使用的默认坐标系是基于 LLO 的。

这两个坐标系如下图所示：

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/default-coordinate-system-in-iOS.png" width=80% />

> **注意**: OS X 系统的默认坐标系统的是基于 LLO 的。尽管 Core Graphics 和 AppKit 框架中的绘制函数和方法是完全适合默认的坐标系统，AppKit 对于翻转绘制的坐标系提欧提供了程序上的支持，以让你有一个 upper-left 原点。

在你调用视图的 `drawRect:` 方法前，UIKit 通过给绘制操作创建一个 graphics context，为绘制到屏幕提供了一个默认的坐标系。在视图的 `drawRect:` 方法里面，一个应用可以设置 graphics state 参数 (such as fill color) ，并且不需要显式的引用 graphics context 就可以绘制到 graphics context。隐式的 graphics context 是 ULO 的默认坐标系。

### Points Versus Pixels
在 iOS 中，在你的绘制代码中指定的坐标系和潜在设备的像素坐标系之间有个区别。当你使用如 Quartz，UIKit，Core Animation 之类的本地绘制技术时，drawing coordinate space 和视图的 coordinate space 都是 *logical coordinate spaces*，距离是以 *points* 为单位的。这些 logical coordinate systems 和系统框架所使用来管理屏幕像素的设备 coordinate space 是解耦的。

系统框架自动将视图坐标系中的 points 映射到设备坐标系中的像素，但是这种映射并不总是一对一的。这种行为导致了一个你永远要记住的事实：

> **一个 point 并不总是对应到屏幕上的一个像素**

使用 point (和 logical coordinate system) 的目的是提供一个一致的输出尺寸，这个尺寸是设备无关的。对于大部分情况，point 的真实大小并不重要。points 的目的是提供一个相对一致的大小，你可以在你的代码中指定视图的大小和位置并渲染内容。points 实际是怎么映射到像素的是系统框架处理的一件具体事情。例如，在一个高分辨率的设备上，一个 one point 宽的线可能在实际的设备上是两像素宽。结果是你绘制相同的内容到两个相似的设备上，其中一台是高分辨率屏，这时两个设备上显示的内容是相同大小的。

> 在 PDF 渲染和打印的上下文下，Core Graphics 使用工业标准一个点映射到 one inch 的 1/72。

在 iOS 中，`UIScreen`, `UIView`, `UIImage`, `CALayer` 类提供了属性来获取 (某些时候还能设置) 一个 *scale factor*，这个属性描述了对于特定对象 point 和 pixel 的关系。例如，对于 UIKit 视图有个 `contentScaleFactor` 属性。在一个标准的分辨率屏幕上，scale factor 通常是 1.0. 在一个高分辨率的屏幕上，scale factor 通常是 2.0。在未来，其它的 scale factor 也是可能的。

本地绘制技术，如 Core Graphics，会为你考虑当前的 scale factor。例如，如果一个你的视图实现了 `drawRect:` 方法，UIKit 自定设置视图的 scale factor 为屏幕的 scale factor。另外，UIKit 自动修改任何用来会绘制的 graphics context 的 current transformation matrix，并将视图的 scale factor 考虑在内。因此，任何你在 `drawRect:` 方法中绘制的内容为潜在的屏幕很好的改变大小。

因为这种自动的映射，当编写绘制代码时，pixel 通常不重要。然而，有时候你可能需要根据 points 是怎么映射到 pixels 来改变应用的绘制行为——例如，下载高分辨率的图片到高分辨率的设备上，避免伸缩图片当在低分辨率设备上。

当你在 iOS 中绘制内容到屏幕上时，graphics subsystem 会使用一个叫 `antialiasing` 的技术来在低分辨率屏上模仿一个高分辨率的图片。解释这种技术的最好方式是通过示例。当你在一个纯白的背景上绘制一个竖直的黑线时，如果这条线刚好落在一个像素里，它会显示为在白色区域里的一系列黑色像素。如果它刚好在两个像素上的话，这个两个像素会显示为灰色。如下图。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/one_point_line_centered_at_a_whole_numbered_point_value.png" width=80% />

整数点定义的位置会落入像素之间。例如，如果你绘制一个从 (1.0, 1.0) 到 (1.0, 10.0) 的 one-pixel-wide 竖线，你会看到一个模糊的灰色线。如果你绘制绘制一个 two-pixel-wide 的竖线，你会看到一个纯黑的线因为这条线完全覆盖两个像素。规律是，奇数物理像素宽的线看上去要比偶数物理像素宽的线更柔和。除非你显式的调整它们的位置来让它们覆盖更多像素。

在决定多少像素被一个 one-point-wide 的线覆盖时，scale factor 就发挥作用了。

在一个低分辨率的显示设备上，one-point-wide 线只有一像素宽。当你绘制一个一个 one-point-wide 的水平或竖直线时为了防止 antialiasing，如果这条线的宽是奇数，你必须往任一方向移动 0.5 point。如果这条线的款是偶数，为了避免模糊的效果，你一定不要这么做！

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/appearance_of_one-point-wide_lines_on_standard_and_retina_displays.png" width=80% />

在一个高分辨率的显示设备上，一条 one-point-wide 的线没有防锯齿效果是因为它完全占有两个像素宽。要绘制一条一像素宽的线，你需要是它的宽度为 0.5 point 宽，并 offset 它的位置 0.25 points。两种不同的屏幕的对比如上图。

当然，基于一个 scale factor 来改变绘制特点也许有意外的结果。一个 1-pixel-wide 的线可能在一些设备上看起来不错，但是在一个高分辨率的设备上看起来过细以致很难清除的辨别。当然由你来决定是否要做类似的改变。

### Obtaining Graphics Contexts
大部分时候，graphics context 会为你配置。每个视图对象自动创建一个 graphics context，以便你的自定义方法 `drawRect:` 被调用的时候可以立即开始绘制内容。潜在的 `UIView` 类为绘制环境创建一个 graphics context (一个 `CGContextRef` opaque type) ，这个过程作为配置的一部分。

如果你想要视图外的某些地方绘制的话 (例如把一系列的绘制操作输出到一个 PDF 或一个位图文件中)，或如果你需要调用需要一个 context 对象的 Core Graphics 函数，你必须采取额外的步骤来获得一个 graphics context 对象。下面的部分描述了怎么做。

关于 Graphics contexts，修改 graphics state 信息，和使用 graphics context 的更多信息，可以参见 *Quartz 2D Programming Guide*。关于和 graphics context 一起使用的函数列表可以参见 *CGContext Reference*, *CGBitmapContext Reference*, 和 *CGPDFContext Reference*。   

#### Drawing to the Screen
如果你使用 Core Graphics 函数绘制到一个视图的话，要么是在 `drawRect:` 方法或其他地方，你都将需要一个 graphics context 来绘制。(这些函数中许多的第一个参数都是一个 `CGContextRef` 对象) 你可以调用函数 `UIGraphicsGetCurrentContext` 来获取一个隐式为方法 `drawRect:` 创建的 graphics context 的显式版本。因为是同一个 graphics context，绘制函数也应该引用一个默认的 ULO 坐标系统。

如果你想要使用 Core Graphics 函数来在一个 UIKit 视图里绘制的话，你应该使用 UIKit 的 ULO 坐标系来进行绘制操作。或者，应用一个 flip transform 到 CTM，然后使用 Core Graphics 本地的 LLO 坐标系绘制到 UIKit 视图上。

`UIGraphicsGetCurrentContext` 函数返回目前有效的 graphics context。例如，如果你创建一个 PDF context，然后调用 `UIGraphicsGetCurrentContext`，你将获得这个 PDF context。你必须使用 `UIGraphicsGetCurrentContext` 返回的 graphics context，如果你使用 Core Graphics 来绘制到一个视图的话。

> `UIPrintPageRenderer` 类申明了一些绘制可打印内容的方法。某种程度上和 `drawRect:` 相似，UIKit 会插入一个隐式 graphics context 在这些方法的实现中。这个 graphics context 建立一个 ULO 的默认坐标系。

#### Drawing to Bitmap Contexts and PDF Contexts
UIKit 提供函数在一个位图 graphics context 中渲染图片，和通过在一个 PDF graphics context 中绘制来产生 PDF 内容。这些方案都要求你首先调用一个函数来创建一个 graphics context —— 一个 bitmap context 或一个 PDF context。这个返回的对象作为当前的 graphics context 供接下来的绘制和状态设置调用。当你完成在 context 中的绘制，你可以调用另一个函数来关闭这个 context。

UIKit 提供的 bitmap context 和 PDF context 都建立了一个 ULO 默认坐标系。Core Graphics 有相应的方法来在一个 bitmap graphics context 和在一个 PDF graphics context 中绘制。一个应用通过 Core Graphics 直接创建的 context 会建立一个 LLO 默认坐标系。

> **注意**: 在 iOS 中，对于绘制到 bitmap context 和 PDF context，推荐你使用 UIKit 函数。然而，如果你确实使用的是 Core Graphics 的话并想要显示渲染的结果，你将必须调整你的代码来抵消这些不同坐标系的区别。

### Color and Color Spaces
iOS 支持在 Quartz 中可用的全范围 color spaces；然而，大部分应该应该只需要 RGB 颜色空间。因为 iOS 被设计跑在嵌入式设备上，RGB 颜色空间是适用的。

`UIColor` 对象提供了便利的方法来指定颜色值，你可以使用 RGB，HSB，和灰度值。当你使用这种方法创建颜色的时候，你不需要指定颜色空间。`UIColor` 对象会自动帮你决定。

你也可以使用 `Core Graphics` 中的 `CGContextSetRGBStrokeColor` 和 `CGContextSetRGBFillColor` 函数创建和设置颜色。尽管 Core Graphics 框架包含支持使用其他 color space 创建颜色，和创建自定义颜色空间，在你的绘制代码使用这些颜色是不推荐。你的绘制代码应该总是使用 RGB 颜色。

## Drawing with Quartz and UIKit
Quartz 是 iOS 中本地绘制技术的泛称。Quartz 的核心是 Core Graphics 框架，是你使用来绘制内容的主要接口。这个框架提供了数据类型和函数来操作下面的内容：

* Graphics contexts
* Paths
* Images and bitmaps
* Transparency layers
* Colors, pattern colors, and color spaces
* Gradients and shadings
* Fonts
* PDF content

UIKit 基于 Quartz 的基本特性为图形相关操作提供了一系列更专注的类。UIKit 图形类并不打算作为一个图形工具的全覆盖—— Core Graphics 已经做了这些。相反，这些类为其它的 UIKit 类提供了绘制基础。UIKit 的图形支持包括以下类和函数：

* `UIImage`，实现了显示图片的 immutable class
* `UIColor`，为设备颜色提供基本的支持
* `UIFont`，为需要它的类提供了字体信息
* `UIScreen`，提供了关于屏幕的基本信息
* `UIBezierPath`，使你的应用可以绘制直线，弧线，椭圆和其他现状
* 产生一个 `UIImage` 对象的 `JPEG` 或 `PNG` 表示的函数
* 绘制到一个 bitmap graphics context 的函数
* 通过绘制到 PDF graphics context 产生 PDF 数据的函数
* 绘制矩形和 clipping the drawing area 的函数
* 改变和获取当前 graphics context 的函数

关于组成 UIKit 的类和方法的更多信息可以查看 *UIKit Framework Reference*。关于组成 Core Graphics 框架的 opaque type 和函数，可以查看 *Core Graphics Framework Reference*

### Configuring the Graphics Context
在调用 `drawRect:` 方法之前，视图对象创建一个 graphics context 并设置它为当前的 context。这个 context 只在方法 `drawRect:` 生命周期内存在。你可以通过调用 `UIGraphicsGetCurrentContext` 函数来获取这个 graphics context 的指针。这个函数会返回一个 `CGContextRef` 类型的引用，你传递这个引用给 Core Graphics 函数来修改当前 graphics states。下表列出了设置 graphics state 不同方面的主要函数。要查看完整的列表可以参见 *CGContext Reference*。下表也列出了存在的 UIKit 替换选择。

Graphics state | Core Graphics functions | UIKit alternatives
-------------------- | ---------------------------------- | -----------------------
Current transformation matrix | `CGContextRotateCTM`<br />`CGContextScaleCTM`<br />`CGContextTranslateCTM`<br />`CGContextConcatCTM` | None
Clipping area | `CGContextClipToRect` | `UIRectClipFuncation` 
Line: Width, join,cap, dash, miter limit | `CGContextSetLineWidth`<br />`CGContextSetLineJoin`<br />`CGContextSetLineCap`<br />`CGContextSetLineDash`<br />`CGContextSetMiterLimit` | None
Accuracy of curve estimation | `CGContextSetFlatness` | None
Anti-aliasing setting | `CGContextSetAllowsAntialiasing` | None
Color: Fill and stroke settings | `CGContextSetRGBFillColor`<br />`CGContextSetRGBStrokeColor ` | `UIColor` 类
Alpha global value | `CGContextSetAlpha` | None
Rendering intent | `CGContextSetRenderingIntent` | None
Color space: Fill and Stroke settings | `CGContextSetFillColorSpace`<br />`CGContextSetStrokeColorSpace` | `UIColor` 类
Text: Font, font size, character spacing, text drawing mode | `CGContextSetFont`<br />`CGContextSetFontSize`<br />`CGContextSetCharacterSpacing` | `UIFont` 类
Blend mode | `CGContextSetBlendMode` | `UIImage` 类和不同的绘制函数让你指定使用哪种 blend mode

Graphics context 一个包含保存的 graphics states 的栈结构。当 Quartz 创建一个 graphics context 的时候，栈是空的。使用 `CGContextSaveGState` 函数会将当前的 graphics context 压栈。因此你对 graphics context 的修改会影响接下来的绘制操作，但是不会影响栈上的拷贝。当你结束修改的时候，你通过使用 `CGContextRestoreGState` 可以将之前保存的 graphic state 出栈从而返回到之前的 graphics state。像这样压栈并出栈是一种返回之前状态的快速方式，也消除了单个 undo 每个 state change 的需要。同样这也是恢复某些状态的唯一方式，如 clipping path 要恢复到之前的设置。

关于 graphics context 的更多信息以及怎么使用它们配置绘制环境，可以参见 *Quartz 2D Programming Guide* 中的 *Graphics Context*。

### Creating and Drawing Paths
一个 path 是一个基于向量的 shape，由一系列 lines 和 Bézier curves 组成。UIKit 包含 `UIRectFrame` 和 `UIRectFill` (这是一部分) 等函数，它们可以在视图中绘制简单的 path，如矩形。 Core Graphics 同样包含创建如矩形和椭圆的简单 path 的便利函数。

关于更复杂的 path，你必须使用 UIKit 的 `UIBezierPath` 类创建，或使用 Core Graphics 框架中操作 `CGPathRef`  类型的函数。尽管你这两者任一技术构建一个 path 都不需要一个 graphics context，这个 path 上的 points 仍然必须引用当前的 graphics context，你仍然需要一个 graphics context 来真实的渲染这个 path。

当绘制一个 path 的时候，你必须有一个 current context 被设置好。这个 context 可以是自定义视图的 context (在方法 `drawRect:` 中)，一个 bitmap context，或一个 PDF context。坐标系决定了 path 怎么样被渲染。`UIBezierPath` 会假设使用一个 ULO 坐标系。因此，如果你的视图是翻转的 (使用 LLO 坐标系)，渲染的结果形状可能跟期望的结果不同。为了最好的结果，你总是应该使用相对于当前 graphics context 坐标系坐标原点的坐标来渲染。

> **注意**: Arcs 除遵守上面的 "规则" (指上面最后一句相对于坐标原点这句) 外，还需要额外的工作。  如果你在 ULO 坐标系中使用 Core Graphics 函数创建一个 path，然后在视图中渲染这个 path，一个 arc 指向的方向是不同的。(这句原文 "the direction an arc “points” is different")，关于这个主题的更多信息可以参见 *Side Effects of Drawing with Different Coordinate Systems*

在 iOS 中创建 paths，推荐你使用 `UIBezierPath` 而不是 `CGPath` 函数，除非你需要 Core Graphics 才提供的功能，如添加椭圆到 path。关于在 UIKit 中创建和渲染 path 的信息，参见这边文档的 *Drawing Shapes Using Bézier Paths*。

关于使用 `UIBezierPath` 绘制 path 的更多信息，可以参见这篇文档 *Drawing Shapes Using Bézier Paths*。关于怎么使用 Core Graphics 绘制，包括制定复杂 path 元素的点的更多信息，可以查看 *Quartz 2D Programming Guide*。关于使用函数创建 paths 的更多信息，可以参见 *CGContext Reference* 和 *CGPath Reference*。

### Creating Patterns, Gradients, and Shadings
Core Graphics 框架包含额外的函数来创建 patterns，gradients，和 shadings。你使用这些类型来创建非单一的混合色并使用这些颜色来填充你创建的 path。Patterns 通过重复图片或内容来创建。Gradients 和 Shadings 提供了不同的方式来创建颜色到颜色的平滑过渡。

创建和使用 patterns，gradients，和 shadings 的具体信息在 *Quartz 2D Programming Guide* 中都被覆盖了。

### Customizing the Coordinate Space
默认情况下，UIKit 创建一个 current transformation matrix 来将 points 映射到 pixels。尽管你可以在不修改这个 matrix 的情况下做所有你的绘制，但有时这么做的话更方便。

当视图的 `drawRect:` 方法被初次调用的时候，CTM 被配置成系统的坐标原点和你的视图的原点相配，X 轴的正方向向右，Y 轴的正方向向下。然而，你可以通过添加 scaling，rotation，和 translation 变化到它来改变这个 CTM，从而改变默认坐标系相对于视图或 window 的潜在坐标系的大小，方向，和位置。

#### Using Coordinate Transforms to Improve Drawing Performance
对于在一个视图中绘制修改 CTM 是一个标准技术，因为它允许你重用 paths，这样很可能减少绘制时所需的总计算量。例如，如果你从 (20, 20) 点开始绘制一个正方形，你可以绘制一个移到 (20, 20) 的点，然后绘制所需的线集合来完成正方形。然而，如果你稍后决定移动这个正方形到点 (10, 10)，你将需要从新的起点重绘这个正方形。因为创建 path 是相对昂贵的操作，更可取的方案是创建一个起点在 (0, 0) 的正方形，然后修改它的 CTM 以让它期望的原点上绘制。

在 Core Graphics 框架中，有两种方式修改 CTM。你可以直接使用 *CGContext Reference* 中定义的 CTM 操作函数修改 CTM。你也可以创建一个 `CGAffineTransform` 结构，应用任何你想要的 transformation，然后拼接这个 transform 到这个 CTM 上。使用一个 affine transform 允许你打包 transformations，然后一次应用它们到 CTM 上。你可以计算和反转 affine transforms， 具体查看 *Quartz 2D Programming Guide* 和 *CGAffineTransform Reference*

#### Flipping the Default Coordinate System
在 UIKit 绘制中的翻转 (flipping) 修改了支撑视图的 `CALayer`，从而将一个 LLO 坐标系的绘制环境与 UIKit 的默认坐标系对齐。如果你只使用函数和方法来绘制，你不需要翻转 CTM。然而，如果混合使用 Core Graphics 或 Image I/O 函数和 UIKit 调用，翻转 CTM 可能是需要的。

明确的说，如果通过直接调用 Core Graphics 绘制一个图片或 PDF 文档，这个对象在 view 的 context 中被上下颠倒的渲染。你必须翻转 CTM 来正确的显示图片和页面。

要翻转一个绘制到 Core Graphics context 的对象，从而让它正确的显示到一个 UIKit 视图上。你必须分两步修改 CTM。你移动原点到绘制区域的左上角，然后你应用一个 scale translation，将 y 坐标乘以 -1。代码如下：

```objc
CGContextSaveGState(graphicsContext);
CGContextTranslateCTM(graphicsContext, 0.0, imageHeight);
CGContextScaleCTM(graphicsContext, 1.0, -1.0);
CGContextDrawImage(graphicsContext, image, CGRectMake(0, 0, imageWidth, imageHeight));
CGContextRestoreGState(graphicsContext);
```

如果你使用一个 Core Graphics image 对象创建一个 `UIImage` 对象，UIKit 会为你 flip transform。每个 `UIImage` 对象底层都有一个 `CGImageRef` opaque type。你可以通过 `CGImage` 属性来获取一个 Core Graphics 对象，并使用这个对象来做一些工作。(Core Graphics 有一些 image-related facilities 在 UIKit 中不可用) 当你结束使用后，你可以从修改过的 `CGImageRef` 对象中来重新创建 `UIImage` 对象。

> **注意**: 你可以使用 Core Graphics 函数 `CGContextDrawImage` 来绘制图片到任何目的地。这个函数有两个参数，一个 graphics context 和一个定义图片大小和平面中绘制位置的 rectangle。当你使用 `CGContextDrawImage` 绘制一个图片的时候，如果你不调整当前的坐标系统为 LLO 方向的话，在 UIKit 视图中图片会看上去倒立显示。另外，传递给这个 rectangle 的原点是相对于函数调用时当前正在使用的系统坐标系而言的。

#### Side Effects of Drawing with Different Coordinate Systems
当你使用一种绘制技术的默认坐标系来绘制一个对象，然后在另一个坐标系的 graphics context 中渲染时一些渲染时的怪事就会发生。你也许需要调节你的代码来消除这些副作用。

##### Arcs and Rotations
如果你使用 `CGContextAddArc` 和 `CGPathAddArc` 来绘制一个 path 并假设一个 LLO 坐标系统，那么你需要 flip the CTM 来正确的在一个 UIKit view 中渲染这个弧线。然而，如果你使用相同的函数在一个 ULO 坐标系中创建一个弧线然后在 UIKit 视图中渲染这个弧线的话，你会注意到这个弧线是原来的另一个版本。弧线的结束点现在指向的方向和你使用 `UIBezierPath` 创建的弧线的结束点所指的方向是相反的。例如，一个向下指向的箭头现在指向向上 (如下图)。弧线弯曲的方向也是相反的。你必须为 ULO 坐标系改变 Core Graphics 绘制弧线的方向。这个方向是由这些函数的 `startAngle` 和 `endAngle` 决定的。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/arc_rendering_in_core_graphics_versus_uikit.png" width=80% />

如果你旋转一个对象 (例如，通过调用`CGContextRotateCTM`)，你可以观察到相同的镜面效果。如果你在一个 ULO 坐标系中使用 Core Graphics 来旋转一个对象，这个对象在 UIKit 中渲染时它的方向是相反的。你必须考虑在你的代码中考虑到旋转方向的不同。使用 `CGContextRotateCTM`，你通过翻转 `angle` 参数的符号来做这件事。

##### Shadows
一个对象的 shadow 的方向有它的 offset 值来指定，这个 offset 的意义是一个绘制框架的 convention。在 UIKit 中，offset 的 x 和 y 是正值的话，使 shadow 在对象的右下方。在 Core Graphics 中，offset 的 x 和 y 使得 shadow 在对象的右上。Flipping the CTM 来对齐 UIKit 的对象与默认坐标系并不影响这个对象的 shadow，所以一个 shadow 会正确的紧随它的对象。为了正确的追随它的对象，你必须根据当前的坐标系正确的修改 offset。

## Applying Core Animation Effects
Core Animation 是一个 Objective-C 框架，提供了快速方便创建流畅、实时动画的基础架构。从 Core Animation 自身并不提供创建 shapes，images，或其它类型的内容的角度来看，Core Animation 自身并不是一个绘制技术。相反，它是一个操作和展示你使用其他技术创建的内容的技术。

iOS 中的大部分应用都在某种程度上受益于 Core Animation。动画提供了正在发生内容的反馈。例如，当用户在设置应用的中导航时，屏幕基于用户是进一步查看更深层次的内容还是退回上层内容来滑入滑出视图。这种类型的反馈很重要，给用户提供了上下文信息。同样也加强了应用的视觉信息。

大部分时候，你不需要太费经就可以获得 Core Animation 的益处。例如，`UIView` 的一些属性 (包含 view 的 `frame`, `center`, `color`, `opacity`，这些只是部分) 可被配置成当它们的值发生改变的时候触发动画。你必须做些工作让 UIKit 知道你想要这些动画进行，但是动画会自动被创建和运行。更多关于怎么触发视图内置动画的更多信息，可以查看 *UIView Class Reference* 中的 Animating Views。

当你不再需要一般的动画的时候，你必须与 Core Animation 类和方法更直接的交互。下面的部分提供了关于 Core Animation 的一些信息，并展示了怎么使用它里面的类和方法创建典型的动画。关于 Core Animation 的更多信息，可以参见 *Core Animation Programming Guide*。

### About Layers
Core Animation 中的核心技术是 layer 对象。Layer 是轻量级的对象，性质上跟视图较像，但是实际上它们是封装了 geometry，timing，和你想要展示的内容的一些视觉属性的模型对象 (model objects)。layer 的内容有以下三种方式提供：

* 你可以指定一个 `CGImageRef` 对象给 layer 对象的 `contents` 属性
* 你可以给 layer 对象设置一个 delegate，并让这个 delegate 来处理绘制
* 你可以继承 CALayer 并覆盖它的一个 display 方法

当你操作一个 layer 对象的属性时，你实际上操作的是 model-level 的数据，这些数据决定了相关的内容是怎么被显示的。内容的实际渲染是在你的代码之外被处理的，并被极大的优化来加快这个过程。你所需要做的是设置 layer 的内容，配置它的动画属性，并让 Core Animation 来接管。

关于 layers 和怎么使用它们的更多信息，你可以参见 *Core Animation Programming Guide*

### About Animations
当需要给 layers 做动画的时候，Core Animation 使用单独的动画对象来控制动画的计时和行为。`CAAnimation` 类和它的子类提供了不同类型的动画行为，你可以在你的代码中使用这些动画。你可以创建将属性从一个值移动到另一个值的简单动画，你可以创建复杂的 keyframe 动画。

Core Animation 同样让你组合多个动画到一起成为一个单元，称为一个 transaction。`CATransaction` 对象以一个单元管理一组动画。你可以使用这个类的方法来设置动画的时长。

关于怎么创建自定义动画的样例，可以参见 *Animation Types and Timing Programming Guide*

### Accounting for Scale Factors in Core Animation Layers
使用 Core Animation layers 直接提供内容的应用也许需要调整它们的绘制来将 scale factors 考虑在内。通常，当你在视图的 `drawRect:` 绘制时，或在 layer's delegate 方法 `drawLayer:inContext:` 中绘制的时候，系统会根据 scale factor 来调整 graphics context。然而，知道或改变 scale factor 在视图做以下事情时仍然是需要的：

* 使用不同的 scale factor 创建额外的 Core Animation Layers 并组合它们到视图的内容中去。
* 直接设置一个 Core Animation layer 的 `contents` 属性

Core Animation 的 compositing engine 会检查每个 layer 的 `contentsScale` 属性来决定 layer 的内容在组合的时候是否需要 scaled。如果你的应用创建的 layer 没有关联的 view，每个 layer 的 scale facotr 初始值为 1.0。如果你不改变这个 scale factor，并且如果接下来你绘制 layer 到一个高分辨率的屏幕上，layer 的内容会自动的被拉伸以抵消这种 scale factor 上的不同。如果你不想要 layer 的内容被拉伸，你可以改变 layer 的 scale facotr 为 2.0，通过设置 layer 的 `contentsScale` 属性，但是如果你这样不提供高分辨率的内容的话，你现有的内容可能看上起要比你期望的小。要修复这个问题的话，你需要给这个 layer 提供高分比率的内容。

> **重要**: `contentsGravity` 属性在决定标准分辨率的 layer 内容在一个高分辨率的屏幕上是否拉伸中扮演了一个角色。这个属性的默认值是 `kCAGravityResize`，会导致 layer 的内容被缩放来适应 layer 的 bounds。改变这个属性为一个 noresizing 选项消除了自动伸缩的行为。在这种情况下，你也许需要调整你的内容或 scale factor。

调整 layer 的内容是你直接设置 layer 的 `contents` 属性时适应不同的 scale factor 的最好方案。Quartz 图片没有 scale factor 的概念，因此是直接与像素工作的。因此，在创建你计划使用来作为 layer 的内容的 `CGImageRef` 对象之前，检查 scale factor 并响应的调整图片的大小。具体的说，从你的 app bundle 中加载合适大小的图片或使用 `UIGraphicsBeginImageContextWithOptions` 函数创建一张 scale factor 与 layer 的 scale factor 匹配的图片。如果你不创建一个高分辨率图片，现有的位图可能像之前讨论的那样被缩放。

关于怎么指定和加载图片，可参见本文的 *Loading Images into Your App*。关于怎么创建高分辨率的图片，可以参见本文的 *Drawing to Bitmap Contexts and PDF Contexts*。

-----
# Drawing Shapes Using Bézier Paths
iOS 3.2 起，你可以使用 `UIBezierPath` 类来创建基于向量的 path。`UIBezierPath` 类是一个 Core Graphics 框架中 path 相关特性的一个 Objective-C 封装。你可以使用这个类来定义简单的 shapes，如椭圆和矩形，那些包含多个直线和曲线段的复杂 shape 也可以。

你可以使用 path 对象来绘制应用 UI 接口的形状。你可以绘制 path 的轮廓，填充它包围的空间，或同时填充空间和轮廓。你可以使用 path 来定义当前 graphics context 的 clipping path，clipping path 会影响接下来在 context 中绘制操作。

## Bézier Path Basics
一个 `UIBezierPath` 对象是一个 `CGPathRef` 数据类型的包装。*Paths* 是基于直线和曲线段构建的基于向量的形状。你可以使用直线段来创建矩形和多边形，你可以使用曲线来创建弧线，圆，和复杂的曲线形状。不论直线段还是曲线段都是由一个或多个点 (在当前的坐标系中) 和一个定义点是怎么解释的绘制命令构成。

每个连接的直线或曲线段的集合组成了一个 *subpath*。subpath 的一条线段或曲线段的结束点定义了下一个的开始。一个单一的 `UIBezierPath` 对象可能包含一个或多个 subpath 来定义真个 path，这些 subpath 由 `moveToPoint:` 命令分隔，这个命令实质上是提提起画笔并移动它到一个新位置。

构建和使用一个 path 对象的过程是分开的。构建这个 path 是第一个过程，涉及到以下步骤：

1. 创建这个 path 对象
2. 设置你的 `UIBezierPath` 对象任何相关绘制属性，如 `lineWidth`，或 `lineJoinStyle`，或 `usesEvenOddFillRule`，这些绘制属性应用到真个 path
3. 调用 `moveToPoint:` 设置开始的部分的起始点 
4. 添加直线和曲线段来定义一个 subpath
5. 选择性的调用 `closePath` 来关闭 subpath，这会从最后一个部分结束点画一条直线到第一个部分的起始点。
6. 可选的重复 3，4，5 来定义额外的 subpaths

当你构建你的 path 时，你应该相对于原点 (0, 0) 来指定你的 path 上点的坐标。这样做使得将来移动 path 更简单。在绘制的过程中，path 的 points 使用起来和 current graphics context 的坐标系是一样的。如果你的 path 的方向是相对于原点的，要重定位它的话你只需要使用一个有 translation factor 的 affine transform 到 current graphics context。修改 graphics context 的优点 (相对于 path 对象自身) 是通过保存和恢复 graphics state 你可以轻易的撤销这个 transformation。

要绘制一个你的 path 对象，你使用 `stoke` 和 `fill` 方法。这些方法在当前的 graphics context 中渲染你的 path 的直线和曲线部分。渲染的过程设计到使用你的 path 对象的属性栅格化你的直线和曲线部分。栅格化的过程并不会改变 path 对象自身。结果是，你可以多次渲染一个 path 对象，在当前的 context 或另一个 context。

## Adding Lines and Polygons to Your Path
直线和多边形是简单的形状，你可以使用 `moveToPoint:` 和 `addLineToPoint:` 构建。`moveToPoint:` 方法设置你想要创建的形状的起始点。从这点开始，你通过调用 `addLineToPoint:` 方法创建形状的线段。你连续的创建这些线段，每条线段由你之前指定的点和你新指定的点构成。

下面的例子展示了使用线段创建一个 pentagon 形状所需的代码。(Figure 2-1 显式了使用合适的 stroke 和 fill 颜色绘制这个形状的结果，将在 *Rendering the Contents of a Bézier Path Object* 介绍) 这段代码设置了形状的初始点，然后添加四个相连的线段。第五个线段由 `closePath` 调用添加，它连接了最后的 (0, 40) 和起点  (100, 0)。

```objc
UIBezierPath *aPath = [UIBezierPath bezierPath];
// Set the starting point of the shape.
[aPath moveToPoint:CGPointMake(100.0, 0.0)];

// Draw the lines.
[aPath addLineToPoint:CGPointMake(200.0, 40.0)];
[aPath addLineToPoint:CGPointMake(160, 140)];
[aPath addLineToPoint:CGPointMake(40.0, 140)];
[aPath addLineToPoint:CGPointMake(0.0, 40.0)];
[aPath closePath];
```

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/shape_drawn_with_methods_of_the_uibezierpath_class.png" width=80% />

使用 `closPath` 方法不仅关闭描述形状的 subpath，同时它也在终点和起点间绘制一个线段。这是绘制一个多边形的方便方式，因为不需要绘制最后一个线段。

## Adding Arcs to Your Path
`UIBezierPath` 类提供了使用弧线段来初始化一个 path 对象的支持。方法 `bezierPathWithArcCenter:radius:startAngle:endAngle:clockwise:` 的参数定义了包含期望弧线的圆和弧线的起始和结束点。下图展示了创建一个弧线所需的组成，包括定义弧线的圆和指定它的弧度。这个例子中，弧线是按时钟的方向创建的。(按照反时钟方向的话将绘制圆的虚线部分) 。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/an_arc_in_the_default_coordinate_system.png" width=80% />

```objc
// pi is approximately equal to 3.14159265359.
#define   DEGREES_TO_RADIANS(degrees)  ((pi * degrees)/ 180)

- (UIBezierPath *)createArcPath {
    UIBezierPath *aPath = [UIBezierPath bezierPathWithArcCenter:CGPointMake(150,150) radius:75 startAngle:0 endAngle:DEGREES_TO_RADIANS(135) clockwise:YES];
    return aPath;
}
```

如果你想包含弧线到一个 path。你必须直接修改 path 对象的 `CGPathRef` 的数据类型。关于使用 Core Graphics 函数直接修改 path 的更多信息，可以查看本文的 *Modifying the Path Using Core Graphics Functions*。

## Adding Curves to Your Path
`UIBezierPath` 类提供了添加立方和平方 Bézier 曲线到一个 path 的支持。曲线段从当前的点开始，在你指定的点结束。曲线的形状是使用起点终点和一个或多个 control points 之间的射线决定的。下图粗略的展示了两种曲线的类型和 control points 和曲线形状之间的关系。每段的曲度涉及到一个所有点之间的复杂数学关系，这种关系在 [Wikipedia](https://en.wikipedia.org/wiki/Bezier_curve) 中有很好的解释。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/curve_segments_in_a_path.png" width=80% />

你使用以下方法添加曲线到一个 path:

* Cubic curve: `addCurveToPoint:controlPoint1:controlPoint2:`
* Quadratic curve: `addQuadCurveToPoint:controlPoint:`

因为依赖于 path 的当前点，你必须设置当前的点在调用上述任一方法前。一旦完成曲线，当前的点被更新到你指定的新的点。

## Creating Oval and Rectangular Paths
椭圆和矩形是是通用的 path 类型，它们基于曲线和直线段的组合。`UIBezierPath` 类包括 `bezierPathWithRect:` 和 `bezierPathWithOvalInRect:` 等便利方法来创建使用 oval 或矩形框的 path。这两个方法都创建一个新的 path 对象并使用指定的形状初始化它。你可以立刻使用返回的 path 对象或根据需要添加更多的形状到里面。

如果你想要添加一个矩形框到一个现有的 path 对象，你必须使用 `moveToPoint:`，`addLineToPoint:` 和 `closePath` 来完成，跟你创建任何多边形是一样的。使用 `closePath` 方法来完成矩形框的最后一条边是一个添加 path 最后线段和标记 subpath 结束的简便方法。

如果你想要添加一个椭圆到现有 path 对象的话，最简单的方法是使用 Core Graphics。胫骨啊你可以使用 `addQuadCurveToPoint:controlPoint:` 方法来接近一个椭圆面，`CGPathAddEllipseInRect` 函数要简单很多，而且更准确。要了解更多信息的话，可以参见本文的 *Modifying the Path Using Core Graphics Functions*

## Modifying the Path Using Core Graphics Functions
`UIBezierPath` 类实际上只是一个 `CGPathRef` 数据类型和与 path 相关绘制属性的包装。尽管你通常使用 `UIBezierPath` 类的方法来添加直线和曲线段，这个类也暴露了一个 `CGPath` 属性，你可以使用它来直接修改潜在的 path 数据。当你更喜欢使用 Core Graphics 框架中的函数来构建你的 path 的话，你也可以使用这个属性。

有两种方式修改与一个 `UIBezierPath` 对象关联的 path。你可以完全使用 Core Graphics 函数来修改，或者混合使用 Core Graphics 函数和 `UIBezierPath` 方法。完全使用 `Core Graphics` 函数调用修改 path 在某些方面要简单很多。你创建一个 mutable 的 `CGPathRef` 数据类型，然后调用任何你需要的函数来修改 path 的信息。当你修改完成的时候，将 path 对象赋给相应的 `UIBezierPath` 对象。如下所示

```objc
// Create the path data.
CGMutablePathRef cgPath = CGPathCreateMutable();
CGPathAddEllipseInRect(cgPath, NULL, CGRectMake(0, 0, 300, 300)); CGPathAddEllipseInRect(cgPath, NULL, CGRectMake(50, 50, 200, 200));

// Now create the UIBezierPath object.
UIBezierPath *aPath = [UIBezierPath bezierPath];
aPath.CGPath = cgPath;
aPath.usesEvenOddFillRule = YES;

// After assigning it to the UIBezierPath object, you can release
// your CGPathRef data type safely.
CGPathRelease(cgPath);
```

如果你选择混合使用 Core Graphics 函数和 `UIBezierPath` 方法，你必须谨慎的在两者之间移动 path 信息。因为一个 `UIBezierPath` 对象拥有潜在的 `CGPathRef` 数据，你不能简单的获取这个类型然后直接修改。相反，你必须创建一个 mutable 拷贝，修改这份拷贝，然后将这个拷贝赋给 `CGPath` 属性。

```objc
UIBezierPath *aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, 300, 300)];

// Get the CGPathRef and create a mutable version.
CGPathRef cgPath = aPath.CGPath;
CGMutablePathRef  mutablePath = CGPathCreateMutableCopy(cgPath);

// Modify the path and assign it back to the UIBezierPath object.
CGPathAddEllipseInRect(mutablePath, NULL, CGRectMake(50, 50, 200, 200));
aPath.CGPath = mutablePath;

// Release both the mutable copy of the path.
CGPathRelease(mutablePath);
```

## Rendering the Contents of a Bézier Path Object
在创建了一个 `UIBezierPath` 对象之后，你可以在当前 graphics context 中使用 `stroke` 和 `fill` 方法来渲染它。在你调用这些方法之前，有些其他任务你需要进行来保证你的 path 被正确的绘制：

* 使用 `UIColor` 设置想要的 stroke 和 fill 颜色
* 将你的 shape 放在指定视图中你想要的位置 <br />如果你相对于 (0, 0) 创建你的 path，你可以应用一个合适的 affine transform 到当前绘制 context。例如，从点 (10, 10) 开始绘制你的 shape，你可以调用 `CGContextTranslateCTM` 来在水平和竖直方向上移动 10。调整 graphics context (相对于 path 中的点而言) 是更受欢迎的方式，因为你可以通过保存和恢复之前的 graphics state 来简单的撤销之前改变。
* 更新 path 对象的绘制对象。`UIBezierPath` 实例的绘制属性在渲染时会覆盖掉与 graphics context 关联的值。

下面的代码展示了在一个自定义视图的 `drawRect:` 中绘制一个椭圆的样例实现。椭圆的 bounding rectangle 的左上角在视图坐标系的点 (50, 50) 。因为填充操作刚好填充到 path 边界，这个方法在 stroking path 之前填充它。这阻止了填充色遮住 stroked line。

```objc
- (void)drawRect:(CGRect)rect {
    // Create an oval shape to draw.
    UIBezierPath *aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, 200, 100)];
    
    // Set the render colors.
    [[UIColor blackColor] setStroke];
    [[UIColor redColor] setFill];
    
    CGContextRef aRef = UIGraphicsGetCurrentContext();
    
    // If you have content to draw after the shape,
    // save the current state before changing the transform.
    //CGContextSaveGState(aRef);
    
    // Adjust the view's origin temporarily. The oval is
    // now drawn relative to the new origin point.
    CGContextTranslateCTM(aRef, 50, 50);
    
    // Adjust the drawing options as needed.
    aPath.lineWidth = 5;
    
    // Fill the path before stroking it so that the fill
    // color does not obscure the stroked line.
    [aPath fill];
    [aPath stroke];
    
    // Restore the graphics state before drawing any other content.
    //CGContextRestoreGState(aRef);
```

## Doing Hit-Detection on a Path
要决定一个 touch 事件是否发生在一个填充了的 path 内部，你可以使用 `UIBezierPath` 的 `containsPoint:` 方法。这个方法测试 path 对象所包含的所有 closed subpaths 是否包含这个点，如果它在某个 subpath 上或里面的话返回 `YES`。

> **重要**: `containsPoint:` 方法和 Core Graphics 中的 hit-testing 函数只在 closed path 上工作。这些函数对于 open subpaths 上总是返回 `NO`。如果你想要在一个 open subpath 上进行 hit detection 的话，你必须创建你的 path 对象的一个拷贝，并在测试这些点之前 close 这个 open subpaths。

如果你想要在 path 的 stroked 部分进行 hit-testing 的话  (而不是 fill area)，你必须使用 Core Graphics。`CGContextPathContainsPoint` 函数让你测试点是否在赋给当前 graphics context 的 path 的 fill 或 stroke 部分。下面的代码展示了一个测试指定的点是否与指定的 path 相交。`inFill` 参数让调用者指定点是基于 path 的 filled 部分还是 stroked 部分比配。传入的 path 必须包含一个或多个 close subpaths，如要要 hit test 成功的话。

```objc
- (BOOL)containsPoint:(CGPoint)point onPath:(UIBezierPath *)path inFillArea:(BOOL)inFill {
   CGContextRef context = UIGraphicsGetCurrentContext();
   CGPathRef cgPath = path.CGPath;
   BOOL    isHit = NO;
    
   // Determine the drawing mode to use. Default to
   // detecting hits on the stroked portion of the path.
   CGPathDrawingMode mode = kCGPathStroke;
   if (inFill) {
        // Look for hits in the fill area of the path instead.
        if (path.usesEvenOddFillRule) {
            mode = kCGPathEOFill;
        } else {
            mode = kCGPathFill;
        }
    }

    // Save the graphics state so that the path can be removed later.
    CGContextSaveGState(context);
    CGContextAddPath(context, cgPath);
    
    // Do the hit detection.
    isHit = CGContextPathContainsPoint(context, point, mode);
    
    CGContextRestoreGState(context);
    
    return isHit;
```

---
# Drawing and Creating Images
大部分时候，使用标准的视图显示图片很直接。然而，有两种情况你可能需要额外的工作：

* 如果你的自定义视图的一部分想要展示图片，你必须在视图的 `drawRect:` 方法中自己绘制图片，下面的 *Drawing Images* 有介绍怎样做。
* 如果你想渲染图片到 offscreen (稍后绘制，或保存到一个文件中)，你必须创建一个 bitmap image context。更多信息阅读下文的 *Creating New Images Using Bitmap Graphics Contexts*。

## Drawing Images
为了最好的性能，如果你的图片绘制需求可以通过是一个 `UIImageView` 类满足的话，你应该使用这个图片对象来初始化一个 `UIImageView` 对象。然而，如果你需要显式绘制一张图片，你可以保存这张图片，并稍后在视图的 `drawRect:` 方法中使用。

下面的例子展示了怎么从应用的 bundle 中加载一张图片：

```objc
NSString *imagePath = [[NSBundle mainBundle] pathForResource:@"myImage" ofType:@"png"];
UIImage *myImageObj = [[UIImage alloc] initWithContentsOfFile:imagePath];


// Store the image into a property of type UIImage *
// for use later in the class's drawRect: method.
self.anImage = myImageObj;
```

要在视图的 `drawRect:` 方法中显式绘制结果图片的话，你可以使用 `UIImage` 类中任何可用的绘制方法。这些方法允许你指定在你的视图的什么位置绘制你的图片，因此不需要你在绘制之前创建和应用一个单独的 transform。

下面的代码片段在视图的 (10, 10) 处绘制之前加载的图片。

```objc
- (void)drawRect:(CGRect)rect {
    ...
    
    // Draw the image.
    [self.anImage drawAtPoint:CGPointMake(10, 10)];
}
```

> **重要**: 如果你使用 `CGContextDrawImag` 函数直接绘制 bitmap images 的话，图片将在 y 轴上被倒置。这是因为 Quartz images 假设了一个 LLO 的坐标系。尽管你可以在绘制之前应用一个 transform，绘制 Quartz 图片的更简单 (推荐的) 方式是包装它们到一个 `UIImage` 对象中，这样可以自动抵消掉坐标系的不同。关于使用 Core Graphics 创建和绘制图片的更多信息，可以查看 *Quartz 2D Programming Guide*。

## Creating New Images Using Bitmap Graphics Contexts
大部分时候，当绘制发生时，你的目的是在屏幕上展示些一些内容。然而，有时在 offscreen buffer 中绘制也挺有用的。例如，你需要想要创建一个现有图片的缩略图，绘制到一个 buffer 中，以便你可以保存到一个文件等等。为了支持这种需求，你可以创建一个 bitmap image context，使用 UIKit 或 Core Graphics 函数绘制内容到里面，然后从 context 中获取图片。

在 UIKit 中，过程如下：

1. 调用 `UIGraphicsBeginImageContextWithOptions` 创建一个 bitmap context，然后将它压入当前 graphics 栈中。<br />对于第一个参数，参入一个 CGSize 来指定 bitmap context 的几何大小 (单位为 point) <br/>第二个参数 (`opaque`)，如果你的图片包含 transparency (一个 alpha channel)，传递 `NO`。否则，传递 YES 来最大优化性能。<br />最后的一个参数 (`scale`)，传递 0.0 如果你的图片为设备的屏幕合适的被 scaled 的话，或传递你选择的 scale factor。<br />例如，下面的代码片段创建一个 200x200 pixels 的图片

```objc
UIGraphicsBeginImageContextWithOptions(CGSizeMake(100.0,100.0), NO, 2.0);
```

> **注意:** 通常你不需要调用相似名称的 `UIGraphicsBeginImageContext` 函数 (除了作为向后兼容的备选方案外)，因为它总是创建一个 scale factor 为1.0 的图片。如果潜在的设备是一个高分辨率的屏幕的话，一个使用 `UIGraphicsBeginImageContext` 创建的图片在渲染的时候看上去并不平滑。

2. 使用 UIKit 或 Core Graphics 代码来绘制图片的内容到新创建的 graphics context 中
3. 调用 `UIGraphicsGetImageFromCurrentImageContext` 来产生并返回一个 `UIImage` 对象，图片的内容是你所画的。如果你想要的话，你可以继续绘制并重新调用这个方法来产生额外的图片。
4. 调用 `UIGraphicsEndImageContext` 来将当前的 context 从当前的 graphcis 栈出栈

下面的代码或从 Internet 上获取一张图片，绘制到一个 image-based context 中，缩小成一个 app icon 的大小。然后从创建的 bitmap data 中获取一个 UIImage 对象，并将它赋给一个成员变量。注意 bitmap (`UIGraphicsBeginImageContextWithOptions` 的第一个参数) 的大小和绘制内容的大小 (`imageRect` 的大小) 应该匹配。如果绘制内容比 bitmap 大的话，内容的一部分将被裁剪，并最终不会显示在结果图片中。

```objc
- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    UIImage *image = [[UIImage alloc] initWithData:self.activeDownload];
    if (image != nil && image.size.width != kAppIconHeight && image.size.height != kAppIconHeight) { 
        CGRect imageRect = CGRectMake(0.0, 0.0, kAppIconHeight, kAppIconHeight);
        UIGraphicsBeginImageContextWithOptions(itemSize, NO, 0.0);
        [image drawInRect:imageRect];
        self.appRecord.appIcon = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
    } else {
        UIGraphicsEndImageContext();
    }
    self.appRecord.appIcon = image;
    self.activeDownload = nil;
    self.imageConnection = nil;
    [delegate appImageDidLoad:self.indexPathInTableView];
}
```

你可以调用 Core Graphics 方法来绘制产生的位图图片内容；代码片段如下，绘制了一个缩小的 PDF 页面的图片。注意代码在调用 `CGContextDrawPDFPage` 函数之前要翻转 graphics context 来对齐图片与 UIKit 的默认坐标系。

```objc
// Other code precedes...

CGRect pageRect = CGPDFPageGetBoxRect(page, kCGPDFMediaBox);
pdfScale = self.frame.size.width/pageRect.size.width;
pageRect.size = CGSizeMake(pageRect.size.width * pdfScale, pageRect.size.height * pdfScale);
UIGraphicsBeginImageContextWithOptions(pageRect.size, YES, pdfScale);
CGContextRef context = UIGraphicsGetCurrentContext();

// First fill the background with white.
CGContextSetRGBFillColor(context, 1.0,1.0,1.0,1.0);
CGContextFillRect(context,pageRect);
CGContextSaveGState(context);

// Flip the context so that the PDF page is rendered right side up
CGContextTranslateCTM(context, 0.0, pageRect.size.height);
CGContextScaleCTM(context, 1.0, -1.0);
  
// Scale the context so that the PDF page is rendered at the
// correct size for the zoom level.
CGContextScaleCTM(context, pdfScale,pdfScale);
CGContextDrawPDFPage(context, page);
CGContextRestoreGState(context);
UIImage *backgroundImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
  
backgroundImageView = [[UIImageView alloc] initWithImage:backgroundImage];

// Other code follows...
``

如果你跟喜欢完全使用 Core Graphics 在一个 bitmap graphics context 中绘制的话，你可以使用 `CGBitmapContextCreate` 函数来创建这个 context 并绘制你的图片内容到里面。当你结束绘制的时候，调用 `CGBitmapContextCreateImage` 函数来从这个 bitmap context 中获取一个 `CGImageRef` 类型的对象。你可以直接绘制 Core Graphics 图片，使用它来初始化一个 `UIImage` 对象。当结束后，调用 `CGContextRelease` 函数来释放这个 graphics context。

---
# Generating PDF Content
UIKit 框架提供了一系列的函数来使用本地绘制代码产生 PDF 内容。这些函数让你创建一个目标为 PDF 文件或 PDF 数据对象的 graphics context。然后你可以创建一个或多个 PDF 页面，并使用同样的 UIKit 和 Core Graphics 绘制程序绘制到这些页面。当你完成绘制的时候，你将获得你绘制内容的一个 PDF 版本。

总的绘制过程和创建任何其它图片 (本文的 *Drawing and Creating Images* 中有描述) 的过程是相似的。有以下步骤：

1. 创建一个 PDF context，并将它压栈到 graphics 栈 (本文的 *Creating and Configuring the PDF Context* 中有描述)
2. 创建一个页面 (本文的 *Drawing PDF Pages* 中有所描述)
3. 使用 UIKit 或 Core Graphcis 函数例程绘制内容到页面
4. 根据需要添加 links (如本文的 *Creating Links Within Your PDF Content* 中描述的那样) 
5. 根据需要重复 2，3，4 步
6. 结束 PDF context (本文的 *Creating and Configuring the PDF Context* 中有所描述) 以便从 context 栈中弹出 graphics。依赖于 context 是怎样创建的，要么写入结果数据到指定的 PDF 文件要么存储它到指定的 `NSMutableData` 对象。

下面的部分将使用一个简单的样例描述 PDF 创建过程。关于你使用是用来创建 PDF 内容的函数的更多信息，可以参见 *UIKit Function Reference*。

## Creating and Configuring the PDF Context
你可以使用 `UIGraphicsBeginPDFContextToData` 或 `UIGraphicsBeginPDFContextToFile` 函数来创建一个 PDF graphics context。这些函数创建一个 graphics context 并关联它到 PDF 数据的目的地。对于 `UIGraphicsBeginPDFContextToData` 函数，目的地是你提供的一个 `NSMutableData` 对象。对于 `UIGraphicsBeginPDFContextToFile` 函数，目的地是你指定的一个目的文件。

PDF 文档使用基于页面的方式组织它们的内容。这个结构强加了两个限制到你的所作的绘制上：

* 在你发出任何绘制命令之前必须有一个 open page
* 你必须指定每个 page 的大小

你使用来创建一个 PDF context 的函数允许你指定一个默认大小但是它们不会自动打开一个页面。在你创建你的 context 之后，你必须显式的打开一个新页面，要么使用 `UIGraphicsBeginPDFPage` 或 `UIGraphicsBeginPDFPageWithInfo` 函数。并且每次你想创建一个新页面，你都必须调用这些函数的一个来标记和打开一个新页面。`UIGraphicsBeginPDFPage` 函数创建一个默认大小的页面，`UIGraphicsBeginPDFPageWithInfo` 函数让你自定义页面大小和其它页面属性。

当你完成绘制之后，你通过调用 `UIGraphicsEndPDFContext` 函数来关闭这个 PDF graphics context。这个函数关闭最后的页面，并将 PDF 内容写入到你创建 context 时指定的文件或数据对象。这个函数也将这个 PDF context 从 graphics context 栈中移除。

下面的代码展示了一个应用使用 text view 中的 text 创建一个 PDF 文件的处理循环。出去三个函数调用来配置和管理 PDF context 外，大部分代码都和绘制相关。`textView` 成员变量指向包含期望内容的 `UITextView` 对象。应用使用 Core Text 框架来处理 text 布局和管理连续的页面。自定义函数 `renderPageWithTextRange:andFramesetter:` 和 `drawPageNumber:` 在后面。

```objc
- (IBAction)savePDFFile:(id)sender {
    // Prepare the text using a Core Text Framesetter.
    CFAttributedStringRef currentText = CFAttributedStringCreate(NULL,(CFStringRef)textView.text, NULL);
    if (currentText) {
        CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString(currentText);
        if (framesetter) {
            NSString *pdfFileName = [self getPDFFileName];
            // Create the PDF context using the default page size of 612 x 792.
            UIGraphicsBeginPDFContextToFile(pdfFileName, CGRectZero, nil);
            
            CFRange currentRange = CFRangeMake(0, 0);
            NSInteger currentPage = 0;
            BOOL done = NO;
            
            do {
                // Mark the beginning of a new page.
                UIGraphicsBeginPDFPageWithInfo(CGRectMake(0, 0, 612, 792), nil);
                
                // Draw a page number at the bottom of each page.
                currentPage++;
                [self drawPageNumber:currentPage];
                
                // Render the current page and update the current range to
                // point to the beginning of the next page.
                currentRange = [self renderPageWithTextRange:currentRange andFramesetter:framesetter];
                
                // If we're at the end of the text, exit the loop.
                if (currentRange.location == CFAttributedStringGetLength((CFAttributedStringRef)currentText)) {
                    done = YES;
                }
            } while (!done);
            
            // Close the PDF context and write the contents out.
            UIGraphicsEndPDFContext();  
            
            // Release the framewetter.
            CFRelease(framesetter);
        } else {
            NSLog(@"Could not create the framesetter needed to lay out the atrributed string.");
        }
        // Release the attributed string.
        CFRelease(currentText);
    } else {
        NSLog(@"Could not create the attributed string for the framesetter");
    }
}
```

## Drawing PDF Pages
所有的 PDF 绘制必须在一个页面的 context 中完成。每个 PDF 文件至少有一张页面，需要有多个页面。你通过调用 `UIGraphicsBeginPDFPage` 或 `UIGraphicsBeginPDFPageWithInfo` 函数来开始一个新页面。这些函数关闭前面的页面(如果页面是打开的话)，创建一个新的页面，并将它为绘制做好准备。`UIGraphicsBeginPDFPage` 函数使用默认的大小创建一个新页面，`UIGraphicsBeginPDFPageWithInfo` 函数允许你自定义页面大小和 PDF 页面的其他的方面。

在你创建一个页面后，你接下来的绘制命令将被 PDF graphics context 捕获并翻译成 PDF 命令。你可以在这个页面绘制任何你想要的内容，包括 text，vector shapes，和图片，就像你在你应用的自定义视图中那样。你发出的绘制命令被 PDF context 捕获，并被翻译成 PDF 数据。内容在页面上的位置完全由你决定，但是必须在页面的 boudning rectangle 内发生。

下面的代码展示两个自定义方法用来在一个 PDF 页面绘制内容。`renderPageWithTextRange:andFramesetter:` 方法使用 Core Text 创建一个适应页面的 text frame ，然后在这个 frame 中布局一些文字。在你布局你的文字后，它返回一个更新过的范围，这个范围反应了当前页面的结尾和下个页面的开始。`drawPageNumber:` 方法使用 `NSString` 绘制功能来绘制页数到每页 PDF 底脚出。

> **主意**: 这段代码使用了 Core Text 框架。请确保将它添加到项目。

```objc
// Use Core Text to draw the text in a frame on the page.
- (CFRange)renderPage:(NSInteger)pageNum withTextRange:(CFRange)currentRange andFramesetter:(CTFramesetterRef)framesetter {
    // Get the graphics context.
    CGContextRef    currentContext = UIGraphicsGetCurrentContext();
    
    // Put the text matrix into a known state. This ensures
    // that no old scaling factors are left in place.
    CGContextSetTextMatrix(currentContext, CGAffineTransformIdentity);
    
    // Create a path object to enclose the text. Use 72 point
    // margins all around the text.
    CGRect    frameRect = CGRectMake(72, 72, 468, 648);
    CGMutablePathRef framePath = CGPathCreateMutable();
    CGPathAddRect(framePath, NULL, frameRect);
    
    // Get the frame that will do the rendering.
    // The currentRange variable specifies only the starting point. The framesetter
    // lays out as much text as will fit into the frame.
    CTFrameRef frameRef = CTFramesetterCreateFrame(framesetter, currentRange, framePath, NULL);
    CGPathRelease(framePath);
    
    // Core Text draws from the bottom-left corner up, so flip
    // the current transform prior to drawing.
    CGContextTranslateCTM(currentContext, 0, 792);
    CGContextScaleCTM(currentContext, 1.0, -1.0);
     
    // Draw the frame.
    CTFrameDraw(frameRef, currentContext);
    
    // Update the current range based on what was drawn.
    currentRange = CTFrameGetVisibleStringRange(frameRef);
    currentRange.location += currentRange.length;
    currentRange.length = 0;
    CFRelease(frameRef);
     
    return currentRange;
}

- (void)drawPageNumber:(NSInteger)pageNum {
    NSString *pageString = [NSString stringWithFormat:@"Page %d", pageNum];
    UIFont *theFont = [UIFont systemFontOfSize:12];
    CGSize maxSize = CGSizeMake(612, 72);
    
    CGSize pageStringSize = [pageString sizeWithFont:theFont constrainedToSize:maxSize lineBreakMode:UILineBreakModeClip];
    CGRect stringRect = CGRectMake(((612.0 - pageStringSize.width) / 2.0), 720.0 + ((72.0 - pageStringSize.height) / 2.0), pageStringSize.width, pageStringSize.height);
    
    [pageString drawInRect:stringRect withFont:theFont];
}
```

## Creating Links Within Your PDF Content
除了绘制内容外，你也可以包含链接，将用户导航到同一个 PDF 文件的另一页或一个外部链接。创建一个链接，你必须添加一个 source rectangle 和一个链接描述到你的 PDF 页面上。链接目的地的一个属性作为这个链接的唯一标示符。要创建一个链接到指定目的地的链接，你在创建 source rectangle 的时候指定目的地的唯一标示符。

要添加一个新的链接描述到 PDF 内容，你使用 `UIGraphicsAddPDFContextDestinationAtPoint` 函数。这个函数关联一个命名的描述在当前页面的指定位置。当你想要链接到这个目的地的时候，你使用 `UIGraphicsSetPDFContextDestinationForRect` 函数来指定这个链接的源 rectangle。

下图展示这两个函数调用应用到 PDF 文档的页面时之间的关系。点击包围 "see Chapter 1" 文本的 rectangle 会降用户带领到相应的目的地，也就是 Chapter 1 的顶点。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/creating_a_link_destination_and_jump_point.png" width=80% />

除了在文档中创建链接外，你也可以使用 `UIGraphicsSetPDFContextURLForRect` 函数创建指向文档外部内容的链接。当使用这个函数创建链接的时候，你指定目标 URL，当前页面的 source rectangle。

-----
# Printing
自 iOS 4.2 起，应用可以打印内容到本地支持 AirPrint 的打印机。尽管不是所有应用需要支持打印，但如果你的应用被用来创建内容的话通常支持打印是一个很有用的特性，像一个文字处理或一个画画一个用，一个购物应用，或其他任务用户想要一个永久的记录。

这章将解释怎么添加打印功能到你的应用。抽象的说你的应用创建一个打印工作，提供要么一个准备好打印的图片数组和 PDF 文档，一个单张图片或 PDF 文档，任何一个内建的 print formatter classes，要么一个自定义的 page renderer。

> **术语备注**: *print job* 的概念在这章将多次提及。一个 *print job* 是一个工作单元，不仅包括要打印的内容，也包括在打印相关的信息，如打印机的标识，print job 的名字，打印的质量和方向。

## Printing in iOS is Designed to be Simple and Intuitive
要打印，通常用户点击与视图或用户选中的想要打印的内容相关联的导航栏或工具栏上的按钮。应用 present 一个打印选项的视图。用户选择一个打印机和不同的选项，然后请求打印。打印的请求被假脱机，控制返回到应用。如果目的打印机当前并不忙的话，打印立即开始。如果打印机正在打印，或队列中任然有工作，那么这个 print job 会保持在队列中，直到它移到到队列的顶端，并被打印。

### The Printing User Interface
关于打印用户首先看到的是一个打印按钮。打印按钮通常是导航栏或工具栏上的 bar-button item. 这个打印按钮逻辑上应该应用到当前显示的视图上；如果用户点击这个按钮，应该应该打印内容。尽管打印按钮可以是任何按钮，但是推荐你使用系统的 item-action. 如下图所示。这是一个 `UIBarButtonItem` 对象，由常量 `UIBarButtonSystemItemAction` 指定，你可以在 Interface Builder 或通过调用 `initWithBarButtonSystemItem:target:action:` 创建。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/system_item_action_button_used_for_printing.png" width=80% />

当你点击一个打印按钮的时候，应用的一个 controller 对象受到一个 action 消息。controller 通过准备好打印和显示一个打印选项视图来响应。选项总是包括目标打印机 (从一个被发现的打印机列表中选择)，打印的份数，有些时候打印页面的范围。如果选择的打印机支持双面打印，用户可以选择单面或双面输出。如果用户决定不打印，iPad 用户点击打印视图的外围，iPhone 上点击 *Cancel* 按钮来 dismiss 打印视图。

打印的用户界面依赖于具体的设备。在 iPad 上，UIKit 框架显示一个 popover 视图，如下图所示。一个你应用可以动画显示这个视图，从打印按钮或应用视图的任意位置。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/printer-options_popover_view_ipad.png" width=80% />

在 iPhone 或 iPod touch 设备上时，UIKit present 一个打印选项的视图。如下图

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/printer-options_sheet_iphone.png" width=80% />

一旦一个打印任务提交后，要么它在被打印，要么在打印队列中等待，用户可以通过双击 Home 按钮访问 Print Center 来检查它的状态。Print Center 是一个后台应用，显示了打印队列中任务的顺序，包括正在打印的这些。这个应用只在一个打印工作正在进行的时候可用。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/print_center.png" width=80% />

用户可以点击一个在 Print Center 的 print job 来获取关于它的具体信息。如下图，并取消正在打印或队列中等待的任务。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/print_center_detail_of_print_job.png" width=80% />

### How Printing Works in iOS
一个应用使用 UIKit 打印 API 来组装一个 print job 的元素，包括打印的内容和 print job 相关的信息。它然后显示本文 *The Printing User Interface* 中描述的打印机选项。用户做出他的选择，然后点击打印。在某些情况下，UIKit 框架要求应用绘制待打印的内容；UIKit 记录应用绘制的内容为 PDF 数据。UIKit 然后提交打印数据到打印子系统。

打印系统做了一些事。当 UIKit 传递打印数据到打印子系统时，它将数据写入存储 (即将数据脱机化)。它同样捕获关于 job 的信息。打印系统管理组合起来的 print data 和 metadata 为每个在 first-in-first-out 的打印队列中打印工作。多个应用可以提交多个打印工作到打印子系统上，所有的打印工作被置于打印队列。每个设备有一个队列供所有的打印工作使用，不管源应用或目的打印机是哪一个。

当一个打印工作到达队列的顶端时，系统的打印守候进程 (`printd`) 会考虑目的打印机的要求，需要的话，转换打印数据成打印机可用的格式。打印系统报告如 "Out of Paper" 的错误给用户。它同样报告打印工作的进度给 Print Center，Print Center 显示一个 print job 的信息如 "page 2 of 5" 。

## The UIKit Printing API
UIKit 打印 API 包括八个类和一个 formal protocol。这些类的对象和实现协议的 delegate 代码间有以下图示关系。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/relationships_of_uikit_printing_objects.png"/>

### Printing Support Overview
抽象的层次看，有两种方式给应用增加打印的支持。如果你使用一个 `UIActivityViewController`，并且如果你不需要控制用户是否能够选择页面范围或覆盖页面选择的能力，你可以添加一个 printing activity。

除了上述方式外，应用要支持打印，你必须使用 `UIPrintInteractionController` 类。一个 `UIPrintInteractionController` 共享实例给应用提供了指定应用打印时应该怎么做的能力。它包含关于 print job 的信息 (`UIPrintInfo`) 和页面的大小和供打印内容区域 (`UIPrintPaper`)。它也包含一个实现了 `UIPrintInteractionControllerDelegate` 协议的 delegate 对象引用来进一步配置打印行为。

更重要的是，这个 print interaction controller 让你的应用提供打印的内容。`UIPrintInteractionController` 类提供了三种方式来打印内容：

* 静态的图片或 PDFs。对于简单的内容，你可以使用 print interaction controller 的 `printingItem` 属性或 `printingItems` 属性来提供一张图片 (可以有多种格式)，一个 PDF 文件，或一组图片或 PDF 文件
* Print formatters. 如果你需要自动排版来打印 text 和 HTML 内容的话，你可以赋任何一个内建的 print formatter 类的实例给 print interaction controller 的 `printFormatter` 属性。
* Page renderers. Page renderers 让你来提供对于自定义内容的绘制程序，并给你对页面布局，包括页头和页尾的完全控制。要使用一个 page renderer，你必须要先写一个 page renderer 的子类，然后赋一个它的实例给 print interaction controller 的 `printPageRenderer` 属性。

> **重要**: 这四个属性是相互排斥的。也就是说，如果你附一个值给其中一个属性的话，UIKit 会保证其他的属性的值为 nil。

有这么多选择可供你选择，对于你的应用的最好选择是什么？下表澄清了做这个决策时涉及到的因素。

if... | Then...
---- | --------
你的应用直接访问可打印的内容 (图片或 PDF 文档) | 使用 `printingItem` 或 `printingItems` 属性
你想要打印单张图片或 PDF 文档并想要用户能够选择一个页面范围 | 使用 `printingItem` 属性
你想要打印一段纯文本或 HTML 文档 (并且不想要诸如 headers 和 footers 等额外内容) | 赋一个 `UISimpleTextPrintFormatter` 或 `UIMarkupTextPrintFormatter` 对象给 `printFormatter` 属性。这个 print-formatter 对象必须使用纯文本或 HTML 文本来初始化
你想要打印一个 UIKit 视图 (并且不想要诸如 header 和 footer 等额外内容) | 从视图获取一个 `UIViewPrintFormatter` 对象并将它赋给 `printFormatter` 属性
你想要打印的页面有重复的 headers 和 footers，可能的话有递增的页面数 | 赋一个 `UIPrintPageRenderer` 的自定义子类实例给 `printPageRenderer` 属性。这个自定义子类应该实现绘制所需 headers 和 footers 所需的方法
你有混合的内容或源需要打印——例如，HTML 和自定义绘制内容 | 赋一个 `UIPrintPageRenderer`  (或自定义子类)  的实例给 `printPageRenderer` 属性。你可以添加一个或多个 print formatters 来渲染特定的页的内容。如果你使用一个 `UIPrintPageRenderer` 类的自定义子类，你也同样可以选择提供自定义绘制代码来渲染一些或全部页面。
你想要拥有对绘制给打印的内容的最大控制 | 赋一个 `UIPrintPageRenderer` 的自定义子类实例给 `printPageRenderer` 属性并绘制任何供打印的内容

### Printing Workflow
打印一个图片，文档，或其它可打印的内容的一般工作流如下：

1. 获取一个 `UIPrintInteractionController` 类的共有实例
2. (可选，但强烈推荐) 创建一个 `UIPrintInfo` 对象，设置输出类型、job 名称，和打印方向；然后赋这个对象给 `UIPrintInteractionController` 实例的 `printInfo` 属性。 (设置输出类型和 job 名称是强烈建议的)<br />如果你不设置 print-info 对象，UIKit 给 print job 设置假设默认的属性 (例如，job 的名字是应用的名字)
3. (可选) 赋一个自定义 controller 对象给 `delegate` 属性。这个对象必须实现 `UIPrintInteractionControllerDelegate` 协议。这个 delegate 可以做一定范围的任务。当打印选项显示或消失，打印工作开始或结束的时候它可以合适的响应。它返回一个 parent view controller 给 print interaction controller。<br />同样，默认情况下，UIKit 基于输出类型选择一个默认页面大小和可打印的区域，这个 output type 标识着你的应用打印的内容的类型。如果你的应用需要关于页面大小的更多控制，你可以覆盖这种标准行为。参见本文的 *Specifying Paper Size, Orientation, and Duplexing Options*
4. 赋下面的情况之一给 `UIPrintInteractionController` 实例的一个属性：
	* 单个 `NSData`, `NSURL`, `UIImage`, 或一个包含或引用 PDF 数据或图片数据的 `ALAsset` 到 `printingItem` 属性。
	* 一组直接可以打印的图片或 PDF 文档给 `printingItems` 属性。数组的元素必须是 `NSData`, `NSURL`, `UIImage`, 包含、引用或代表 PDF 数据或支持的图片。
	* 一个 `UIPrintFormatter` 对象到 `printFormatter` 属性。print formatters 进行可打印内容的自定义布局。
	* 一个 `UIPrintPageRenderer` 对象给 `printPageRenderer` 属性。
	
	对于任何一个打印 job 这些属性中只能有一个可以是 non-nil。查看本文的 *Printing Support Overview* 来了解关于这些属性的更多消息。
5. 如果你在一个之前的步骤中赋一个 page renderer，这个对象通常是一个 `UIPrintPageRenderer` 自定义子类的实例。这个对象绘制 UIKit 框架要求打印的页面内容。它可以在页面的 headers 和 footer 内绘制内容。一个自定一个 page renderer 必须覆盖一个或多个 "draw" 方法，如果它至少绘制内容的一部分的话 (除去 headers 和 footers)，它必须计算和返回 print job 的页面数。(注意你可以使用一个 `UIPrintPageRenderer` 实例来连接一些列的 print formatters)
6. (可选) 如果你使用一个 page renderer，你可以使用这个类具体子类创建一个或多个 `UIPrintFormatter` 对象；然后添加指定的页面 (或页面范围) 的 print formatters 到 `UIPrintPageRenderer` 实例，要么通过调用 `UIPrintPageRenderer` 的 `addPrintFormatter:startingAtPageAtIndex:` 方法或通过创建包含一个或多个 print formatters (每个有它自己的开始页) 的数组，并将数组赋给 `UIPrintPageRenderer` 的 `printFormatters` 属性。
7. 如果你的设备是一个 iPad，显示打印接口给用户的方式是调用 `presentFromBarButtonItem:animated:completionHandler:` 或 `presentFromRect:inView:animated:completionHandler:` . 如果设备是一个 iPhone 或 iPod touch，调用 `presentAnimated:completionHandler:`. 替换方案是，你可以将 print UI 嵌入你现有的 UI 中通过实现一个 `printInteractionControllerParentViewController:` delegate 方法。如果你的应用使用一个 activity sheet， 你也可以使用一个 printing activity item。

从这里开始，过程会依赖于你是否在静态内容，一个 print formatter 或一个 page  renderer 来打印。

## Printing Printer-Ready Content
iOS 打印系统接受某些对象，并直接打印它们的内容，需要应用很少的介入。这些对象是 `NSData` `NSURL` `UIImage` 和包含或指向 image 数据或 PDF 数据的 `ALAsset` 类。图片数据涉及到所有这些对象类型。PDF 数据要么被 `NSURL` 对象引用或被 `NSData` 对象封装。对于这些 printer-ready 的对象有额外的要求。

* 一张图片必须是 Image I/O 框架支持的类型。在  *UIImage Class Reference* 中查看支持的图片类型的更多信息。
* `NSULR` 对象必须使用 `file:` `assets-library:` 的 sheme，或其使用一个注册了协议的对象可以返回一个 `NSData` (例如，QuickLook's `x-apple-ql-id:` sheme)
* 一个类型为 `ALAssetTypePhoto` 的 `ALAsset` 对象

你可以将 printer-ready 的对象给 `UIPrintInteractionController` 共享实例的 `printingItem` 或 `printingItems` 属性。单个 printer-ready 对象赋给 `printingItem`，一组 printer-ready 对象赋给 `printingItems` 属性。

> **注意**: 通过提供 printer-ready 内容，你是让打印系统来提供内容的布局。因此，诸如打印方向的设置没有效果。如果你的应用需要控制布局，你必须自己绘制。<br />同样，如果你的应用使用 `printingItems` (而不是 `printingItem`)，用户在打印机选项页不能设置页面范围，即使有多个页面并且 `showsPageRange` 属性被设置为 `YES`

在给这些属性赋值之前，你应该使用 `UIPrintInteractionController` 的方法之一来验证这些对象。例如如果你有一个图片的 UTI，你想要验证下这张图片是否是 printer-ready 的，你可以使用 `printableUTIs` 类方法来测试它，这个方法返回对于打印系统有效的 UTIs 集合。

```objc
if ([[UIPrintInteractionController printableUTIs] containsObject:mysteryImageUTI]) {
    printInteractionController.printingItem = mysteryImage;
}
```

同样，你可以应用 `UIPrintInteractionController` 的 `canPrintURL:` 和 `canPrintData:` 类方法到 `NSURL` 和 `NSData` 对象，在将这些对象赋给 `printingItem` 或 `printingItems` 属性前。这些方法决定了打印系统是否可以直接打印这些对象。强烈推荐使用它们，尤其是 PDF。

下面的代码展示了打印一个封装在 `NSData` 对象里的 PDF 文档。在将它赋给 `printingItem` 属性前，会先测试它的有效性。代码中也让 print interaction controller 包括 page-range controls 到打印可选项中。

```objc
- (IBAction)printContent:(id)sender {
    UIPrintInteractionController *pic = [UIPrintInteractionController sharedPrintController];
    if  (pic && [UIPrintInteractionController canPrintData: self.myPDFData] ) {
        pic.delegate = self;
        UIPrintInfo *printInfo = [UIPrintInfo printInfo];
        printInfo.outputType = UIPrintInfoOutputGeneral;
        printInfo.jobName = [self.path lastPathComponent];
        printInfo.duplex = UIPrintInfoDuplexLongEdge;
        pic.printInfo = printInfo;
        pic.showsPageRange = YES;
        pic.printingItem = self.myPDFData;
        
        void (^completionHandler)(UIPrintInteractionController *, BOOL, NSError*) = ^(UIPrintInteractionController *pic, BOOL completed, NSError *error) {
            self.content = nil;
            if (!completed && error) {
                NSLog(@"FAILED! due to error in domain %@ with error code %u", error.domain, error.code);
            }
        };
        if (UI_USER_INTERFACE_IDIOM() == UIUserInterfaceIdiomPad) {
            [pic presentFromBarButtonItem:self.printButton animated:YES completionHandler:completionHandler];
        } else {
            [pic presentAnimated:YES completionHandler:completionHandler];
        }
    }
}
```

对于一次提交几个 printer-ready 的对象的过程是相同的——除了，你必须赋一组这样的对象给 `printingItems` 属性。

```objc
pic.printingItems = [NSArray arrayWithObjects:imageViewOne.image, imageViewTwo.image, imageViewThree.image, nil];
```

## Using Print Formatters and Page Renderers
Print formatters 和 page renderers 布局可打印内容到多个页面的对象。它们指定页面内容的开始并基于开始页面计算最后页面的内容，内容的区域，和它们布局的内容。它们也可以指定相对于页面打印区域的间距 (margins)。区别总结如下：

* Print formatters 布局单份内容 (一块文本或 HTML，一个视图等等)
* Page renderers 让你添加 headers 和 footers，进行自定义的绘制等等。Page renderers 可以使用 print formatters 做大部分它们的工作。 <br />如果你创建一个 `UIPrintPageRenderer` 的自定义子类，你可以绘制可打印内容每一页的部分或全部内容。一个 page renderer 可以有一个或多个 print formatters 与指定的页面或页面范围的打印内容相关联。

### Setting the Layout Properties for the Print Job
要为可打印内容定义页面区域，`UIPrintFormatter` 类给它的子类申明了 4 个关键属性。这些属性，包括 `UIPrintPageRenderer` 的 `footerHeight` 和 `headerHeight` 和 `UIPrintPaper` 的 `paperSize` 和 `printableRect` 属性，定义了多页打印任务的布局。下表对这种布局有所说明。

Property | Description
------------ | -----------------
`contentInsets` | 离 printable rectangle 的上下左右边界的 inset，这些值打印内容的 margeins，但是它们可以被 `maximumContentHeight` 和 `maximumContentWidth` 的值覆盖。top inset 只应用到一个给定 formatter 的第一个页面。
`maximumContentHeight` | 指定内容区域的最大高度，它是任何 header 或 footer 高度的因子。UIKit 将这个值与内容的高度减去 top content-inset 做比较，并使用两者间更小的值
`maximumContentWidth` 指定内容区域的最大宽度。UIKit 将这个值与左右 content-inset 创建的值作比较，取更小的值。
`startPage` | formatter 应该开始给打印绘制内容的页数。这个值是基于 0 的——也就是说输出的第一页值为 0 —— 但是一个 page renderer 使用的 formatter 可以从稍后的某页开始绘制。<br />例如，告诉一个 formatter 从第三页开始，一个 page renderer 将会指定 2 给 `startPage`

> **注意**: 如果你使用 `UIPrintInteractionController` 的 `printFormatter` 属性的话，就不会有 `UIPrintPageRenderer` 对象，这意味着你不能指定页首和页脚。

<img src="https://raw.githubusercontent.com/0oneo/iOSTranslation/master/images/iOSDrawingPrinting/the_layout_of_a_multi-page_print_job.png" width=80% />

`UIPrintFormatter` 使用上图所示的所有属性来计算打印工作所需的页面数，它将这个值存在只读属性 `pageCount` 上。

### Using a Print Formatter
UIKit 允许你设置一个简单的 print formatter 给一个 print job。如果你有纯文本或 HTML 需要打印的话，这可以是一个有用的功能，因为对于这类文本内容，UIKit 有具体的 print-formatter 类。框架也实现了具体的子类以一种友好的方式支持打印 UIKit 中视图。

`UIPrintFormatter` 类是系统提供的 print formatters 的抽象基类。当前，iOS 提供下面的内建 print formatters：

* `UIViewPrintFormatter`——自动布局一个视图的内容到多个页面。要获取一个视图的 print formatter，调用视图的 `viewPrintFormatter` 方法。<br />注意并不是所有的内置 UIKit 视图都支持打印。当前，只有 `UIWebView`, `UITextView`, `MKMapView` 等视图知道怎么绘制它们的内容供打印。
* `UISimpleTextPrintFormatter` —— 自动绘制和布局纯文本内容。这个 formatter 允许你为文本设置全局的属性。如字体，颜色，对其，和换行模式。
* `UIMarkupTextPrintFormatter` —— 自动绘制和布局 HTML 文档。

> **注意**: `UIPrintFormatter` 类不是为了给第三方类继承。如果你需要自定义布局，相反你应该编写一个 page renderer。

尽管下面的讨论适用于使用单个 formatter，但关于 print formatters 的大部分信息同样适用于与 page renderers 一起使用的 print formatter，printer formatter 与 page renderers 一起使用在 *Using One or More Formatters with a Page Renderer* 中描述。

#### Printing Text or HTML Documents
许多应用包括用户想要打印的文本信息。如果内容是纯文本或 HTML 文本，并且你能够访问所显示文本内容的 backing string 的话，你可以使用使用 `UISimpleTextPrintFormatter` 或 `UIMarkupTextPrintFormatter` 实例来布局和绘制打印文本。简单的创建这个实例，使用 backing string 初始化它，指定布局属性。然后将它赋给 `UIPrintInteractionController` 的 `printFormatter` 成员变量。

下面的代码说明了怎么使用 `UIMarkupTextPrintFormatter` 对象来打印一个 HTML 文档。它添加了一英寸的间隔给打印区域。你可以通过设置 `UIPrintInteractionController` 对象的 `printPaper` 属性来决定打印区域。关于怎么创建指定宽度间隔的完整例子，可以查看 *UIKit Printing with UIPrintInteractionController and UIViewPrintFormatter* 样例代码项目。

```objc
- (IBAction)printContent:(id)sender {
    UIPrintInteractionController *pic = [UIPrintInteractionController sharedPrintController];
    pic.delegate = self;
    
    UIPrintInfo *printInfo = [UIPrintInfo printInfo];
    printInfo.outputType = UIPrintInfoOutputGeneral;
    printInfo.jobName = self.documentName;
    pic.printInfo = printInfo;
    
    UIMarkupTextPrintFormatter *htmlFormatter = [[UIMarkupTextPrintFormatter alloc] initWithMarkupText:self.htmlString];
    htmlFormatter.startPage = 0;
    htmlFormatter.contentInsets = UIEdgeInsetsMake(72.0, 72.0, 72.0, 72.0); // 1 inch margins
    pic.printFormatter = htmlFormatter;
    pic.showsPageRange = YES;
      
    void (^completionHandler)(UIPrintInteractionController *, BOOL, NSError *) = ^(UIPrintInteractionController *printController, BOOL completed, NSError *error) {
        if (!completed && error) {
            NSLog(@"Printing could not complete because of error: %@", error);
        }
    };
    if (UI_USER_INTERFACE_IDIOM() == UIUserInterfaceIdiomPad) {
        [pic presentFromBarButtonItem:sender animated:YES completionHandler:completionHandler];
    } else {
        [pic presentAnimated:YES completionHandler:completionHandler];
    }
}        
```

记住对于一个 print job 你使用单一 print formatter (也就是说一个 `UIPrintFormatter` 对象赋给 `UIPrintInteractionController` 实例的 `printFormatter` 属性)，你不能在打印的每页内容上设置页首和页脚。要这么做的话，你必须使用一个 `UIPrintPageRenderer` 对象外加任何需要的 print formatters。更多的信息可以参见本文的 *Using One or More Formatters with a Page Renderer* 

使用 `UISimpleTextPrintFormatter` 对象来布局和打印纯文本文档的过程几乎完全相同。然而，这个对象的类包括属性来设置字体，颜色，打印文本的对齐方式。

#### Using a View Print Formatter
使用一个 `UIViewPrintFormatter` 实例来布局和打印一些系统视图的内容。UIKit 框架为视图创建这些 view print formatter。通常用户绘制视图显示的代码被用于绘制视图来打印。目前，你可以使用一个 view print formatter 来打印视图内容的系统视图有 `UIWebView`, `UITextView`, `MKMapView` (MapKit)。

要获取一个 `UIView` 对象的 view print formatter，调用视图的 `viewPrintFormatter` 方法。设置开始页和任何布局信息，然后设置这个对象给 `UIPrintInteractionController` 的 `printFormatter` 属性。另一种方式是，你可以添加 view print formatter 到一个 `UIPrintPageRenderer` 对象，如果你使用这个对象来绘制部分你打印的内容。下面的代码展示了使用一个 `UIWebView` 的 view print formatter 来打印视图的内容。

```objc
- (void)printWebPage:(id)sender {
    UIPrintInteractionController *controller = [UIPrintInteractionController sharedPrintController];
    void (^completionHandler)(UIPrintInteractionController *, BOOL, NSError *) = ^(UIPrintInteractionController *printController, BOOL completed, NSError
  *error) {
        if(!completed && error){
            NSLog(@"FAILED! due to error in domain %@ with error code %u", error.domain, error.code);
        }
    };
    UIPrintInfo *printInfo = [UIPrintInfo printInfo];
    printInfo.outputType = UIPrintInfoOutputGeneral;
    printInfo.jobName = [urlField text];
    printInfo.duplex = UIPrintInfoDuplexLongEdge;
    controller.printInfo = printInfo;
    controller.showsPageRange = YES;
    
    UIViewPrintFormatter *viewFormatter = [self.myWebView viewPrintFormatter];
    viewFormatter.startPage = 0;
    controller.printFormatter = viewFormatter;
    
    if (UI_USER_INTERFACE_IDIOM() == UIUserInterfaceIdiomPad) {
        [controller presentFromBarButtonItem:printButton animated:YES completionHandler:completionHandler];
    } else {
        [controller presentAnimated:YES completionHandler:completionHandler];
    }
}
```

基于 `UIViewPrintFormatter`完整例子，可以查看 *UIKit Printing with UIPrintInteractionController and UIViewPrintFormatter* 样例代码项目。

### Using a Page Renderer
一个 page renderer 是 `UIPrintPageRenderer` 自定义子类的一个实例，它绘制一个 print job 的部分或全部内容。要使用一个，你可以创建子类，添加它到你的项目，当你为一个 print job 准备 `UIPrintInteractionController` 实例时实例化它。然后将这个 page renderer 赋给 `UIPrintInteractionController` 实例的 `printPageRenderer` 属性。一个 page renderer 可以有一个或多个 print formatters 与之关联，如果确实有的话，它会将它的绘制和 print formatters 的绘制混合使用。

一个 page renderer 靠自己就可以绘制和布局可打印内容，它也可以使用 print formatters 来处理指定页面范围的部分或全部渲染。因此，为了相对直接的格式化需求，你可以使用一个 `UIPrintPageRenderer` 实例来连接多个 print formatters。然而，大部分 page renderers 通常是 `UIPrintPageRenderer` 自定义子类的实例。

> **注意**: 如果你想要打印页首或页脚的话，如一个重复的文档标题和一个递增的页面数，你必须使用一个 `UIPrintPageRenderer` 的子类。

`UIPrintPageRenderer` 基类包括页面数相关的属性，页首和页脚高度相关的属性。它也申明了几个你可以在你覆盖的方法来绘制页面指定区域的部分：页首，页脚，内容自身，或集成 page renderer 和 print formatter 的绘制。

#### Setting Page Renderer Attributes
如果你的 page renderer 将要在每个打印页的页首和页脚绘制的话，你应该给页首和页脚指定一个高度。要这么做的话，在你的子类中赋一个浮点数给 `headerHeight` 和 `footerHeight` 属性。如果这些属性有 0 值的话 (即默认值)，`drawHeaderForPageAtIndex:inRect:` 和 `drawFooterForPageAtIndex:inRect:` 方法不会被调用。

如果你的 page renderer 将要在页面的内容区绘制的话——也就是，在页首和页脚之间的区域——那么你的子类应该覆盖 `numberOfPages` 方法，这个方法计算出并返回你的 page renderer 需要绘制的页面数。如果与这个 page renderer 相关联的 print formatters 将要绘制页首和页脚之间的所有内容，那么 print formatters 将为你计算页面数。这种情况发生在你的 page renderer 只在页首和页脚区域绘制，并且你让 print formatter 绘制所有其他的内容。

使用一个没有关联 print formatter 的 page renderer，打印内容的每页布局完全由你决定。当计算布局参数时，你可以考虑 `UIPrintPageRenderer` 的 `headerHeight`, `footerHeight`, `paperRect`, `printableRect` 属性。如果 page renderer 使用 print formatters，那么布局尺度同样要包括 `UIPrintFormatter` 的 `contentInsets`, `maximumContentHeight` 和 `maximumContentWidth` 属性。图示和解释可以查看本文的 *Setting the Layout Properties for the Print Job* 。

#### Implementing the Drawing Methods
当一个应用使用一个 page renderer 来绘制可打印内容的时候，UIKit 对于每页请求的内容调用以下方法。注意没有保证说 UIKit 调用这些方法以页面的书序来。不仅如此，如果用户请求一部分页面打印，UIKit 对于不在这部分的页面都不调用这些方法。

> **注意**: 记住如果你使用 non-UIKit API 绘制图片或 PDF 文档，那么你必须 "flip" UIKit 的坐标系——将原点放在左下角正值向上——与 Core Graphics 使用的坐标系匹配。更多信息查看本文的 *Coordinate Systems and Drawing in iOS*。

`drawPageAtIndex:inRect:` 方法按照下表的顺序调用每个其它的绘制方法。如果你想要完全控制什么被绘制来打印的话，你可以覆盖这个方法。

Override ... | To ...
---------------- | --------
`drawHeaderForPageAtIndex:inRect:` | 在页首绘制内容。这个方法在 `headerHeight` 为 0 时不被调用。
`drawContentForPageAtIndex:inRect:` | 绘制 print job 的内容 (也就是，在页首和页尾之间区域)
`drawPrintFormatter:forPageAtIndex:` | 混入 print formatter 进行的自定义绘制。这个方法对于每个与一个给定页面相关联的 print formatter 都会调用。更多信息，可以参见本文的 *Using One or More Formatters with a Page Renderer* 
`drawFooterForPageAtIndex:inRect:` | 绘制页尾内容。这个方法在 `footerHeight` 为 0 时不被调用。

所有这些绘制方法被设置来绘制到当前的 graphics context (就是 `UIGraphicsGetCurrentContext`)。传给每个方法的 rectangle —— 定义页首区域，页脚区域，内容区域，和整个页面——是相对于页面的原点而言的，这点在左上角。

下面的代码展示了 `drawHeaderForPageAtIndex:inRect:` 和 `drawFooterForPageAtIndex:inRect:` 方法的样例实现。它们使用 `CGRectGetMaxX` 和 `CGRectGetMaxY` 来计算文本在可打印内容坐标系中 `footerRect` 和 `headerRect` 矩形框中的位置。

```objc
- (void)drawHeaderForPageAtIndex:(NSInteger)pageIndex inRect:(CGRect)headerRect {
    UIFont *font = [UIFont fontWithName:@"Helvetica" size:12.0];
    CGSize titleSize = [self.jobTitle sizeWithFont:font];
    //center title in header
    CGFloat drawX = CGRectGetMaxX(headerRect)/2 - titleSize.width/2;
    CGFloat drawY = CGRectGetMaxY(headerRect) - titleSize.height;
    CGPoint drawPoint = CGPointMake(drawX, drawY);
    [self.jobTitle drawAtPoint:drawPoint withFont: font];
}

- (void)drawFooterForPageAtIndex:(NSInteger)pageIndex  inRect:(CGRect)footerRect {
    UIFont *font = [UIFont fontWithName:@"Helvetica" size:12.0];
    NSString *pageNumber = [NSString stringWithFormat:@"%d.", pageIndex+1];
    // page number at right edge of footer rect
    CGSize pageNumSize = [pageNumber sizeWithFont:font];
    CGFloat drawX = CGRectGetMaxX(footerRect) - pageNumSize.width - 1.0;
    CGFloat drawY = CGRectGetMaxY(footerRect) - pageNumSize.height;
    CGPoint drawPoint = CGPointMake(drawX, drawY);
    [pageNumber drawAtPoint:drawPoint withFont: font];
}  
```

#### Using One or More Formatters with a Page Renderer
一个 page renderer 可以和一个或多个 print formatters 一起绘制可打印内容。例如，一个应用可以使用一个 `UISimpleTextPrintFormatter` 对象来绘制页面的文本内容来供打印，但是使用一个 page renderer 绘制文档标题到每页的页首。或者一个应用可以使用两个 print formatters，一个在第一页的头部绘制头部信息 (概述)，另一个 print formatter 绘制剩余的内容。然后它可能使用一个 page renderer 来绘制一条线来分隔两部分。

你可能记得，通过设置 `UIPrintInteractionController` 的 `printFormatter` 属性可以为一个 print job 指定一个简单的 print formatter。但是如果你使用一个 page renderer 和 print formatters 的话，你必须关联每个 print formatter 到 page renderer。你通过以下方式之一来这么做：

* 添加每个 print formatter 到一个数组，将数组赋给 `printFormatters` 属性
* 通过调用 `addPrintFormatter:startingAtPageAtIndex:` 方法添加每个 print formatter

在你关联一个 print formatter 到一个 page renderer 之前，确保设置了它的布局属性，包括 print job 的开始页 (`startPage`)。一旦你设置了这些属性，`UIPrintFormatter` 会给这个 print formatter 计算页面数。注意如果你通过调用 `addPrintFormatter:startingAtPageAtIndex:` 来指定一个开始页的话，这个值会覆盖掉任何赋给 `startPage` 的值。关于 print formatters 的讨论和布局度量，可以查看本文的 *Setting the Layout Properties for the Print Job*。

一个 page renderer 可以覆盖 `drawPrintFormatter:forPageAtIndex:` 来将它的绘制与赋给给定页面的 print formatter 进行的绘制集成。它可以在 print formatter 没有绘制的页面区域绘制，然后调用传进来的 print formatter 的 `drawInRect:forPageAtIndex:` 方法绘制页面的部分内容。或者 page renderer 可以通过让 print formatter 先绘制然后绘制一些内容到 print formatter 绘制的内容上面。

关于 `UIPrintPageRenderer` 的完整例子，可以查看 *PrintPhoto , Sample Print Page Renderer* 和 *UIKit Printing with UIPrintInteractionController and UIViewPrintFormatter* 样例代码项目。

## Testing the Printing of App Content
iOS 4.2 的SDK 起提供了一个 Print Simulator 应用，你可以使用来测试应用的打印能力。应用是模拟各种通用类型的打印机 (inkjet, black-and-white-laser, color laser 等等)。它在 OS X preview 应用中展示打印的页面。你可以设置一个偏好来显示每个页面的可打印区域。Print Simulator 记录从打印系统关于每个 print job 的日志信息。

你可以通过以下方式之一来跑起 Printer Simulator：

* 从 iOS Simulator 的 File 菜单中选择打开 Printer Simulator
* 在 Xcode 中选择 Xcode > Open Developer Tool 菜单
* 你可以在下面路径中找到 Printer Simulator

```bash
<Xcode>/Platforms/iPhoneOS.platform/Developer/Applications/PrinterSimulator.app
```

当测试应用的打印代码时，你也应该实现传给 `present...` 方法的 completion handler，并记录任何从打印系统来的错误日志。这些错误通常是编程错误，通常应该在应用部署前解决。具体查看本文的 *Responding to Print-Job Completion and Errors* 

## Common Printing Tasks
下面描述的编程任务是应用在响应用户打印的请求的一些。尽管大部分任务可能以任何顺序发生，你首先应该检查设备是否能够打印，然后以显示打印选项结束。具体例子请查看本文的 *Printing Printer-Ready Content*， *Using Print Formatters and Page Renderers* 和 *Using a Page Renderer*。

一个重要的任务这里没有提及是添加打印按钮到应用合适的位置，申明一个 action 方法，创建一个 target-action 连接，然后实现这个 action 方法。 (关于使用哪个按钮的推荐请看本文的 *The Printing User Interface* )，下面的任务 (除了 *Specifying Paper Size, Orientation, and Duplexing Options*) 是实现 action 方法的一部分。

### Testing for Printing Availability
一些 iOS 设备不支持打印。一旦你的视图加载你就应该立即知道这个情况。如果打印对于你的设备不可用，你要么通过编程不添加任何打印界面元素 (按钮，bar button item等等)，或你应该移除任何从 nib 文件中加载的任何打印元素。要决定打印是否可用，调用 `UIPrintInteractionController` 类的类方法 `isPrintingAvailable`。下面的代码给你展示怎么做，它假设一个 print button 从 nib 中加载。

```objc
- (void)viewDidLoad {
    if (![UIPrintInteractionController isPrintingAvailable])
        [myPrintButton removeFromSuperView];
    // other tasks...
}
```

> **注意**: 尽管你可以关闭一个打印元素，但移除是推荐的。方法 `isPrintingAvailable` 对于一个给定设备返回值从不会改变，它反映了这个设备是否支持打印，而不是打印是否当前可用。

### Specifying Print-Job Information
一个 `UIPrintInfo` 类实例封装了一个 print job 的信息，具体如下：

* 输出类型 (表明了内容的类型)
* print-job 的名字
* 打印方向
* duplex 模式
* 选择的打印机标识

你不需要赋值给所有的 `UIPrintInfo` 属性；用户选择设置一些，对于其他属性 UIKit 假设默认值。(实际情况是，你甚至不需要显式创建 `UIPrintInfo` 的实例)

然而，在大部分情况下你将想要指定一个 print job 的一些方面，如输出类型。通过调用 `printInfo` 类方法获取一个 `UIPrintInfo` 实例。赋值给这个对象你想要配置的属性。然后将这个对象赋给 `UIPrintInteractionController` 的 `printInfo` 属性。下面的代码展示了一个样例。

```objc
UIPrintInteractionController *controller = [UIPrintInteractionController sharedPrintController];
UIPrintInfo *printInfo = [UIPrintInfo printInfo];
printInfo.outputType = UIPrintInfoOutputGeneral;
printInfo.jobName = [self.path lastPathComponent];
printInfo.duplex = UIPrintInfoDuplexLongEdge;
controller.printInfo = printInfo;
``` 

`UIPrintInfo` 的一个属性是打印方向：portrait 或 landscape。你也许想要打印方向适应被打印内容的尺寸。换句话说，如果对象很大，并且它的宽大于高，landscape 更适合于它。下面的代码使用一张图片进行了说明。

```objc
UIPrintInteractionController *controller = [UIPrintInteractionController sharedPrintController];
// other code here...
UIPrintInfo *printInfo = [UIPrintInfo printInfo];
UIImage *image = ((UIImageView *)self.view).image;
printInfo.outputType = UIPrintInfoOutputPhoto;
printInfo.jobName = @"Image from PrintPhoto";
printInfo.duplex = UIPrintInfoDuplexNone;
// only if drawing...
if (!controller.printingItem && image.size.width > image.size.height) {
    printInfo.orientation = UIPrintInfoOrientationLandscape;
}  
```

### Specifying Paper Size, Orientation, and Duplexing Options
默认情况下，UIKit 基于目的打印机和 print job 的输出类型，展示一系列默认页面尺寸给打印内容，输出类型由 `UIPrintInfo` 对象的 `outputType` 属性指定。

例如，如果输出类型为 `UIPrintInfoOutputPhoto` 的话，默认的页面尺寸是 4 x 6 英寸，A6，或其它标准大小，依赖于语言环境。如果输出类型是 `UIPrintInfoOutputGeneral` 或 `UIPrintInfoOutputGrayscale`，默认的页面尺寸是 US letter ( 8 1/2 x 11 英寸)， A4 或其它标准大小，依赖于语言环境。

对于大部分应用，这些默认页面尺寸大小是可以接受的。然而，一些应用可能需要一个特殊的页面大小。一个基于页面的应用需要展示用户内容实际上将要怎么显示在给定大小的页面上，一个生产小册子或问候卡的应用可能有它自己喜欢的尺寸，等等。

在这种情况下，print interaction controller delegate 可以实现 `UIPrintInteractionControllerDelegate` 协议方法 `printInteractionController:choosePaper:` 来返回一个 `UIPrintPaper` 对象来表示可用的页面大小和给定内容大小的可打印区域。

delegate 可以采用两种方式。它可以检测传入的 `UIPrintPaper` 对象数组，找到最合适的那一个。或者可以让系统调用 `UIPrintPaper` 类方法 `bestPaperForPageSize:withPapersFromArray:` 来选择最合适的对象。下面的代码展示了这个方法的一个实现，这个应用支持多种文档类型，每个有它自己的页面大小。

```objc
- (UIPrintPaper *)printInteractionController:(UIPrintInteractionController *)pic
    choosePaper:(NSArray *)paperList {
    // custom method & properties...
    CGSize pageSize = [self pageSizeForDocumentType:self.document.type];
    return [UIPrintPaper bestPaperForPageSize:pageSize
        withPapersFromArray:paperList];
}
```

通常，使用自定义 page renderers 的应用使得页面的尺寸成为了计算一个 print job 页面数的因子。

如果你需要向用户显示页面大小 (对于一个文字处理应用，例如)，你必须自己实现这个 UI，并且你必须在你的 `printInteractionController:choosePaper:` 实现中使用页面大小。例如：

```objc
// Create a custom CGSize for 8.5" x 11" paper.
CGSize custompapersize = CGSizeMake(8.5 * 72.0, 11.0 * 72.0);
```

`UIPrintInfo` 类也让你提供额外的设置，如打印方向，选择的打印机，和 duplexing 模式 (如果打印机支持 duplex printing) 。用户可以改变你选择的打印机和 duplex 设置。

### Integrating Printing Into Your User Interface
有两种方式来集成打印到你的用户界面中：

* 使用一个 print interaction controller
* 从一个 activity sheet 中 (iOS 6.0 起)

使用哪种技术来集成打印依赖于你的选择。

#### Presenting Printing Options Using a Print Interaction Controller
`UIPrintInteractionController` 申明了三个方法来显示打印选项给用户，每个有它自己的动画：

* `presentFromBarButtonItem:animated:completionHandler:` 从导航栏或工具栏的一个按钮上动画显示一个 popover
* `presentFromRect:inView:animated:completionHandler:` 从应用的视图中的任意矩形框处动画显示一个 popover
* `presentAnimated:completionHandler:` 从底部滑入页面滑入屏幕

前两个方法是为了在 iPad 设备上调用；第三个方法是为了在 iPhone 和 iPod 设备上调用。你可以根据条件为每种设备编码 (或，user-interface idiom) 通过调用 `UI_USER_INTERFACE_IDIOM` 并与 `UIUserInterfaceIdiomPad` 和 `UIUserInterfaceIdiomPhone` 比较结果。下面的代码展示了一个例子：

```objc
if (UI_USER_INTERFACE_IDIOM() == UIUserInterfaceIdiomPad) {
    [controller presentFromBarButtonItem:self.printButton animated:YES
        completionHandler:completionHandler];
} else {
    [controller presentAnimated:YES completionHandler:completionHandler];
}
```

如果你的应用在 iPhone 上调用 iPad 特定的方法，默认的行为是在一页 sheet 上展示打印选项，sheet 页从屏幕下方滑出。如果你的应用在 iPad 上调用 iPhone 特定的方法的话，默认的行为是从当前的 window 的 frame 动画显示这个 popover 视图。

如果打印选项页已经显示了，你仍调用 `present...` 方法之一的话，`UIPrintInteractionController` 会隐藏打印选项视图或 sheet。你必须再次调用这个方法来显示这些选项。

如果赋一个 printer ID 或一个 duplex mode 作为 print-info 值，这些会显示为打印选项的默认值。(为了 duplex 空间显示，打印机必须支持这个功能) 如果你想要让你的用户选择打印页面范围，你应该设置 `UIPrintInteractionController` 对象的 `showsPageRange` 值为 `YES` ( `NO` 为默认值)。要知道，尽管没有如果你通过 `printingItems` 或总的打印页数为 1 的话，即使 `showsPageRange` 为 `YES`，也不会显示页面范围选择控件。

如果想要打印 UI 在一个特定的视图中显示的话，你可以通过实现 `UIPrintInteractionControllerDelegate` 中的方法 `printInteractionControllerParentViewController:`。这个方法应该返回一个作为 print interaction controller 父 view controller 的 controller。如果提供的 view controller 是一个 `UINavigationController` 实例，打印 UI 被压入。对于其他类型的 view controller，printing navigation 以一个 modal  dialog 在指定的视图中显示。

#### Printing From the Activity Sheet
如果你的应用使用一个 activity sheet (iOS 6 起)，你可以从 activity sheet 中选择打印。从 activity sheet 打印比使用一个 print interaction printer 打印要简单些。但有两点需要申明：

* 应用不能用户是否能选择一个页面范围
* 应用不能通过 delegate 来重写行为，如手动覆盖页面大小选择过程

要使用这个技术，应用必须创建一个 activity items 数组，包含下面内容：

* 一个 `UIPrintInfo` 对象
* 一个 page renderer  formatter 对象，一个 page renderer，一个 printable item。三者之一
* 其他任何对于你的应用合适的 activity items

你可以调用 `initWithActivityItems:applicationActivities:`，传递这个数组第一个参数，nil 作为第二个参数 (或一组自定义 activities，如果你的应用提供了的话)。

最后，你使用标准的 view controller 的 `present*` 方法 prensent 这个 activity view。如果用户选择打印的话， iOS 为你创建一个 print job。更多的信息可以参见 *UIActivityViewController Class Reference* 和 *UIActivity Class Reference *。

### Responding to Print-Job Completion and Errors
`UIPrintInteractionController` 的 presentation 方法声明的最后一个参数，在本文的 *PresentingPrintingOptionsUsingaPrintInteractionController* 中有描述，是一个 completion handler。它的类型是 `UIPrintInteractionCompletionHandler`，在一个 print job 成功完成或因为错误结束的时候调用。你可以提供一个 block 实现来清理你给打印工作设置的状态。也可以用来记录错误。

下面的代码清理一些内容属性，如果有错误，做日志记录。

```objc
void (^completionHandler)(UIPrintInteractionController *, BOOL, NSError *) =
        ^(UIPrintInteractionController *pic, BOOL completed, NSError *error) {
    self.content = nil;
    if (!completed && error)
};
NSLog(@"FAILED! due to error in domain %@ with error code %u",
    error.domain, error.code);
};
```

UIKit 自动释放所有赋给 `UIPrintInteractionController` 实例的对象在打印工作结束的时候 (除了 delegate 对象)，所以你不需要在 completion handler 做这些。

一个打印错误通过一个 error domain 为 `UIPrintErrorDomain` 的 `NSError` 实例来表示，相应的 error code 在 `UIPrintError.h` 中有申明。大部分时候，这些代码表示着变成错误，所以通常没有需要来通知用户。然而，一些可以是因为试着去打印一个文件引起的。

--- 
# Appendix A: Improving Drawing Performance
绘制在任何平台都是一个相对昂贵的操作，优化你的绘制代码总是你开发过程中一个重要的步骤。下表一些保证你的绘制代码尽可能优化的提示。除了这些提示外，你总是应该使用性能工具来测试你的代码，移除人点和冗余。

Tip | Action
----- | ---------
Draw minially | 对每次的 update cycle 中，你应该只更新视图中发生改变的部分。如果你使用 `UIView` 的 `drawRect:` 方法来做你的绘制，使用传入的更新的 rectangle 来限制你绘制的范围。对于 OpenGL 绘制，你必须要自己追踪更新
明智的调用 `setNeedsDisplay:` | 如果你调用 `setNeedsDisplay:`，总是花时间计算真实需要重绘的区域。不要只是传入一个包含真个视图的 rectangle。<br />同样，除非你真的需要重绘内容，否则不要调用 `setNeedsDisplay:`，如果内容没有发生改变，不要重回它。
如果视图是不透明的，标记它为 opaque | compositing 一个内容不透明的视图比一个部分透明的视图要简单很多。使一个视图为 opaque，视图的内容必须不包含任何透明部分，视图的 opaque 属性必须设置为 `YES`
在滑动过程中重用 table cells | 滑动过程中创建新的视图应该尽力避免。花费时间去创建新视图减少了更新屏幕可用的时间，这会导致不均衡的滑动行为。
通过修改当前 transformation matrix 来重用 paths | 通过修改 current transformation matrix，你可以使用一个单一 path 再屏幕的不同部分绘制内容。更多信息，可以查看本文的 *Using Coordinate Transforms to Improve Drawing Performance*
在滑动避免清除之前的内容 | 默认情况下，UIKit 会清除调用一个视图的当前 context buffer 在调用它的 `drawRect:` 方法来更新这片区域的时候。如果你在你的视图中响应滑动事件，重复的清除这个区域在滑动更新中可能是很昂贵的。要关闭这个行为的话，你可以改变 `clearsContextBeforeDrawing` 属性的值为 `NO`
在绘制过程中最小化 graphics state 改变 | 改变 graphics state 需要潜在的 graphics 子系统的工作。如果你需要使用相似的状态信息绘制内容，试着一起绘制这些内容来减少所需的状态改变
使用 Instruments 来调试你的性能 | Core Animation instruments 可以帮助你标识应用中的绘制性能问题。特别是: <ul><li>Flash Updated Regions 使得查看你视图的那部分有更新变得容易</li><li>Color Misaligned Images 帮助你查看糟糕对齐的图片，糟糕对齐的图片导致了 fuzzy images 和差的性能</li></ul> 更多的信息可以查看 *Instruments User Guide* 的 *Measuring Graphics Performance in Your iOS Device*

----
# Appendix B: Supporting High-Resolution Screens In Views
自 iOS 4.0 起应用需要准备好跑在不同屏幕分辨率的设备上。幸运的是，iOS 使得支持不同屏幕分辨率变得简单。大部分处理不同类型屏幕的工作已经被系统框架所处理。然而，你的应用仍需要做些工作来更新 raster-based images，并且你也许想要做些额外的工作来利用额外的像素，当然这依赖于你的应用。

查看本文的 *Points Versus Pixels* 来了解关于这个主题的更多背景信息。

## Checklist for Supporting High-Resolution Screens
要为了高分比率的设备更新你的应用，你需要做以下事情：

* 为应用 bundle 中的每个图片资源提供一个高分辨率的图片，像下文 *Updating Your Image Resource Files* 所描述的那样。
* 提供高分辨率的 app 和 document icons，如下文 *Updating Your App’s Icons and Launch Images* 中所描述的那样
* 对于基于矢量的形状和内容，继续使用你的自定 Core Graphics 和 UIKit 绘制代码。如果你想要添加额外的详细的内容的话，查看本文的 *Points Versus Pixels* 来了解怎么做的信息。
* 如果你使用 OpenGL ES 来绘制，决定你是否想要支持高分辨率绘制，并相应的设置你的 layer 的 scale facotr。见下文 *Drawing High-Resolution Content Using OpenGL ES or GLKit*。
* 关于你创建的自定义图片，修改你的图片创建代码，将当前 scale factor 考虑进来。具体可以查看本文的 *Drawing to Bitmap Contexts and PDF Contexts*
* 如果你的应用使用 Core Animation，根据需要调整你的代码来调用 scale factor 的不同。具体可以查看上文的 *Accounting for Scale Factors in Core Animation Layers*

## Drawing Improvements That You Get for Free
iOS 提供的绘制技术提供了许多支持来帮助你，使得你渲染的内容看起来不错，不管潜在屏幕的分辨率是怎样的：

* 标准的 UIKit 视图 (text views, buttons, table views 等等) 自动在不同分辨率的情况正确的渲染。
* 基于矢量的内容 (`UIBezierPath`, `CGPathRef`, PDF) 自动利用额外的像素来渲染形状的线条。
* 文本自动在高分辨率下渲染
* UIKit 支持自动加载你的图片的高分比率变种

如果你的应用只使用 native 的绘制技术来渲染，你支持高分辨率屏幕唯一需要做的是提供你图片的高分辨率变种。

## Updating Your Image Resource Files
跑在 iOS 4 中的应用对于每个图片资源应该有两个单独的文件。一个文件提供标准分辨率的版本，一个提供同一张图片的高分率的版本。每对文件的命名规范如下：

* 标准：*<ImageName><device_modifier>.<filename_extension>*
* 高分辨率：*<ImageName>@2x<device_modifier>.<filename_extension>*

每个名字的 *<ImageName>* 和 *<filename_extension>* 部分指定了文件的可视名字和后缀。*<device_modifier>* 部分是可以选的，要么包含 `~ipad` 或 `~iphone`。你包含这些修饰符之一，当你想要为 iPad 和 iPhone 指定不同版本的的图片时。对于高分辨率的 @2x 修饰符是新加的，让系统知道这张图片是标准图片的高分辨率变种。

> **重要**: 修饰符的顺序很关键。如果你不正确的将 @2x 放在设备修饰符的后面的话，iOS 将找不到这个图片。

当创建高分辨率图片时，将新的版本放在与老版本同样的位置。

### Loading Images into Your App
`UIImage` 类处理加载高分辨率图片到应用所需的所有工作。当创建新的 image 对象时，你使用同样的名字来获取你图片的标准和高分辨率的版本。例如，如果你有两张图片文件，名字为 `Button.png` 和 `Button@2x.png`，你将使用下面的代码来获取按钮的图片：

```objc
UIImage *anImage = [UIImage imageNamed:@"Button"];
```

> **注意**: 自 iOS 4 起，你在指定图片名字的时候你可以忽略文件扩展

在高分辨率的设备上，`imageNamed:`，`imageWithContentsOfFile:`，和 `initWithContentsOfFile:` 方法自动查找一个在文件名字中有 *@2x* 修饰符的版本。如果它找到一个，它会加载这张图片。如果你不提供一个高分辨率版本的图片，这个图片对象仍会加载标准分辨率的图片 (如果有的话) 并在绘制的时候拉伸它。

当它加载一张图片的时候，一个 `UIImage` 对象基于图片文件的后缀自动设置它的 `size` 和 `scale` 属性到合适的值。对于标准分辨率的图片，它设置 `scale` 属性为 1.0 并设置图片对象的 size 为图片的像素尺寸。对于文件名字中有 @2x 后缀的图片，它设置 `scale` 属性值为 2.0，并将宽高值减半来抵消 scale factor。这些减半的值正确的与在 logical coordinate space 中你使用来渲染图片的基于 point 的尺寸相关联。

> **注意**: 如果你使用 Core Graphics 来创建一个图片，记住 Quartz 图片没有一个显式的 scale factor，所以它们的 scale factor 假设为 1.0。如果你想要从一个 `CGImageRef` 数据类型创建一个 `UIImage` 对象，使用 `initWithCGImage:scale:orientation:` 来做这些。这个方法允许你关联一个指定的 scale factor 到你的 Quartz image 数据。

一个 `UIImage` 对象在绘制的过程中自定考虑到它的 scale factor。因此，任何你用于渲染图片的代码应该是一样工作的只要你在应用的 bundle 中提供了正确的图片资源。

### Using an Image View to Display Multiple Images
如果应用为了高亮或动画使用 `UIImageView` 类来显示多张图片的话，你赋给这个视图所有的图片必须有相同的 scale factor。你可以使用 image view 来显示单个图片或对多张图片做动画，你也可以提供一个高亮图片。因此，如果你为其中一个图片提供了高分辨率的版本，那么你必须为所有都提供高分辨率的版本。

### Updating Your App’s Icons and Launch Images
除了更新你的应用的自定义图片资源外，你也应该提供新的好分辨率 icon 给应用的 icon 和 launch images。更新这些图片资源的过程和更新所有其他图片资源的过程一样。创建这个图片的一个新版本，添加 @2x 修饰符到相应的图片文件名中，跟之前对待图片样对待。例如，对于应用图标，添加高分辨率图片文件名到应用的 `Info.plist` 文件的 `CFBundleIconFiles` key 中。

指定应用 icons 和 launch images 的更多信息，可以查看 *App Programming Guide for iOS* 的 *App-Related Resources*。

## Drawing High-Resolution Content Using OpenGL ES or GLKit
如果你的应用使用 OpenGL ES 或 GLKit 来绘制的话，你现有的绘制代码应该不需要任何改变就可以继续工作。当在一个高分辨率屏幕上绘制的时候，但是你的内容会自动拉伸，看上去更显块状。块状外表的原因是 `CAEAGLLayer` 类的默认行为，你使用这种行为来支撑你的 OpenGL ES renderbuffers (直接或间接的)，跟其它的 Core Animation layer 对象是一样的。换句话说， 它的 scale factor 初始被设置为 1.0，这会导致 Core Animation 在高分辨率上的屏幕上拉伸 layer 的内容。为了避免 blocky appearance，你需要增加你的 OpenGL ES renderbuffers 的大小来匹配屏幕的大小。(有更多的像素，你可以给你的内容提供更多的具体信息) 因为添加更多的像素到你的 renderbuffers 有些性能隐患，但是，你必须显示的支持高分辨率屏幕。

要启用高分辨率绘制，你必须改变你用来显示你的 OpenGL ES 或 GLKit 内容的视图的 scale factor。将视图的 `contentScaleFactor` 从 1.0 变为 2.0 会触发潜在 `CAEAGLLayer` 对象的 scale factor。你使用来绑定 layer 对象到你的 renderbuffers 的方法 `renderbufferStorage:fromDrawable:` 通过将 layer 的边框乘以它的 scale factor 计算 render buffer 的大小。因此将 scale factor 加倍后会将结果的 render buffer 的长和宽都加倍，给你为你的内容提供了更多的像素。在那之后，由你来为这些额外的像素提供内容。

下面的代码展示合适的方式来绑定你的 layer 对象到你的 renderbuffers，并获取结果大小信息。如果你使用 OpenGL ES 应用模板来创建你的代码，那么这步已经为你做了，你唯一需要做的是正确的设置你视图的 scale factor。如果你不使用 OpenGL ES app 模板，你应该使用与这相似的代码来获取 render buffer 的大小。你绝不应该假设 render buffer 的大小是固定的对于一个给定类型的设备。

```objc
GLuint colorRenderbuffer;
glGenRenderbuffersOES(1, &colorRenderbuffer);
glBindRenderbufferOES(GL_RENDERBUFFER_OES, colorRenderbuffer);
[myContext renderbufferStorage:GL_RENDERBUFFER_OES fromDrawable:myEAGLLayer];

// Get the renderbuffer size.
GLint width;
GLint height;
glGetRenderbufferParameterivOES(GL_RENDERBUFFER_OES, GL_RENDERBUFFER_WIDTH_OES, &width);
glGetRenderbufferParameterivOES(GL_RENDERBUFFER_OES, GL_RENDERBUFFER_HEIGHT_OES, &height);
```

> **重要**: 一个视图由 `CAEAGLLayer` 对象支撑的视图不应该实现一个自定义 `drawRect:` 方法。实现 `drawRect:` 方法会使得系统改变视图的默认 scale factor，从而导致它与屏幕的 scale factor 相匹配。如果你的绘制代码不期望这种行为，你的应用内容将不会正确的渲染。

如果你选择支持高分辨率绘制，你也需要调整 mode 和 texture assets。例如，当跑在 iPad 或在一个高分辨率的设备上，你也许想要选择更大的 models 和更具体的 textures 来利用增加的像素。相反，在一个标准分辨率的 iPhone 上，你可以继续使用更小的 models 和 textures。

当决定是否支持高分辨率内容的一个重要因子是性能。发生在你将 layer 的 scale factor 从 1.0 变为 2.0 的四倍化像素给 fragment processor 增加了额外的压力。如果你的应用进行许多 per-fragment 的计算，像素的增加可能会降低应用的帧率。如果你发现你的应用在高 scale factor 时跑得很慢的话，考虑以下选项：

* 根据 *OpenGL ES Programming Guide for iOS * 中的性能优化指导来优化你的 fragment shader 的性能。
* 在你的 fragment shader 中选择一个简单算法来实实现。通过这么做，你可以减少单个像素的质量来在高分辨率下渲染整个图片。
* 使用一个介于 1.0 到 2.0 的 scale factor。一个 1.5 的 scale factor 提供了比 1.0 更好的质量但比 2.0 填充更少的像素。
* iOS 4 起 OpenGL ES 提供了 multisampling 的选项。即使你的应用可以使用一个更小的 scale factor (甚至是 1.0)，不论如何实现 multisampling。这个技术额外的一个的好处是在不知道高分辨率的屏幕上也可以提供高质量。

最好的解决方案依赖于你的 OpenGL ES 应用的需求； 你应该测试这些选项中的多个，并选择在图片和性能间提供最好平衡的方案。

# Appendix C: Loading Images
为了美学和功能的原因，图片是应用界面中的一个普遍元素。它们可以是一个应用的一个关键不同因素。

应用使用的大部分图片，包括 launch images 和 应用 icons，以文件的形式存在于应用 main bundle 中。你可以有特定设备的 launch images 和 icons，并为高分辨率屏优化。你可以在 *App Programming Guide for iOS* 的 *Advanced App Tricks and App-Related Resources* 中找到关于这些图片文件的完整描述。本文的 *Updating Your Image Resource Files* 讨论了使得你的图片文件与高分辨率屏幕兼容的调整。

另外，iOS 提供了使用 UIKit 和 Core Graphics 框架加载和显示图片的支持。你怎么决定使用哪一个类和方法来绘制图片依赖于你想要怎样使用它们。可能的话，推荐你使用 UIKit 中的类来在代码中表示图片。下表列出一些使用情况和处理它们所推荐的方式。

Scenario | Recommended usage 
------------- | -----------------------------
显示图片作为视图的内容 | 使用 `UIImageView` 来显示图片，这个选择假设你视图的内容只有图片。你仍然可以层叠其他的视图到 image view 上来绘制额外的控件或内容
显示一张图片作为一个视图一部分的装饰 | 使用 `UIImage` 加载绘制这张图片
保存一些位图数据到一个图片对象 | 你可以本文的 *Creating New Images Using Bitmap Graphics Contexts* 中描述的 UIKit 函数或 Core Graphics 函数
保存一张图片为 JPEG 或 PNG 文件 | 从原始的图片数据中创建一个 `UIImage` 对象。调用 `UIImageJPEGRepresentation` 或 `UIImagePNGRepresentation` 方法来获取一个 `NSData` 对象，使用这个对象的方法来保存数据到一个文件。

## System Support for Images
iOS 的 UIKit 框架和底层的系统框架提供了创建，访问，绘制，编写和操作图片的广泛可能。

### UIKit Image Classes and Functions
UIKit 框架有 3 个类和一个协议以某种方式与图片相关联：

<dl>
	<dt> UIImage</dt>
	<dd>这个类代表了 UIKit 框架中的图片。你可以从几个不同源创建它们，包括文件和 Quartz 图片对象。这个类的方法使你能使用不同的 blend modes 和 opacity 值来绘制到 current graphics context <br /> `UIImage` 类自动为你处理所需的 transformation，如应用合适的 scale factor 当给定 Quartz 图片时，修改图片的坐标系统以便与 UIKit 的坐标系吻合</dd>
	<dt>UIImageView</dt>
	<dd> 这个类的对象是一个视图，要么显示一张图片，要么动画显示一系列的图片。如果一张图片是视图的唯一内容，使用 `UIImageView` 类而不是绘制这张图片</dd>
	<dt>UIImagePickerController 和 UIImagePickerControllerDelegate</dt>
	<dd>这个类和协议给你的应用一种方式来获取用户提供的图片和视频。这个类显示和管理用户界面来选择和拍照和视频。当用户选择一张图片时，它传递选择的图片对象给这个 delegate</dd>
</dl>

除了这些类外，UIKit 申明了你可以调用来进行一些列与图片相关的任务：

* **Drawing into an image-backed graphics context**。`UIGraphicsBeginImageContext` 函数创建一个 offscreen bitmap graphics context。你可以绘制到这个 graphics context，然后从中获取一张图片。查看本文的 *Drawing Images* 了解更多信息
* **Getting or caching image data**。每个 `UIImage` 对象有一个 backing Core Graphics 图片对象 (`CGImageRef`)，你可以直接访问这个对象。你可以传递这个 Core Graphics 对象给 Image I/O 框架来保存这个数据。你也可以转变这个 `UIImage` 对象中的图片数据到一个 PNG 或者 JPEG 格式，通过调用 `UIImagePNGRepresentation` 或 `UIImageJPEGRepresentation` 函数。你可以方位这个 data 对象的中字节，你也可以将图片数据写入到一个文件。
* **Writing an image to the Photo Album on a device**。调用 `UIImageWriteToSavedPhotosAlbum` 函数写入一个 `UIImage` 对象到一个设备的 Photo Album。

*Drawing Images* 指明了什么时候你应该使用这些 UIKit 类和函数。

### Other Image-Related Frameworks
要创建、访问、修改和写入图片，除了 UIKit 你可以使用其他的几个系统框架。如果你发现你使用 UIKit 方法或函数不能完成一个特定图片相关的任务的话，这些底层框架中的一个的某个函数也许能够做到你想做的。这些函数的一些页面需要 Core Graphics 图片对象。 你可以通过 `CGImage` 属性访问支撑 `UIImage` 对象的 `CGImageRef` 对象。

> **注意**: 如果一个 UIKit 方法或函数存在来帮你完成一个给定的图片相关的任务的话，你应该使用它而不是相应的底层函数。

Quartz 的 Core Graphics 框架是底层系统框架中最重要的一个。它的一些函数与 UIKit 函数和方法是对应的；例如，一些 Core Graphics 函数允许你创建和绘制到 bitmap graphics context，其他的让你从不同的源创建图片。然而，Core Graphics 为操作图片提供了更多的选项。使用 Core Graphics 你可以创建和应用 image masks，从现有图片的部分创建图片，应用 color spaces，和访问一些图片属性，包括每行字节数，每像素字节数，和渲染深度。

Image I/O 框架是和 Core Graphics 紧密相连的。它允许一个应用读取或写入大部分图片文件格式，包括标准的 web 格式，high dynamic range 图片，和 raw camera 数据。这个框架有快速编码和解码的特点，图片元数据和图片缓存的特点。

Assets Library 是一个允许应用访问 *Photos* 管理的 assets 的框架。你要么通过 representation (例如，PNG 或 JPEG) 获取一个 asset，要么通过 URL。从 representation 或 URL你可以获取一个 Core Graphics 图片对象或 raw image 数据。这个框架也让你写入图片到这个 Saved Photos Album。

### Supported Image Formats
下表列处理 iOS 直接支持的图片格式。在这些格式中，PNG 格式是应用中最推荐使用的。通常，UIKit 支持的图片格式和 Image I/O 框架支持的格式是一样的。

Format | Filename extensions
---------- | --------------------------
Portable Network Graphic (PNG) | .png
Tagged Image File Format | .tiff 或 .tif
Joint Photographic Experts Group (JPEG) | .jpeg 或 .jpg
Graphic Interchange Format (GIF) | .gif
Windows Bitmap Format (DIB) | .bmp 或 .BMPf
Windows Icon Format | .ico
Windows Cursor | .cur
XWindow bitmap | .xbm

## Maintaining Image Quality
为你用户界面提供高质量的图片应该是你的设计中主要的事。图片提供了一种合理并有效的方式来展示复杂的图形，合适的时候应该使用。当为你的应用创建图片时，记住以下引导：

* **Use the PNG format for images**。PNG 格式提供了无损的图片内容，这意味着保存图片数据到一个 PNG 格式，然后尝试读回它将得到完全一样的像素值。PNG 同样有一种优化存储方式，这种方式被设计来更快的读取图片数据。它也是 iOS 偏爱的图片格式。
* **Create images so that they do not need resizing**。如果你计划使用一个指定大小的图片，确保创建相应尺寸的图片资源。不要创建一个过大的图片然后缩小它来适应视图，因为拉伸要求额外的 CPU cycles 和填补。如果你需要在可变的尺寸上显示图片，包含这张图片在不同尺寸的不同版本，缩小与目的 size 接近的图片。
* **Remove alpha channels from opaque PNG files**。如果一个 PNG 图片的每个像素是不透明的，移除图片的 alpha channel 避免了 blend 包含图片的 layers。这样相当大的简化了组合 (compositing)，并提供了绘制性能。

