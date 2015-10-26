# Introduction
Quartz 2D 是一个高级的二维绘制引擎，可用于 iOS 应用开发和 Mac OS X 内核外的应用环境。Quartz 2D 提供一个拥有无与伦比的保真度的轻量级的底层 2D 渲染，这是不论显示或打印设备的。Quartz 2D 不依赖于分辨率和设备。当你使用 Quartz 2D API 来绘制的时候你不需要考虑最终的目的地。

Quartz 2D API 很容易使用，并提供了 access to transparency layers, path-baed drawing, offscreen rendering, advanced color management, anti-aliased rendering, and PDF document creation, display, and parsing.

Quartz 2D API 是 Core Graphics 框架的一部分，所以你也会看打动 Quartz 被认为是 Core Graphics，或简单的说，CG.

## Who Should Read This Document?
这篇文档是给需要进行一下任何的 iOS 和 Mac OS X 开发者：

* 绘制图形
* 在一个应用中提供编辑图形的能力
* 创建或显示位图图片
* 需要同 PDF 一起工作

## Organization of This Document
这篇文档被组织成以下章节：

* **Overview of Quartz 2D** 描述了 page, drawing destinations, Quartz opaque data types, graphics states, coordinates, and memory management, 并看了下 Quartz 是怎么工作的。 
* **Graphics Contexts** 描述了绘制目的地的类型并提供了创建不同类型 graphics contexts 的步骤。
* **Paths** 讨论了基本组成 paths 的基本元素，说明了怎么创建和绘制它们，也说明了怎么设置一个 clipping area，并解释了 blend mode 是怎么影响绘制的。
* **Color and Color Spaces** 讨论了颜色的值和使用 alpha 值来调控透明度，并描述了怎么创建一个 color space，设置颜色，创建颜色对象，和设置 rendering intent。
* **Transforms** 描述了当前的 transformation matrix，并解释了怎样来修改它，展示了怎么设置 affine transform，展示了怎么转换 user 和 device space，并提供了关于 Quartz 进行的数学操作的背景信息。
* **Patterns** 定义了一个 pattern 是什么和它的组成部分，论述了 Quartz 怎么渲染它们，展示了怎么创建 colored 和 stenciled patterns
* **Gradients** 讨论了轴向的和辐射状的 gradients，并展示了怎么创建和使用 `CGShading` 和 `CGGradient` 对象。
* **Transparency Layers** 示例 transparency layers 看起来是怎样的，讨论了它们是怎么工作的，并提供了实现它们的步骤。
* **Data Management in Quartz 2D** 讨论了怎么移入或移出 Quartz
* **Bitmap Images and Image Masks** 介绍了一个 bitmap image 的定义，展示了怎么使用 bitmap image 作为一个 Quartz 绘制原语。它也描述了你可以使用于图片的 masking 的技巧，并展示了在绘制图片时通过使用 blend modes 你可以获得不同效果。
* **Core Graphics Layer Drawing** 描述了怎么创建和使用 drawing  layers 来达成高性能的 patterned 绘制或来做离屏渲染
* **PDF Document Creation, Viewing, and Transforming** 展示了怎么打开和查看 PDF 文档，给它们应用 transforms，创建一个 PDF 文件，访问 PDF 元数据，添加链接，和添加安全特性 (如密码保护)。
* **PDF Document Parsing** 描述了怎么使用 `CGPDFScanner` 和 `CGPDFContentStream` 对象来解析和检测 PDF 文档。
* **PostScript Conversion** 概述了 Mac OS X 中你可以用来转换一个 Postfile 到一个 PDF 文件的函数。这些函数在 iOS 中不可用。
* **Text** 描述了 Quartz 2D 对于 text 和 glyphs 的底层支持，同时也提供了 Unicode 和高级的 text 支持。它也讨论了怎么复制 font 变种。
* **Glossary** 定义了这篇文档中使用的专用名词。

## See Also
要使用 Quartz 2D 下面的这些也很有必要阅读：

* *Quartz 2D Reference Collection*， 这篇文档以头文件的方式组织，提供了 Quartz 2D API 的完整参考
* *Color Management Overview* 简单的介绍了颜色感知的原则，颜色空间，和颜色管理系统。
* Mailing lists. 加入 **quartz-dev** 邮件列表来讨论使用 Quartz 2D 时遇到的问题。
* **Programming With Quartz: 2D and PDF Graphics in Mac OS X** 提供了关于使用 Quartz 的深度信息。这本书目前是通过  Mac OS X v10.4 写的，并且是 iOS 问世之前写的。这本书包含怎么支持 OS X 之前版本和怎么使用 v10.4 引入的特性的样例。与书关联的样例代码可从出版商那里获取。

---
# Overview of Quartz 2D
Quartz 2D 是一个 2D 绘制引擎，可用于 iOS 环境和 OS X 中内核外的应用空间。你可以使用 Quartz 2D API 来访问诸如 path-based drawing, painting with transparency, sahding, drawing shadows, transparency layers, color management,  anti-aliased rendering, PDF document generation,  and PDF metadata access. 只要有可能，Quartz 2D 就会使用图形硬件的能力。

在 OS X 中，Quartz 2D 可以和所有其它图形技术—— Core Image, Core Video, OpenGL, and QuickTime——一起工作。使用 QuickTime 函数 `GraphicsImportCreateCGImage` 可以从一个 QuickTime graphics importer 中创建一个 Quartz 中的图片。查看 *QuickTime Framework Reference* 来了解具体信息。本文的 **“Moving Data Between Quartz 2D and Core Image in Mac OS X** 有描述你怎么来提供图片给 Core Image——它是一个支持图片处理的框架。

同样，在 iOS 中，Quartz 2D 可以和可用的图形和动画技术一起使用，如 Core Animation, OpenGL ES, 和 UIKit 类。

## The Page
Quartz 2D 使用 **painter’s model** 来做图。在 *painter’s model* 中，每个后续的绘制操作会应用一层 "paint" 到一个输出的 "画布"，这个 "画布" 常被称作一个 **page**。在 page 上的 paint 可以被后续的绘制操作所覆盖。一个在 page 上绘制的对象除了被覆盖外不能被修改。这中模型允许你从一组数量很小但强大的原语来创建复杂的图片。

下图展示了 painter's model 是怎么工作的。要获取图中上部的图片，左边的形状先被绘制，然后才是实心的形状。实心形状会覆盖第一个形状，第一个形状的边界外都被遮蔽。图中下面的实例绘制顺序刚好相反，实心形状先被绘制。跟你看到的一样，在 painter's model 中绘制的次序很重要。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/the_painters_model.png" width=80% />

Page 也许是一张真实的纸张 (如果输出设备是一个打印机的话); 它也可能是一个虚拟的一页纸 (如果输出设备是一个 PDF 文件的话); 它甚至可能是一个 bitmap image。Page 的准确性质依赖于你使用的特定 graphics context。 

## Drawing Destinations: The Graphics Context
一个 **graphics context** 是一个 opage 数据类型 (`CGContextRef`)，这个类型封装了 Quartz 绘制图片到一个输出设备的信息，如一个 PDF 文件，或显示器上的一个窗口。Graphics context 内的信息包括图形绘制参数和一个 page 上绘制内容的设备特定呈现 (device-specific representation)。Quartz 中所有的对象被绘制到一个 graphics context，或被一个 graphics context 限制。

你可以认为一个 graphics context 是一个绘制目的地，像下图所示。当你使用 Quartz 绘制的时候，所有特定设备的特点被限制在你使用的特定类型的 graphics context。换句话说，你只需要提供一个不同的 graphics context 到相同的绘制 Quartz 绘制代码，你就可以绘制相同的图片到不同的设备。你不需要进行任何设备特定的计算，Quartz 会为你做掉。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/quartz_drawing_destinations.png" width=80% />

对你的应用有以下 graphics context 可用：

* **bitmap graphics context** 允许你绘制 RGB 颜色，CMKY 颜色，或灰度到一个位图。一个 **bitmap** 是一个矩形的像素数组 (raster or array of pixels)，每个像素代表了图中的一点。位图图片也被称为 *sampled images*，更多信息查看本文的 **Creating a Bitmap Graphics Context**。
* **PDF graphics context** 允许你创建一个 PDF 文件。在一个 PDF 文件中，你的绘制以一系列的命令保存起来。在 PDF 文件和 bitmaps 之间有些很大的区别：
	* PDF 文件，不像位图，可能不止不含一页内容
	* 当你在不同的设备上从一个 PDF 文件中绘制一页内容，结果图片会为设备的显示特定而优化。
	* PDF 文件本身是不依赖于分辨率的——它们绘制的大小可以无限的增大或缩小而丢失任何细节。用户接受到的一张位图图片质量与图片被查看的设备分辨率有依赖关系。<br /> 查看本文的 **Creating a PDF Graphics Context** 了解更多信息。
* **window graphics context** 可以被用来绘制到一个窗口。注意因为 Quartz 2D 是一个图形引擎，并不是一个窗口管理系统，你使用应用矿建之一来获取一个 window 的graphics context。查看本文的 **Core Graphics Layer Drawing** 来了解更多的信息。
* **layer context** (`CGLayerRef`) 是一个与另一个 graphics context 相关联的离屏绘制目的地。它被设计来优化绘制 layer 到一个创建这个 layer context 的 graphics context。对于离屏渲染一个 layer context 相对于 bitmap graphics context 而言是一个好得多的选择。查看本文的 **Core Graphics Layer Drawing** 来了解更多的信息。
* 当你想要在  OS X 中打印的时候，你发送你的内容到一个 **PostScript graphics context**，这个 context 被打印框架管理。更多信息查看本文档的 **Obtaining a Graphics Context for Printing**。

## Quartz 2D Opaque Data Types
除 graphics context 外，Quartz 2D 还API 定义了多种多样的不透明数据类型。因为 API 是 Core Graphics 框架的一部分，这些数据类型和操作于它们的例程使用了 **CG** 前缀。

Quartz 2D 从这些数据类型创建对象，应用在这些对象上操作来达到特定目的的绘制输出。下图 (Figure 1-3) 展示了你应用不同的绘制操作到 Quartz 2D 提供的三种对象上得到的不同结果。例如：

* 通过创建一个 PDF page 对象，你可以旋转和显示一个 PDF page，应该一个 rotation 操作给 graphics context，并要求 Quartz 2D 绘制这个 page 到一个 graphics context。
* 你可以通过创建一个 pattern 对象来绘制一个 pattern，定义组成 pattern 的形状，并设置好 Quartz 2D 使用这个 pattern 来绘制到 graphics context。
* 通过创建 shading 对象你可以使用一个轴向或辐射状 shading 来填满一块区域，提供一个决定 shading 中每点颜色的函数，然后要求 Quartz 2D 使用这个 shading 作为填充色。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/opaque_data_types_are_the_basis_of_drawing_primitives_in_quartz_2d.png" width=80% />

Quartz 2D 中可用数据类型如下：

* `CGPathRef`, 被用来为创建矢量图形创建 paths，paths 可以填充或沿线画出 (stroke)，具体查看本文的 **Paths**。
* `CGImageRef, 被用来表示你提供的样品数据的位图图片和 bitmap image masks，更多信息查看本文的 **Bitmap Images and Image Masks**。
* `CGLayerRef` 被用来表示一个可以被用来重复绘制 (如为了背景或 patterns) 或被用来离屏绘制的 drawing layer。查看本文的 **Core Graphics Layer Drawing** 来了解更多信息。
* `CGPatternRef`, 被用于重复绘制。具体查看本文的 **Patterns**。
* `CGShadingRef` 和 `CGGradientRef` 被用来绘制渐变，具体查看本文的 **Gradients**。
* `CGFunctionRef` 被用来定义一个接受任意数量浮点数类型参数的回调函数。当你为一个 shading 创建 gradients 时，你使用这个数据类型。具体查看本文的 **Gradients**。
* `CGColorRef` 和 `CGColorSpaceRef`, 被用来告知 Quartz 怎么翻译颜色。具体查看本文的 **Color and Color Spaces**。
* `CGImageSourceRef` 和 `CGImageDestinationRef` 你用来从 Quartz 移出或移入数据。具体请查看本文的 **Data Management in Quartz 2D** 和 *Image I/O Programming Guide*。
* `CGFontRef`, 被用来绘制文本。具体请查看 **Text**。
* `CGPDFDictionaryRef` `CGPDFObjectRef` `CGPDFPageRef` `CGPDFStream` `CGPDFStringRef` `CGPDFArrayRef` 提供了机制方位 PDF 元数据。具体请查看本文的 **PDF Document Creation, Viewing, and Transforming**。
* `CGPDFScannerRef` 和 `CGPDFContentStreamRef`, 被用来解析 PDF 元数据。具体请查看本文的 **PDF Document Parsing**。
* `CGPSConverterRef` 被用来转换 PostScript 为 PDF。iOS 中不可用。具体请查看本文的 **PostScript Conversion**。

## Graphics States
Quartz 根据 **current graphics state** 中的参数来调整绘制操作最后的结果。Graphics state 包含一些参数，这些参数会作为绘制例程参数。绘制到一个 graphics context 的例程会咨询 graphics state 来决定怎么渲染它们的结果。例如，当你调用一个函数来设置填充颜色时，你在修改存储在当前 graphics state 的一个值。其他常用的 graphics state 元素包括 line width, current position, text font size.

Graphics context 有一个包含 graphics states 的栈。当 Quartz 创建一个 graphics context 的时候，栈是空的。当你保存这个 graphics state，Quarzt 将当前 graphics state 的拷贝压入栈内。当你恢复 graphics state 时，Quartz 会从栈顶弹出这个 graphics state。弹出的 graphics state 成为当前 graphics state。

要保存当前的 graphics state，使用函数 `CGContextSaveGState` 来压入当前 graphics state 的拷贝到栈内。要恢复之前保存的 graphics state，使用函数 `CGContextRestoreGState` 来用栈顶的 graphics state 替换当前的 graphics state。

注意并不是当前绘制环境的所有方卖弄都是 graphics state 的元素。例如，当前的 path 并不认为是 graphics state 的一部分，因此你在调用 `CGContextSaveGState` 函数时，并不会保存。调用这个函数时被保存的 graphics state 参数如下表：

Parameters | Discussed in this chapter
--------------- |--------------------------------
Current transformation matrix (CTM) | "**Transforms**"
Clipping area | **Paths**
Line: width, join, cap, dash, miter limit | **Paths**
Accuracy of curve estimation (flatness) | **Paths**
Anti-aliasing setting | **Graphics Contexts**
Color: fill and stroke settings | **Color and Color Spaces**
Alpha value (transparency) | **Color and Color Spaces**
Rendering intent | **Color and Color Spaces**
Color space: fill and stroke settings | **Color and Color Spaces**
Text: font, font size, character spacing, text drawing mode | **Text**
Blend mode | **Paths** and **Bitmap Images and Image Masks**

## Quartz 2D Coordinate Systems
一个坐标系，如下图所示，定义了位置的范围，被用来表达绘制到页面的对象位置和大小。你可以在 user-space 坐标系中指定图形的位置和大小，简单来说，这个坐标系可称为 **user space**。坐标是通过浮点数来定义的。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/the_quartz_coordinate_system.png" width=80% />

因为不同的设备有不同的图形能力，图形的位置和大小必须以不依赖设备的方式定义。例如，一个屏幕显示设备也许可以显示的能力不超过每英寸 96 像素，然而一个打印机可以每英寸 300 像素。如果你在设备的层面定义坐标系 (在这个例子里，96 或 300 像素), 绘制到这个坐标系的对象在其他设备上不可无损的重新绘制。它们会看上去太大或太小。

Quartz 使用一个独立的坐标系——user space——来完成不依赖设备的，通过映射 user space 到输出设备的坐标系——device space——使用 **current transformation matrix**，或 **CTM**。一个 **matrix** 是一个数学结构用来有效的描述一系列相关的等式。Current transformation matrix 是 matrix 的一种特殊类型，称为 **affine transform**，它会通过应用 **translation**, **rotation**, **scaling** 操作 (移动，旋转，放大缩小一个坐标系的计算) 将一个坐标空间的点映射到另一个。

current transformation matrix 有另外一个目的：它允许你 tranform 对象怎样被绘制。例如，要绘制一个旋转 45° 的盒子，你可以在绘制这个盒子之前，旋转页面 (the CTM) 的坐标系。Quartz 使用旋转的坐标系绘制到输出设备。

在用户空间的点是通过 (x, y) 坐标对来表示的，x 表示水平轴方向的位置，y 表示竖直轴方向的位置。用户坐标空间的 **origin** 在 (0, 0)。原点在页面的左下角，如图 Figure 1-4 所示。在 Quartz 默认的坐标系中，x 轴是从左向右递增的，y 轴是从下向上递增的。

一些技术使用不同于 Quartz 坐标系来设置它们的 graphics context。相对于 Quartz，这样的坐标系是一个 **modified coordinate system** 而且在进行一些 Quartz 绘制操作时必须被抵消掉。最常用的 modified coordinate system 将原点置于 context 的右上角并将 y 轴的方向指向页面的底部。下面的一些地方你会遇到这种特殊的坐标空间：

* 在 OS X 中，一个覆盖了 `isFlipped` 方法并返回 `YES` 的 `NSView` 的子类。
* 在 iOS 中，`UIView` 返回的绘制 context
* 在 iOS 中，通过调用 `UIGraphicsBeginImageContextWithOptions` 函数返回的绘制 context

UIKit 返回一个 modified coordinate system 的 Quartz 绘制 context 的原因是 UIKit 使用了一个不同的默认坐标约定；UIKit 应用这个 transform 到它创建的 Quartz  context，以便它们与这个约定匹配。如果你的应用想要使用相同的绘制例程同时绘制到一个 `UIView` 对象和一个 PDF graphics context (通过 Quartz 创建并使用默认的坐标系)，你需要应用一个 transform 以便 PDF graphics context 使用相同的 modified coordinate system。要这么做的话，应该一个 transform 移动原点到 PDF context 的左上角，然后缩放 y 坐标 -1 倍。

使用一个缩放 transform 来对 y 轴坐标取负值，这样就改变了在 Quartz 绘制中的一些约定。例如，如果调用 `CGContextDrawImage` 来绘制一个图片到 context，这个图片在绘制到目的地时候被 transform 所修改。同样，绘制 path 的例程接受参数来指定一个弧线在默认坐标系中是顺时针方向还是逆时针方向。如果一个坐标系被修改了，结果也会被修改，就像镜子中的图片。Figure 1-5 中，传递相同的参数给 Quartz ，结果是在默认坐标系中是一个顺时针方向，在 y 轴取负之后图像就成了一个逆时针的。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/modifying_the_coordinate_system_creates_a_mirrored_image.png" width=80% />

你的应用负责调整调整那些使用一个有 transform 应用到它的 context 的 Quartz 调用。例如，如果你想要一张图片或 PDF 正确的绘制到一个 graphics context，你的应用可能需要暂时调整 graphics context 的 CTM。在 iOS 中，如果使用一个 `UIImage` 对象来包含一个你创建的 `CGImage` 对象， 你不需要修改这个 CTM。`UIImage` 自动抵消 UIKit 使用的 modified coordinate system。

> **重要：**如果你计划编写直接使用 Quartz 的 iOS 应用的话，上述讨论很有必要理解，但不是必须的。自 iOS 3.2 起，当 UIKit 为你的应用创建一个 graphics context 时，UIKit 会对 context 做一些额外的修改来匹配默认的 UIKit 约定。尤其是 patterns 和 shadows，它们不被 CTM 所影响，它们被单独调整以便它们的约定和 UIKit 的坐标系匹配。在这种情况，对于 CTM 而言，没有等同的机制来修改一个 Quatz 提供给你的应用使用的 context，以便它匹配 UIKit 提供的 context 的行为。你的应用必须知道正绘制到什么类型的 context，并调整它的行为来匹配 context 的期望。

## Memory Management: Object Ownership
Quartz 使用 Core Foundation 内存管理模型，在模型中，对象是基于引用计数的。一旦创建，Core Foundation 对象就有个 1 的引用计数。你可以通过 retain 对象来增加引用计数，或通过 release 对象来减少对象的引用计数。当对象的引用计数减到 0 时，对象是就被释放了。这个模型允许对象安全的分享计数给其它对象。

有一些简单的规则需要记在心中：

* 如果你创建或赋值一个对象，你拥有它，因为你必须释放它。也就是说，一般如果你从一个有 *Create* 或 *Copy* 的函数获得一个对象，在你使用完这个对象之后，你必须释放它。否则，会导致一个内存泄露。
* 如果你从一个不包含 *Create* 或 *Copy* 的函数获取一个对象，你不拥有这个对象的引用，你一定不要释放它。对象会在未来的某个时间点被它的拥有者释放。
* 如果你拥有一个对象，并且你需要保持它存在，你必须要持有 (retain) 和释放 (release) 这个对象。例如，如果你收到一个 `CGColorspace` 对象的引用，你使用 `CGColorSpaceRetain` 和 `CGColorSpaceRelease` 函数来根据需要持有和释放对象。你也可以使用 Core Foundation 函数 `CFRetain` 和 `CFRelease`，但是你必须注意不要传递 `NULL` 给这些函数。

---
# Graphics Contexts
一个 graphics context 代表了一个绘制目的地。它包含绘制参数和所有绘制系统进行后续绘制命令所需的所有设备特定信息。一个 graphics context 定义了基本的绘制参数，如绘制时所用颜色，裁剪区域，线宽 (line width) 和样式信息，字体信息，组合选项，和一些其他的。

你可以通过使用 Quartz  context 创建函数或通过使用 OS X frameworks 或 iOS 中的 UIKit framework 提供的高级别函数来获取一个 graphics context。Quartz 提供了函数来创建不同类型的 Quartz graphics context，包括 bitmap 和 PDF，你可以使用它们来创建自定义内容。

这章给你展示了怎么创建一个 graphics context 来用于不同类型的绘制目的地。一个 graphics context 在你的代码中是通过数据类型 `CGContextRef` 来表示的，这是一个不透明数据类型。在你获取了一个 graphics context 之后，你可以使用 Quartz 2D 函数来绘制到这个 context，在这个 context 上进行操作 (如移动)，改变 graphics state 参数，如线宽和填充色。

## Drawing to a View Graphics Context in iOS
在 iOS 应用中绘制到屏幕，你设置一个 `UIView` 对象并实现 `drawRect:` 方法来进行绘制。视图的 `drawRect:` 方法在视图变得可见和它的内容需要更新的时候被调用。在你调用你的自定义 `drawRect:` 方法之前，视图图像自动配置它的绘制环境以便你的代码可以立即开始绘制。作为配置的一部分，`UIView` 对象为当前绘制环境创建一个 graphics context (一个 `CGContextRef` 的不透明类型)。你可以在你的 `drawRect:` 方法中通过调用 UIKit 函数 `UIGraphicsGetCurrentContext` 来获取这个 graphics context。

UIKit 使用的默认坐标系与 Quartz 使用的坐标系是不同的。在 UIKit 中，原点在左上角，y 值正向向下。`UIView` 对象修改 Quartz graphics context 的 CTM 来匹配 UIKit 的约定，这些修改是移动原点到视图的左上角并通过乘以 -1 来翻转 y 轴。关于修改坐标系和你自己的绘制代码中可能引起潜在结果的更多信息，可以查阅本文的 **Quartz 2D Coordinate Systems**.

`UIView` 对象在 *View Programming Guide for iOS* 中有描述。

## Creating a Window Graphics Context in Mac OS X
当在 OS X 中绘制时，你需要创建一个适用于你使用的框架的 window graphics context。Quartz 2D API 并不提供函数来获取一个 window graphics context。相反，你使用 Cocoa 来获取一个给 window 用的 graphics context。

你使用以下代码来从一个 Cocoa 应用的 `drawRect:` 例程中获取一个 Quartz graphics context:

```objc
CGContextRef myContext = [[NSGraphicsContext currentContext] graphicsPort];
```

方法 `currentContext` 返回当前线程的 `NSGraphicsContext` 实例。方法 `graphicsPort` 返回底层的特定平台的由接受者表示的 graphics context，也就是一个 Quartz graphics context。(不要被方法名给迷惑，它们是历史原因) 更多信息可以参见 *NSGraphicsContext Class Reference*。

在你获得 graphics context 之后，你可以在你的 Cocoa 应用中，调用任何 Quartz 2D 绘制函数。你可以混合使用 Quartz 2D 调用和 Cocoa 绘制调用。你可以查看 Figure 2-1 中使用 Quartz 2D 绘制到一个 Cocoa 视图的例子。绘制由两个重叠的矩形组成，一个不透明的红色和一个半透明的蓝色。在本文的 **Color and Color Spaces** 你会更多关于透明度的知识。控制你看穿 (see through) 颜色的能力是 Quartz 2D 特征特性之一。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/a_view_in_the_cocoa_framework_that_contains_quartz_drawing.png" width=80% />

要创建 Figure 2-1 中的绘制的话，你先要创建一个 Cocoa 应用项目。在 Interface Builder 中，移动一个自定义视图到 window 并继承它。然后编写这个子类视图的实现，和下面的代码相似。对于这个例子，子类视图叫 `MyQuartzView`. 视图的 `drawRect:` 方法包含所有的 Quartz 绘制代码。代码的后面有每行代码的具体解释。

> **注意**: `NSView` 的 `drawRect:` 方法在每次视图需要绘制的时候自动被调用。要想了解更多关于覆盖 `drawRect:` 方法的信息，查看 *NSView Class Reference*

```objc
@implementation MyQuartzView
- (id)initWithFrame:(NSRect)frameRect {    self = [super initWithFrame:frameRect];    return self;}

- (void)drawRect:(NSRect)rect {
    CGContextRef myContext = [[NSGraphicsContext  currentContext] graphicsPort]; //1    // ********** Your drawing code here ********** //2 
    CGContextSetRGBFillColor (myContext, 1, 0, 0, 1); //3
    CGContextFillRect (myContext, CGRectMake (0, 0, 200, 100 )); //4
    CGContextSetRGBFillColor (myContext, 0, 0, 1, .5); //5 
    CGContextFillRect (myContext, CGRectMake (0, 0, 100, 200)); //6}@end
```

下面是介绍代码做了啥：

1. 获取这个视图的 graphics context
2. 这里是你插入你的绘制代码的地方。跟着的 4 行代码是使用 Quartz 2D 函数的样例
3. 设置一个红色的填充色。关于颜色和 alpha 的更多信息，查看本文的 **Color and Color Spaces**
4. 填充一个原点在 (0, 0) 长为 200 高为 100 的矩形，关于绘制矩形的更多信息，可以查看本文的 **Paths**
5. 设置一个半透明的蓝色填充色
6. 填充一个原点在 (0, 0) 宽100 高 200 的矩形

## Creating a PDF Graphics Context
当你创建一个 PDF graphics context 并绘制到这个 context 时，Quartz 以一系列 PDF绘制命令记录你的绘制到文件。你提供一个给 PDF 输出的位置和一个默认 **media box** ——一个指定页面 bounds 的矩形框。Figure 2-2 展示了绘制到一个 PDF graphics context 的结果，然后在 Preview 中打开了结果 PDF.

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/a_pdf_created_by_using_CGPDFContextCreateWithURL.png" width=80% />

Quartz 2D 提供了两个函数创建一个 PDF graphics context：

* `CGPDFContextCreateWithURL`，当你想指定一个 Core Foundation URL 作为 PDF 输出的位置时使用这个函数。下面的代码展示了怎么使用这个函数来创建一个 PDF graphics context。
* `CGPDFContextCreate` 当你想要 PDF 输出到一个 data consumer. (更多信息请查看本文的 **Data Management in Quartz 2D**). 下面有代码示例显示怎么使用这个函数。

> **iOS Note**: iOS 中的一个 PDF graphics context 使用 Quartz 提供的默认坐标系统，并没有应用一个 transform 来匹配 UIKit 坐标系。如果你的应用计划在 PDF graphics context 和 `UIView` 对象提供的 graphics context 之间分享绘制代码，你的应用应该修改 PDF graphics context 的 CTM 以匹配坐标系。更多信息查看本文的 **Quartz 2D Coordinate Systems**。

```objc
CGContextRef MyPDFContextCreate (const CGRect *inMediaBox, CFStringRef path) {
    CGContextRef myOutContext = NULL;    CFURLRef url;
    
    url = CFURLCreateWithFileSystemPath (NULL, path, kCFURLPOSIXPathStyle, false) //1
    if (url != NULL) {
        myOutContext = CGPDFContextCreateWithURL (url, inMediaBox, NULL); // 2
        CFRelease(url); //3
    }
    return myOutContext;
}
```

下面是上述代码的解释：

1. 调用 Core Foundation 函数来从提供给 `MyPDFContextCreate` 函数的 CFString 对象创建一个 CFURL。你传递 `NULL` 给第一个参数来使用默认的 allocator。你也需要指定一个 path 样式，这个例子中是一个 POSIX-style 的 pathname。
2. 使用刚创建的 PDF 位置和指定 PDF 边界的矩形框来调用 Quartz 2D 函数来创建一个 PDF graphics context 。矩形框是传递给 `MyPDFContextCreate` 函数的参数，也是 PDF 的 default page media bounding box。
3. 释放这个 `CFURL` 对象。
4. 返回这个 PDF graphics context。当调用者不再需要 graphics context 时它必须释放 graphics context。

```objc
CGContextRef MyPDFContextCreate (const CGRect *inMediaBox, CFStringRef path) {
    CGContextRef        myOutContext = NULL;
    CFURLRef            url;
    CGDataConsumerRef   dataConsumer;
    
        url = CFURLCreateWithFileSystemPath (NULL, path, kCFURLPOSIXPathStyle, false); // 1
    if (url != NULL) {
        dataConsumer = CGDataConsumerCreateWithURL (url);  //2
        if (dataConsumer != NULL) {
            myOutContext = CGPDFContextCreate (dataConsumer, inMediaBox, NULL); // 3
            CGDataConsumerRelease (dataConsumer); // 4
        }
        CFRelease(url);  // 5
    }
    return myOutContext;  // 6
}
```

下面上述代码的解释：

1. 从传递给 `MyPDFContextCreate` 函数的一个 `CFString` 对象调用 Core Foundation 函数来创建一个 `CFURL` 对象。传递 NULL 为第一个参数使用默认的 allocator。你也需要指定一个 path style，这个例子中使用的 POSIX-style pathname。
2. 使用上述 `CFURL` 对象创建一个 Quartz data consumer 对象。如果你不想使用一个 CFURL` 对象 (例如，你想将 PDF 数据放在一个 `CFURL` 对象不能指定的一个位置)，你可以选择从一系列你在你的应用中实现的回调函数来创建一个 data consumer。更多信息，可以查看本文的 **Data Management in Quartz 2D**
3. 调用 Quartz 2D 函数来创建一个 PDF graphics context，将 data consumer 和传递给 `MyPDFContextCreate` 函数的矩形框作为参数传入。这个矩形框会作为 PDF 的 default page media bounding box。
4. 释放 data consumer
5. 释放 CFURL 对象
6. 返回 graphicx context 对象。调用必须在不再需要 graphics context 时释放它

下面的代码展示了怎么调用 `MyPDFContextCreate` 例程并绘制到它。代码的下面有每行代码的详细解释。

```objc
CGRect mediaBox;																			//1

mediaBox = CGRectMake (0, 0, myPageWidth, myPageHeight); 		//2
myPDFContext = MyPDFContextCreate (&mediaBox, CFSTR("test.pdf")); //3

CFStringRef myKeys[1];																				//4
CFTypeRef myValues[1];
myKeys[0] = kCGPDFContextMediaBox;
 myValues[0] = (CFTypeRef) CFDataCreate(NULL,(const UInt8 *)&mediaBox, sizeof(CGRect));

CFDictionaryRef pageDictionary = CFDictionaryCreate(NULL, (const void **)myKeys, (const void **) myValues, 1, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks); 
CGPDFContextBeginPage(myPDFContext, &pageDictionary); 				//5

// ********** Your drawing code here **********											//6
CGContextSetRGBFillColor (myPDFContext, 1, 0, 0, 1);CGContextFillRect (myPDFContext, CGRectMake (0, 0, 200, 100 ));CGContextSetRGBFillColor (myPDFContext, 0, 0, 1, .5);CGContextFillRect (myPDFContext, CGRectMake (0, 0, 100, 200 ));
CGPDFContextEndPage(myPDFContext);														//7
CFRelease(pageDictionary);																				//8
CFRelease(myValues[0]);
CGContextRelease(myPDFContext);
```

代码都做了啥:

1. 为被用来定义 PDF media box 的矩形框定义一个变量
2. 设置 media box 原点为 (0, 0) 和 width height
3. 调用 `MyPDFContextCreate` 函数来获取一个 PDF graphics context，提供了一个 media box 和一个 pathname。宏 `CFSTR` 转换一个字符串为一个 `CFStringRef` 数据类型。
4. 使用 page options 设置一个字典。在这个例子中，只有 media box 有指定。你不必传递与你使用来设置 PDF graphics context 相同的矩形框。这里你添加的 media box 接替了你传递来设置 PDF graphics context 的矩形框。
5. 发送信号开始一页。这个函数被用于面向页面的 graphics，而 PDF 绘制正是如此
6. 调用 Quartz 2D 绘制函数，你使用适合你应用的绘制代码替换这个和下面的四行代码
7. 发送信号结束 PDF 页面。
8. 当你不再需要它们的时候释放字典和 PDF graphics context。

你可以写入任何对你应用合适的内容到一个 PDF，包含图片，文本，path 绘制。并且你可以添加连接和加密。更多的信息请查看 **PDF Document Creation, Viewing, and Transforming**。

## Creating a Bitmap Graphics Context
一个位图 graphics context 接受一个指向一块内存区域的指针，这块区域包含了位图的存储空间。当你绘制到 bitmap graphics context 时，buffer 会被更新。在你释放这个 graphics context 之后，你有一个完全更新好的位图，位图是你指定的像素格式。

> **注意**: Bitmap graphics contexts 有时用于离屏绘制。在你决定使用一个 bitmap graphics context 来做这件事前，可以查看下本文的 **Core Graphics Layer Drawing**。CGLayer 对象 (`CGLayerRef`) 为离屏渲染有做优化，因为可能的情况下， Quartz 会在 video 卡缓存 layers。

> **iOS 注意**: iOS 应用应该使用函数 `UIGraphicsBeginImageContextWithOptions` 而不是这里描述的底层 Quartz 函数。如果你的应用使用 Quartz 创建一个离屏位图，位图 graphics context 使用的坐标系是 Quartz 默认的坐标系。相比下，如果你的应用通过调用函数 `UIGraphicsBeginImageContextWithOptions` 创建一个图片 context 的话，UIKit 会应用与 `UIView` 视图对象的 graphics context 相同的 transformation 到 context 的坐标系。这允许你的应用使用相同的绘制代码，而不需要关心不同的坐标系。尽管你的应用可以手动调整坐标 transformation 来达到同样的目的，但实践中，这么做没有任何性能优势。

你使用 `CGBitmapContextCreate` 函数创建一个 bitmap graphics context。这个函数会接受以下参数：

* `data`. 提供一个指针指向内存中的目的地，你的绘制渲染到这个目的地。这块内存的大小至少得 `bytesPerRow` * `height` bytes
* `width`. 指定位图的宽度，以像素为单位
* `height`. 指定位图的高度，以像素为单位
* `bitsPerComponent`. 指定内存中单个像素的每部分 (each component) 使用的字节数。例如，对一个 32-bit 像素格式和一个 RGB 颜色空间，你将指定一个 8bits 作为 component 的大小。具体查看本文的 **Supported Pixel Formats**
* `bytesPerRow`. 指定位图的每行所用内存字节数

	> **Tip**: 当你创建一个 bitmap graphics context 时，如果你确保 `data` 和 `bytesPerRow` 是 16-byte 对齐的话，你将获得最好的性能。

* `colorspace`. 用于 bitmap context 的 color space。你在创建一个 bitmap graphics context。你可以提供一个 Gray, RGB, CMYK, 或 NULL 颜色空间。关于颜色空间和颜色管理原则的具体信息可以查看 *Color Management Overview*。关于在 Quartz 中创建和使用颜色空间，查看本文的 **Color and Color Spaces**。关于支持的颜色空间的更多信息，可以查看本文 **Bitmap Images and Image Masks** 的 **Color Spaces and Bitmap Layout**。
* `bitmapInfo`. 位图布局信息，通过一个 `CGBitmapInfo` 常量。这个常量指定位图是否应该包含一个 alpha component，alpha component (有的话) 在像素中相对位置，alpha component 是否是 premultiplied，和 color components 是整形还是浮点型的。关于这些常量的具体信息，每个在什么时候使用，和对于 bitmap graphics context 和图片的 Quartz 支持的像素格式。具体可以查看本文的 **Bitmap Images and Image Masks** 中的 **Color Spaces and Bitmap Layout**。

下面的代码展示了怎么创建一个 bitmap graphics context。当你绘制到结果的 bitmap graphics context，Quartz 在指定的内存块中记录你的绘制为位图数据。每行代码的具体解释在代码的后面。

```objc
CGContextRef MyCreateBitmapContext (int pixelsWide, int pixelsHigh) {
    CGContextRef        context = NULL;    CGColorSpaceRef colorSpace;    void *                        bitmapData;    int                              bitmapByteCount;    int                              bitmapBytesPerRow;
    
    bitmapBytesPerRow   = (pixelsWide * 4);                                        //1
    bitmapByteCount     = (bitmapBytesPerRow * pixelsHigh);
    
    colorSpace = CGColorSpaceCreateWithName(kCGColorSpaceGenericRGB); //2
    bitmapData = calloc( bitmapByteCount );                                     //3
    if (bitmapData == NULL) {
        fprintf (stderr, "Memory not allocated!");
        return NULL;
    }
    context = CGBitmapContextCreate(bitmapData,                       //4
                                                                     pixelsWide,
                                                                     pixelsHigh,
                                                                     8,               // bits per component
                                                                     bitmapBytesPerRow,
                                                                     colorSpace,
                                                                     kCGImageAlphaPremultipliedLast);
    if (context== NULL) {
        free (bitmapData);                                                                     //5
        fprintf (stderr, "Context not created!");
        return NULL;
    }
    CGColorSpaceRelease( colorSpace );                                      //6
    
    return context;                                                                               //7
}
```

下面介绍了每行代码做了啥：

1. 申明一个变量来代表每行字节数。这个例子中位图的每个像素通过 4 字节来表示；red, green, blue, and alpha 每个 8 位。
2. 创建一个通用的 RGB 颜色空间。你也可以创建一个 CMYK 颜色空间。查看本文的 **Color and Color Spaces** 来了解更多信息，和一个关于通用颜色空间和设备依赖颜色空间的讨论。
3. 调用 `calloc` 函数来创建和清空一块内存，这块内存将保存位图数据。这个例子创建 32位 RGBA 的位图 (也就是说，一个每像素 32 bit 的数据，每个像素包含 8 位的 red, green, blue, alpha 信息)。位图中的每个像素占有 4 字节内存。在 OS X 10.6 和 iOS 4 中，这个步骤可以省略掉——如果你传递 `NULL` 作为位图数据，Quartz 自动为位图分配空间。
4. 创建一个位图 graphics context，提供位图数据，位图的宽和高，每个 component 的 bit 数，每行的字节数，颜色空间，和一个指定位图是否应该包含 alpha 频道和它在像素中的相对位置的常量。常量 `kCGImageAlphaPremultipliedLast` 表示 alpha component 存储在每个像素的最后一个字节且颜色的组成部分 (components) 已经与这个 alpha 值相乘。查看本文的 **The Alpha Value** 来了解更多关于 premultiplied alpha 的信息。
5. 如果 context 因为某些原因没有创建，释放为位图数据分配的内存。
6. 释放颜色空间
7. 返回 bitmap graphics context。在 graphics context 不再被需要的时候，调用者必须释放这个 graphics context。

下面的代码调用 `MyCreateBitmapContext` 来创建一个 bitmap graphics context，并使用 bitmap graphics context 来创建一个 `CGImage` 对象，然后绘制结果图片到一个 window graphics context。图 Figure2-3 展示了绘制到 window 的图片。关于每行代码的具体解释紧接代码。

```objc
CGRect myBoundingBox;                                                                    //1

myBoundingBox = CGRectMake (0, 0, myWidth, myHeight);      //2
myBitmapContext = MyCreateBitmapContext (400, 300);           //3
// ********** Your drawing code here **********                             //4
CGContextSetRGBFillColor (myBitmapContext, 1, 0, 0, 1);CGContextFillRect (myBitmapContext, CGRectMake (0, 0, 200, 100 ));CGContextSetRGBFillColor (myBitmapContext, 0, 0, 1, .5);CGContextFillRect (myBitmapContext, CGRectMake (0, 0, 100, 200 ));
myImage = CGBitmapContextCreateImage (myBitmapContext);   //5
CGContextDrawImage(myContext, myBoundingBox, myImage);   //6
char *bitmapData = CGBitmapContextGetData(myBitmapContext); //7
CGContextRelease (myBitmapContext);                                                   //8
if (bitmapData) free(bitmapData);                                                              //9
CGImageRelease(myImage);                                                                     //10
```

下面是上述代码的相关解释：

1. 申明变量来保存 bounding box 的原点和尺寸，从 bitmap graphics context 中创建的图片将被 Quartz 绘制到这个 bounding box。
2. 设置 bounding box 的原点为 (0, 0) width 和 height 为之前申明的变量，但是这部分代码中没有显示它们的申明。
3. 调用应用提供的 `MyCreateBitmapContext` 函数来创建一个 bitmap context，这个 context 是 400 像素宽和 300 像素高。你可以对你应用合适的尺寸来创建一个 bitmap graphics context。
4. 调用 Quartz 2D 函数绘制到 bitmap graphics context。你将使用适合于你应用的绘制代码来替换这个和接下来的四行代码。
5. 从 bitmap graphics context 创建一个 Quartz 2D 图片 (`CGImageRef`) 
6. 绘制图片到 bounding box 在 window graphics context 中指定的位置。bounding box 指定了 user space 中绘制图片的位置和尺寸。这个样例代码没有显示创建 window graphics context 的代码。查看 **Creating a Window Graphics Context in Mac OS X** 来了解更多信息。
7. 获取与 bitmap graphics context 相关联的 bitmap data。
8. 当 graphics context 不再需要的时候，释放它。
9. 如果 bitmap data 存在的话释放它
10. 当图片不再需要的时候释放它。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/an_image_created_from_a_bitmap_graphics_context_and_drawn_to_a_window_graphics_context.png" width=80% />

### Supported Pixel Formats
下表总结了 bitmap graphics context 支持的像素格式，关联的颜色空间，以及 OS X 是从哪个版本开始支持的。像素格式是通过 *bits per pixel* (bpp) 和 *bits per component* (bpc)。这个表也包含与像素格式相关联的 bitmap information constant。查看 *CGImage Reference* 来了解关于每个 bitmap information format constants 代表什么。

CS | 像素格式和 bitmap information constant | Availability
----- | ----------------------------------- | ----------------------
Null | 8 bpp, 8 bpc, kCGImageAlphaOnly | Mac OS X, iOS
Gray | 8 bpp, 8 bpc,kCGImageAlphaNone | Mac OS X, iOS
Gray | 8 bpp, 8 bpc,kCGImageAlphaOnly | Mac OS X, iOS
Gray | 16 bpp, 16 bpc, kCGImageAlphaNone | Mac OS X
Gray | 32 bpp, 32 bpc, kCGImageAlphaNone|kCGBitmapFloatComponents | Mac OS X
RGB | 16 bpp, 5 bpc, kCGImageAlphaNoneSkipFirst | Mac OS X, iOS
RGB | 32 bpp, 8 bpc, kCGImageAlphaNoneSkipFirst | Mac OS X, iOS
RGB | 32 bpp, 8 bpc, kCGImageAlphaNoneSkipLast | Mac OS X, iOS
RGB | 32 bpp, 8 bpc, kCGImageAlphaPremultipliedFirst | Mac OS X, iOS
RGB | 32 bpp, 8 bpc, kCGImageAlphaPremultipliedLast | Mac OS X, iOS
RGB | 64 bpp, 16 bpc, kCGImageAlphaPremultipliedLast | Mac OS X
RGB | 64 bpp, 16 bpc, kCGImageAlphaNoneSkipLast | Mac OS X
RGB | 128 bpp, 32 bpc, kCGImageAlphaNoneSkipLast \| kCGBitmapFloatComponents | Mac OS X
RGB | 128 bpp, 32 bpc, kCGImageAlphaPremultipliedLast \| kCGBitmapFloatComponents | Mac OS X
CMYK | 32 bpp, 8 bpc, kCGImageAlphaNone | Mac OS X
CMYK | 64 bpp, 16 bpc, kCGImageAlphaNone | Mac OS X
CMYK | 128 bpp, 32 bpc, kCGImageAlphaNone \| kCGBitmapFloatComponents | Mac OS X

### Anti-Aliasing
Bitmap graphics context 支持 **anti-aliasing**，防锯齿是当你绘制文本或形状时人工修正你在位图上看到的带锯齿边的过程。这些锯齿的边在位图的分辨率比眼睛的分辨率低很多的时候出现，Quartz 为像素使用不同的颜色来包围形状的轮廓。

通过这个方式来混合颜色，形状就会看上去很平滑。你可以查看图 Figure 2-4 中使用防锯齿的效果。你可以通过调用函数 `CGContextSetShouldAntialias` 来为一个特定的 bitmap graphics context 关闭防锯齿。防锯齿的设置是 graphics state 的一部分。

你可以通过函数 `CGContextSetAllowsAntialiasing` 来控制一个特定 graphics context 是否允许防锯齿。传 `true` 给这个函数来允许防锯齿；`false` 来关闭。这个设置不是 graphics state 的一部分。Quartz 在 context 和 graphics state 设置被设置为 `true` 的时候进行防锯齿。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/a_comparison_of_aliased_and_anti-aliasing_drawing.png" width=80% />

## Obtaining a Graphics Context for Printing
OS X 中的 Cocoa 应用通过自定义 `NSView` 子类实现打印。通过调用 `print:` 方法来告知一个视图打印。然后视图创建一个以打印机为目的地的 graphics context 然后调用它的 `drawRect:` 方法。你的应用使用与绘制到屏幕相同的绘制代码绘制到打印机。它也可以自定义 `drawRect:` 调用来绘制一张与屏幕上不同的图片到打印机的。

关于 Cocoa 中的打印的具体的讨论，请查看  *Printing Programming Guide for Mac*。

---
# Paths
一个 **path**定义了一个或多个形状或 subpaths. 一个 subpath 可以由直线，曲线或同时组成。它可以是开放 (open) 或闭合 (closed) 的. 一个 subpath 可以是一个简单形状，如一条线，圆形，矩形，或星形，或一个如山的轮廓或一个抽象的乱画等更复杂的形状。图 Figure 3-1 展示了一些你可以创建的 paths。左上角的直线是虚线; 直线也可以是实线。中间上部弯弯曲曲的线是由几个曲线组成并且它是开发的。同心圆是填充的，但轮廓没有描摹。California 州是一个闭合 path，由许多曲线和直线构成，并且 path 是填充并描摹了轮廓。星星描述了填充 path 的两种选项，你将在这章的后面读到相关部分。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/quartz_supports_path-based_drawing.png" width=80% />

在这章，你将学到组成 path 的基础构建块，怎么描摹轮廓和给 path 染色，和影响 path 外表的参数。

## Path Creation and Path Painting
Path 创建和 path 涂色是不同的任务。首先你创建一个 path。当你想要渲染一个 path 的时候，你请求 Quartz 来给它染色。像你在 Figure 3-1 中看到的一样，你可以选择 stroke 这个 path，填充这个 path，或者同事 stroke 或填充这个 path。你也可以使用 path 来限制其它对象的绘制到 path 创建的边界内，效果上成为一个 **clipping area**。

Figure 3-2 展示了一个已经被涂色并包含两个 subpaths 的 path。左边的 subpath 是一个矩形框，右边的 subpath 是一个抽象的形状由直线和曲线组成。每个 subpath 都被填充，轮廓也被 stroked。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/a_path_that_contains_two_shapes.png" width=80% />

Figure 3-3 显示了多条 path 相互独立的被绘制。每条 path 包含一个随机产生的曲线，一些被填充，一些被 stoked。绘制被限制到一个 clipping area 指定的圆形区域。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/a_clipping_area_constrains_drawing.png" width=80% />

## The Building Blocks
Subpaths 是由直线，弧线和曲线构建。Quartz 通过单一的函数调用提供了方便的方法来添加矩形框和椭圆。Points 也是构建 path 的必须组件因为 Points 定义了形状的开始和结束。

### Points
Points 是指定在 user space 中位置的 x 和 y 坐标。你可以调用函数 `CGContextMoveToPoint` 来为一个新 subpath 指定一个开始位置。Quartz 会记录当前点 (**current point**)，这个 point 是用于构建 path 的最后位置。例如，如果你调用函数 `CGContextMoveToPoint` 来设置一个在 (10, 10) 的位置，这会移动当前点到 (10, 10)。如果然后你绘制一个 50单位长的水平线，线上的最后一点，也就是 (60, 10)，成为当前点。Lines, arcs, and curves 总是从当前点开始绘制。

大部分时间你通过传递给 Quartz 函数两个指定 x 和 y 的浮点数来指定一个点。一些函数要求你传递 `CGPoint` 数据结构，这个结构持有两个浮点数。

### Lines
一个线是由它两端的点决定的。它开始的点总是被假设为当前点，所以当你创建一个线是，你只需指定它的结束点。你使用 `CGContextAddLineToPoint` 函数来追加单一线段到一个 subpath。

你可以通过调用函数 `CGContextAddLines` 来添加一系列连接的线到一个 path。你传给这个函数一组点。第一个点必须是第一条线段的起始点，接下来的点都是结束点。Quartz 在第一点上开始一个新的 subpath 并连接一个线段到每个结束点。

### Arcs
弧线是圆形片段。Quartz 提供两个函数创建弧线。函数 `CGContextAddArc` 创建一个从一个圆创建一个弧线段。你指定圆形的圆点，半径，和弧度角 (以弧度为单位)。你可以通过指定一个 2 pi 来创建一个圆形。Figure 3-4 显示多个独立绘制的 paths。每个 path 包含一个随机产生的圆形；一些被填充，一些被 stroked。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/each_path_contains_a_randomly_generated_circle.png" width=80% />

函数 `CGContextAddArcToPoint` 在你想给矩形添加圆角时很适合使用。Quartz 使用你提供的两个点来创建两条射线。你也需要提供圆形的半径以便 Quartz 从圆形裁切弧线。弧线的圆点是两条半径的交点，每条半径垂直于两条射线之一。弧线的每个端点是射线上的射点，像 Figure 3-5 中显示的那样，圆形的红色部分正是真实被绘制的部分。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/defining_an_arc_with_two_tangent_lines_and_a_radius.png" width=80% />

如果当前的 path 已经包含一个 subpath，Quartz 会追加一个连接当前点到弧线开始点的直线段。如果当前点是空的，Quartz 为弧线从当前起始点创建一个新的 subpath，并不添加初始的直线段。

### Curves
二次方和三次方贝塞尔曲线是代数曲线，它们可以指定任意数量的曲线形状。这些曲线上的点通过应用多项式到开始点，结束点和一个或多个控制点上来计算出来。以这种方式定义的形状是矢量图形的基础。相对于存储一组bits 而言一个方程式要紧凑很多，并且有可以在任意分辨率上重绘的优点。

Figure 3-6 通过相互独立的绘制不同的 path 展示了各种曲线。每个 path 包含一个随机产生的曲线；一些被填充，其他被 stroked。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/each_path_contains_a_randomly_generated_curve.png" width=80% />

产生二次和三次贝塞尔曲线的多项式方程，和怎样从方程产生曲线，在许多描述计算机图形的数学书本和在线资源上有介绍。这些具体信息不在此处介绍。

你使用 `CGContextAddCurveToPoint` 来从当前点追加一个立方贝塞尔曲线，这个曲线通过使用控制点和一个中点来指定。Figure 3-7 显示了从图中的当前点，控制点，和端点做出的三次方贝塞尔曲线。两个控制点的位置决定了曲线的几何。如果控制点同时在起始点和终点之上的话，曲线会向上弓起。如果控制点都在开始点和结束点之下的话，曲线向下凹。如果第二个控制点比第一个更靠近当前点的话，曲线会交叉，创建一个循环。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/a_cubic_bezier_curve_uses_two_control_points.png" width=80% />

你可以从当前点通过调用 `CGContextAddQuadCurveToPoint` 函数追加一个二次方贝塞尔曲线，调用时指定一个控制点和一个终点。Figure 3-8 展示了使用相同端点但是不同控制点的曲线。控制点决定了曲线弓起的方向。相对于三次方贝塞尔曲线而言使用一个二次方贝塞尔曲线不大可能创建很多有趣的形状，因为二次方贝塞尔曲线只能使用一个控制点。例如，使用一个控制点不可能创建一个交叉线。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/a_quadratic_bezier_curve_uses_one_control_point.png" width=80% />

### Closing a Subpath
要关闭当前 subpath，你的应用应该调用 `CGContextClosePath` 函数。这个函数添加一个连接当前点和 subpath 起点的线段并关闭 subpath。在一个 subpath 的起点结束的lines, arcs, and curves 实际上并没有关闭 subpath。你必须要显式的调用 `CGContextClosePath` 来关闭一个 subpath。

一些 Quartz 函数会以像应用关闭 subpath 的方式对待一个 path 的 subpath。这些命令对待 subpath 就像你的应用已经调用过 `CGContextClosePath` 来关闭它，这样会隐式的添加一个线段到 subpath 的起始点。

在关闭一个 subpath 之后，如果你的应用调用额外的函数来添加 lines, arcs, 或 curves 到 path，Quartz 会从你刚关闭掉的 subpath 的起始点开始一个新的 subpath。

### Ellipses
一个椭圆本质上是一个被压扁的圆。你通过定义两个焦点，然后标出到这两点距离之和相等的这些点。Figure 3-9 展示多个独立绘制的 path。每条 path 包含一个随机产生的椭圆；一些已经被填充，其他被 stroked。

<img src=https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/each_path_contains_a_randomly_generated_ellipse.png width=80% />

你可以通过调用 `CGContextAddEllipseInRect` 函数来添加一个椭圆到当前的 path。你提供一个定义椭圆的边界的矩形。Quartz 使用一系列贝塞尔曲线来模拟椭圆。椭圆的中心在矩形的中心。如果矩形的长宽相等的话，椭圆将是圆的，圆的半径等于矩形长或宽的一半。如果矩形的长和宽不相等，它们定义了椭圆的主副轴。

添加到 path 的椭圆以一个 `move-to` 操作开始，以一个 `close-subpath` 操作结束，其他所有的操作都是顺时针方向的。

### Rectangles
你可以通过调用 `CGContextAddRect` 添加矩形框到当前的 path。你提供一个包含原点和大小尺寸的 `CGRect` 结构。

添加的矩形框以一个 `move-to` 操作开始，一个 `close-subpath` 操作结束，其他所有的操作都是逆时针方向的。

你可以通过调用提供一组 `CGRect` 结构给 `CGContextAddRects` 函数来添加多个矩形框到当前的 path。Figure 3-10 展示了多个独立绘制的 path。每个 path 包含一个随机产生的矩形框，一些被填充，一些被 stroked。

<img src=https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/each_path_contains_a_randomly_generated_rectangle.png width=80% />

## Creating a Path
当你想要在一个 graphics context 中构建一个 path 时，你通过调用 `CGContextBeginPath` 给 Quartz 发送信号。接着，你通过在 path 中调用函数 `CGContextMoveToPoint` ，设置第一个形状或 subpath 的开始点。在你建立第一个点之后，你可以添加 lines, arcs, and curves 到 path，记住以下步骤：

* 在你开始一个新 path 之前，调用函数 `CGContextBeginPath`
* Lines, arcs, and curves 从前当前点开始绘制。一个 empty path 没有当前点；你必须调用 `CGContextMoveToPoint` 来设置来第一个 subpath 的开始点，或调用一个会隐式帮你做这个的简便函数。
* 当你想要关闭一个 path 中的 subpath 时，调用函数 `CGContextClosePath` 来连接一个线段到 subpath 的开始点。接下来的 path 调用会开始一个新的 subpath，即使你没有显式的设置一个显得开始点。
* 当你绘制 arcs，Quartz 会绘制一条从当前点到 arc 开始点的线。
* 添加椭圆和矩形框的 Quartz 例程会添加一个闭合的 subpath 到这个 path
* 你必须调用一个绘制函数来填充或 stroke 这个 path 因为创建一个 path 不会绘制这个 path。查看本文的 **Painting a Path** 来了解更多的信息。

在你绘制一个 path 之后，它就被从 graphics context 中刷掉了。你也许不想不想你的 path 这么轻易地就被丢失了，尤其是你想多次使用的复杂形状。因为这个原因，Quartz 为创建可重用的 path 提供了两种数据类型 —— `CGPathRef` 和 `CGMutablePathRef`。你可以调用函数 `CGPathCreateMutable` 来创建一个可变的 `CGPath` 对象，你可以添加 lines, arcs, curves, and rectangles 到这个对象。Path 函数操作在一个 path 对象上而不是一个 graphics context 上。这些函数包括：

* `CGPathCreateMutable`，替换 `replacesCGContextBeginPath`
* `CGPathMoveToPoint`，替换 `CGContextMoveToPoint`
* `CGPathAddLineToPoint`，替换 `CGContextAddLineToPoint`
* `CGPathAddCurveToPoint`，替换 `CGContextAddCurveToPoint`
* `CGPathAddEllipseInRect`，替换 `CGContextAddEllipseInRect`
* `CGPathAddArc`，替换 `CGContextAddArc`
* `CGPathAddRect`，替换 `CGContextAddRect`
* `CGPathCloseSubpath`，替换 `CGContextClosePath`

查看 *Quartz 2D Reference Collection* 来了解 path 函数的完整列表。

当你想要追加 path 到一个 graphics context 时，调用函数 `CGContextAddPath`。这个 path 会保持在 graphics context 内，直到 Quartz 绘制它。你可以通过调用 `CGContextAddPath` 来添加这个 path。

> **注意:** 你可以通过调用函数 `CGContextReplacePathWithStrokedPath` 来使用 path 的 stroked 版本替换 graphics context 中的 path。

## Painting a Path
你可以通过 stroking 或填充或同时做这两者来绘制当前 path。**Stroking** 沿着 path 画条线。**Filling** 画 path 包含的区域。Quartz 有功能让你 stroke 一个 path，填充一个 path，或同时 stroke 和填充 path。被 stroked line 的特点 (width, color and so forth), 填充色和 Quartz 是使用来计算填充区域的方法是 graphics state 的一部分。 (查看本文的 **Graphics State**)

### Parameters That Affect Stroking
你可以通过修改下表中的参数来影响一个 path 怎么被 stroked。这些参数是 graphics state 的一部分，这意味着你给一个 parameter 所设置的值会影响所有接下来的 stroking，直到你显式的给这个 parameter 设置另外的值。

Parameter | Function to set parameter value
-------------- | ------------------------
Line Width | `CGContextSetLineWidth`
Line join | `CGContextSetLineJoin`
Line cap | `CGContextSetLineCap`
Miter limit | `CGContextSetMiterLimit`
Line dash pattern | `CGContextSetLineDash`
Stroke color space | `CGContextSetStrokeColorSpace`
Stroke color | `CGContextSetStrokeColor` and `CGContextSetStrokeColorWithColor`
Stroke pattern | `CGContextSetStrokePattern`

**line width** 是线的整体宽度，通过 user space 中的单位表示的。线顺延着 path，在 path 的两侧刚好是线宽度的一半。

**line join** 指定了 Quartz 绘制线段连接处的绘制方式。Quartz 支持下表中描述的 join 样式。默认的样式是 miter join。

Style | Appearance | Description
-------- | ------------------ | ---------------
Miter join | <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/join_style_miter_join.png" width=80% /> | Quartz 延伸连接的两线段的 strokes 的外边直到它们相连，形成一个角。如果这个角过于尖锐，会使用一个 bevel join 来替换。如果 milter 除以 line width 的值大于 miter limit 的话一个 segment 就被认为是太过于尖锐。
Round join | <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/join_style_round_join.png" width=80% /> | Quart 使用一个直径等于 line width 的圆在终点处绘制一个半圆的弧线。被包含的区域被填充。
Bevel join |  <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/join_style_bevel_join.png" width=80% /> | Quartz 使用 butt caps 来结束两个线段。The resulting notch beyond the ends of the segments is filled with a triangle.

**line cap** 指定函数 `CGContextStrokePath` 绘制线段的端点时所使用的方法。Quartz 支持下表中描述的 line cap 样式。默认样式是 butt cap。

Style | Appearance | Description
------- | ----------------- | -----------------
Butt cap | <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/line_cap_butt_cap.png" width=80% /> | Quartz squares off the stroke at the endpoint of the path. There is no projection beyond the end of the path.
Round cap | <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/line_cap_round_cap.png" width=80% /> | Quartz 使用等于 line width 的直径围绕端点绘制一个圆，这个端点是两个线段相连的地方，产生一个圆角。被包含的区域会被填充。
Projecting square cap | <img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/line_cap_projecting_square_cap.png" width=80% /> | Quartz 延伸 path 的 stroke 超过 path 的两端，延伸的距离为 line width 的一半。延伸以矩形结束 (The extension is squared off)。

一个闭合的 subpath 会把起始点当做相连线段的连接点；开始点使用选择的 line-join 方法来渲染。相比之下，如果你通过添加一个连接开始点的线段来关闭一个 path 的话，path 的两端会使用选择的 line-cap 方法来绘制。

一个 **line dash pattern** 允许你绘制沿着 stroked path 绘制一个分隔的线段。你通过指定 dash array 和 dash phase 参数到 `CGContextSetLineDash` 函数，来控制 dash segement 的位置和大小。

```objc
void CGContextSetLineDash (    CGContextRef ctx,    CGFloat phase,    const CGFloat lengths[],    size_t count);
```

`lengths` 参数的元素指定 dash 的宽度，在 painted 和 unpainted segments 之间迭代。 `phase` 参数指定 dash pattern 开始的点。Figure 3-11 展示了一些 line dash patterns。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/examples_of_line_dash_patterns.png" width=80% />

stroke 的 **color space** 决定了 Quartz 怎么翻译 stroke color。你也可以指定一个 Quartz color (`CGColorRef` 数据类型) ，这个类型同时封装了颜色和颜色空间。关于设置颜色和颜色空间的更多信息，可以查看本文的 **Color and Color Spaces**。

### Functions for Stroking a Path
Quartz 提供了下面中的函数来 stroke 当前 path。一些是用来矩形框或椭圆的简便函数。

Function | Description 
------------ | -----------------
`CGContextStrokePath` | Stroke 当前 path
`CGContextStrokeRect` | Stroke 指定的矩形框
`CGContextStrokeRectWithWidth` | 使用指定的 line width 来 stroke 指定的矩形框
`CGContextStrokeEllipseInRect` | 在指定的矩形框内 stroke 一个椭圆
`CGContextStrokeLineSegments` | Stroke 一系列的线。
`CGContextDrawPath` | 如果传递常量 `kCGPathStroke` 的话，stroke 当前 path。如果你想同时 stroke 和填充，查看本文的 **Filling a Path**

函数 `CGContextStrokeLineSegments` 和以下代码是等同的：

```objc
CGContextBeginPath (context);for (k = 0; k < count; k += 2) {    CGContextMoveToPoint(context, s[k].x, s[k].y);    CGContextAddLineToPoint(context, s[k+1].x, s[k+1].y);}CGContextStrokePath(context);
```

当你调用 `CGContextStrokeLineSegments` 函数时，你通过一组点来指定 line segments，这些点以对的形式组织。每对又由 line segment 的起始点和它的结束点组成。例如，数据中第一个点指定第一条线的起始点，第二个点指定了第一条线的结束点，第三个点指定了第二条线的起始点，如此继续下去。

### Filling a Path
当你填充当前的 path 时，Quartz 会假设 path 中的 subpath 都被闭合了。然后它使用这些闭合的 subpath 并计算像素来填充它。Quartz 可以使用两种方式来计算填充区域。诸如椭圆和矩形框的简单 path 有定义好的区域。但是如果 path 由重叠的部分或 path 包含多个 subpaths 的话，如 Figure 3-12 中的多个同心圆，有两个规则你可以使用来决定填充区域。

默认的规则称为 **nonzero winding number rule**. 要决定一个指定的点是否应该上色，从这点开始，画一条线超过绘制的边界。从 0 开始计数，每次一个 path 片段从左往右与线相交给计数加 1，每次一个 path 片段从右往左与线相交计数减 1。如果计数结果为 0，该点不被上色。否则的话，被上色。被画的 path 片段的方向会影响最终结果。Figure 3-12 显示了两组内外圆形使用 nonzero winding number rule 填充的样子。当每个圆形以相同的方向绘制时，两个 circles 被填充。当圆形以不同的方向绘制时，内部圆形不被填充。

你可以选择使用 **even-odd rule**。要决定一个指定点是否被绘制，从这点开始画一条线超过绘制的边界。计算与线相交的次数。如果结果是计数，点被绘制。如果结果是偶数，点不被绘制。绘制的方向不会影响最终结果。正如 Figure 3-12 中你看到的一样，圆的方向并不重要。填充区域总是相同的。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/concentric_circles_filled_using_different_fill_rules.png" width=80% />

Quartz 提供了下表中的方法来填充当前的 path。一些函数使用简便的 stroking 矩形框或椭圆。

Function | Description
------------ | ----------------
`CGContextEOFillPath` | 使用 even-odd 规则填充当前 path
`CGContextFillPath` | 使用 nonzero winding number rule 来填充当前 path
`CGContextFillRect` | 填充指定矩形框内的区域
`CGContextFillRects` | 填充指定矩形框的内部区域
`CGContextFillEllipseInRect` | 填充一个指定矩形框内的椭圆
`CGContextDrawPath` | 如果你传递 `kCGPathFill` 填充当前 path (nonzero winding number rule) 或 `kCGPathEOFill` (even-odd rule)。如果你传递 `kCGPathFillStroke` 或 `kCGPathEOFillStroke` 就填充和 strokes 当前 path。

### Setting Blend Modes
**Blend modes** 指定 Quartz 怎么在一个背景只上涂色。Quartz 默认使用 normal blend mode，默认使用下面的方式使用下面的方程来组合 foreground painting 和 background painting。

``` c
result = (alpha * foreground) + (1 - alpha) * background
```

本文的 **Color and Color Spaces** 提供了关于颜色的 alpha component 的具体讨论，alpha component 指定了颜色的透明度。对于在这部分的例子，你可以假设颜色是完全不透明的 (alpha 值为 1.0)。对于不透明颜色，当你使用 normal blend mode 来绘制时，任何你在背景上的绘制完全阻碍掉背景。

你可以通过传递合适的 blend mode 常量给 `CGContextSetBlendMode` 函数设置 blend mode 来达成一些效果。记住 blend mode 是 graphics state 的一部分。如果你在改变 blend mode 之前使用函数 `CGContextSaveGState` 的话，之后调用 `CGContextRestoreGState` 函数会重置 blend mode 为 normal。

这部分的剩余部分展示了在 Figure 3-14 中的矩形框上绘制 Figure 3-13 中的矩形框时的结果。在每种情况下，背景中的矩形框使用 normal blend mode 来绘制。然后通过使用合适的常量调用 `CGContextSetBlendMode` 函数来改变 blend mode。最后，foreground rectangles 被绘制。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/the_rectangles_painted_in_the_foreground.png" width=80% />

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/the_rectangles_painted_in_the_background.png" width=80% />

> **注意:** 你也可以使用 blend mode 来组合两张图片，或组合图片到 graphics context 上已经绘制的内容。本文的 **Using Blend Modes with Images** 提供了关于怎么使用 blend modes 来组合图片的信息，并展示了应用 blend modes 到两张图片的结果。

#### Normal Blend Mode
因为 normal blend mode 是默认的 blend mode，你会使用常量 `kCGBlendModeNormal` 调用 `CGContextSetBlendMode` 来重置 blend mode 到默认值，而这只会发生在你使用过其他的 blend mode 之后。Figure 3-15 展示了使用 normal blend mode 绘制的结果。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_normal_blend_mode.png" width=80% />

#### Multiply Blend Mode
Multiply blend mode 指定将 foreground image samples 与 background image samples 相乘。结果颜色至少与其中之一一样暗。Figure 3-16 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeMultiply` 调用函数 `CGContextSetBlendMode`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_multiply_blend_mode.png" width=80% />

#### Screen Blend Mode
Screen blend mode 指定将 foreground image samples 的相反色与 background image samples 的相反色相乘。结果颜色至少与两个组成 smaple 颜色之一一样。Figure 3-17 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeScreen` 调用函数 `CGContextSetBlendMode`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_screen_blend_mode.png" width=80% />

####Overlay Blend Mode
Overlay blend mode 指定要么 multiply 要么 screen foreground image samples 和 background image samples，依赖于 background color。背景色与前景色混合以反映背景色的亮度或暗度。Figure 3-18 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeOverlay` 调用函数 `CGContextSetBlendMode`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_overlay_blend_mode.png" width=80% />

#### Darken Blend Mode
通过选择更暗的 samples 来创建组合后的 image samples (要么是选择前景色要么是背景色)。背景色被任何更暗的前景色所替换。否则，背景色不被改变。Figure 3-19 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeDarken` 调用函数 `CGContextSetBlendMode`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_darken_blend_mode.png" width=80% />

#### Lighten Blend Mode
通过选择更亮的颜色 (要么是选择前景色要么是背景色)。背景色被任何更亮的前景色所替换。否则，背景色不被改变。Figure 3-20 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `CGContextSetBlendMode`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_lighten_blend_mode.png" width=80% />

####Color Dodge Blend Mode
指定加亮背景色来反映前景色。前景图中黑色样品数据不发生改变。Figure 3-21 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeColorDodge`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_color_dodge_blend_mode.png" width=80% />

####Color Burn Blend Mode
指定加暗背景色来反映前景色。前景图中非白色样品数据不发生改变。Figure 3-22 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeColorBurn`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_color_burn_blend_mode.png" width=80% />

#### Soft light Blend Mode
依赖于前景图片样品颜色，要么加暗要么加亮颜色。如果前景图片样品颜色比 50% 灰要亮，背景色被加亮，与 dodging 相似。如果前景图片样品色比 50% 灰要暗，背景色被加暗，和 burning 相似。如果等于 50% 灰的话，背景色不变。图片样品中等于全黑或全白的区域产生暗或亮的区域，但不会导致全黑或全白。总的效果和你在前景色上闪烁一个四散的聚光灯效果差不多。使用它来给场景增加高亮。Figure 3-23 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeSoftLight`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_soft_light_blend_mode.png" width=80% />

#### Hard Light Blend Mode
依赖于前景图片样品颜色，要么 multyply 要么 screen colors。如果前景图片样品颜色比 50% 灰要亮，背景色被加亮，与 screening 相似。如果前景图片样品色比 50% 灰要暗，背景色被加暗，和 multiplying 相似。如果等于 50% 灰的话，前景色不变。图片样品中等于全黑或全白的区域产生纯暗或纯亮的区域。总的效果和你在前景色上闪烁一个刺眼的聚光灯效果差不多。使用它来给场景增加高亮。Figure 3-24 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeHardLight`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_hard_light_blend_mode.png" width=80% />

#### Difference Blend Mode
要么从背景图样品颜色中减去前景图样品颜色，或相反，依赖于哪一个样品有更高的亮度值。前景图样品颜色值是黑色时不产生改变；白色会反转背景色。Figure 3-25 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeDifference`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_difference_blend_mode.png" width=80% />

#### Exclusion Blend Mode
产生与 `kCGBlendModeDifference` 效果相似的结果，但具有更低的对比度。前景图样品值是黑色时不会产生改变；白色会反转背景色。Figure 3-26 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeExclusion`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_exclusion_blend_mode.png" width=80% />

#### Hue Blend Mode
使用背景色的 luminance, saturation 值和前景色的 hue 值。Figure 3-27 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeHue`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_hue_blend_mode.png" width=80% />

#### Saturation Blend Mode
指定使用背景色的 luminance, hues 值和前景色的 saturation 值。背景色中没有 saturation 的部分 (也就是，纯 gray 区域) 不会产生改变。Figure 3-28 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeSaturation`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_saturation_blend_mode.png" width=80% />

#### Color Blend Mode
使用背景色的 luminace 值和前景色的 hue, saturation 值。这个模式会保存图像中的 gray levels。你可以使用这个模式来 color monochrome images 或 tint color images。Figure 3-29 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeColor`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_color_blend_mode.png" width=80% />

#### Luminosity Blend Mode
使用背景色的 hue, saturation 值和前景色的 luminance 值。这个模式创建一个与 `kCGBlendModeColor` 效果相反的效果。Figure 3-30 显示了结果图片。要使用这个模式，使用常量 `kCGBlendModeLighten` 调用函数 `kCGBlendModeLuminosity`。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/3path/rectangles_painted_using_luminosity_blend_mode.png" width=80% />

## Clipping to a Path
从一个 path 创建的 **current clipping area** 会起 mask 的作用，允许你屏蔽掉页面你不想上色的部分。例如，如果你有一个很大的位图并且想要展示它的一小部分，你可以设置 clipping area 为你想要展示部分。

当你绘制时，Quartz render 只会给 clipping area 内上色。在 clipping are 的闭合 subpaths 内绘制可见，之外的不可见。

当 graphics context 刚被创建时，clipping area 包含 context 所有的可绘制区域  (例如 PDF context 的 media box)。你通过设置当前 path 然后使用一个 clipping 函数而不是绘制函数。clipping 函数会对当前 path 的填充区域和现有的 clipping area 求交集。因此，你可以与 clipping area 相交，来缩小图片的可见区域，但是你不能增加 clipping are 的区域。

clipping area 是 graphics state 的一部分。要恢复 clipping area 到之前的状态，你可以在你 clip 之前保存 graphics state，然后在你完成 clipped drawing 之后恢复这个 graphics state。

下面的代码片段设置了一个圆形的 clipping area。这个代码会使绘制被 clipped。

```objc
CGContextBeginPath (context);CGContextAddArc (context, w/2, h/2, ((w>h) ? h : w)/2, 0, 2*PI, 0);CGContextClosePath (context);CGContextClip (context);
```

下表列出了裁剪 graphics context 的函数

Funcation | Description 
-------------- | -----------------
`CGContextClip` | 使用 nonzero winding number rule 来计算当前 path 与当前 clipping path 的交集
`CGContextEOClip` | 使用 even-odd rule 来计算当前 path 与当前 clipping path 的交集
`CGContextClipToRect` | 设置 clipping area 为指定的矩形与当前 clipping path 的交集
`CGContextClipToRects` | 设置 clipping area 为指定的多个矩形框与当前 clipping path的交集
`CGContextClipToMask` | 映射一个 mask 到指定的矩形框中，将它与当前 graphics context 的 clipping area 求交集。任何接下来你对 graphics context 进行的 path 绘制将被 clipped。

---
# Color and Color Spaces
设备 (显示器，打印机，扫描仪，摄像机) 并不以相同的方式对待颜色；每个设备都一个它可以如实产生的颜色范围。一个在一台设备上产生的颜色可能不能在另一台设备上产生。

为了有效的与颜色一起工作和理解 Quartz 2D 用于颜色和颜色空间的函数，你应该熟悉 *Color Management Overview* 中描述的术语。那篇文档讨论了 color perception, color values, device-independent and device color spaces，color-matching problem, renderring intent, color management modules, and ColorSync.

在这章，你将学到 Quartz 是怎样表示颜色和颜色空间，alpha component 是什么，这章也讨论了怎么

* 创建颜色空间
* 创建和设置颜色
* 设置 rendering intent

## About Color and Color Spaces
在 Quartz 中的一个颜色是通过值的集合表示的。这些值在没有颜色空间来知道怎么翻译颜色信息时是没有意义的。例如，在下表中的值都表示最强的蓝色。但是不知道颜色空间或对于每个颜色空间可用的范围时，你是没有办法知道这些值的集合代表什么的。

Values | Color space | Components
--------- | ----------------- | -------------------
240 degress, 100%, 100% | HSB | Hue, saturation, brightness
0, 0, 1 | RGB | Red, green, blue
1, 1, 0, 0 |  CMYK | Cyan, magenta, yellow, black
1, 0, 0 | BGR | Blue, green, red

如果你提供了错误的颜色空间的话，你可能获得相当戏剧化的不同结果，就像 Figure 4-1 中显示的那样。尽管绿颜色在 BGR 和 RGB 颜色空间中翻译得相同，但 red 和 blue 反转了。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/4color/applying_a_bgr_and_an_rgb_color_profile_to_the_same_image.png" width=80% />

颜色空间可以有不同的数量的组成成分。表中的三个颜色空间有三个组成成分，然而 CMYK 颜色空间有 4 个。值的范围是相对于颜色空间而言的。对于大部分颜色空间，在 Quartz 中的颜色值范围是从 0.0 到 1.0，1.0 意味着强烈程度最强。例如，蓝色的最强强度在 Quartz 的 RGB 颜色空间中值为 (0, 0, 1.0)。在 Quartz 中，颜色也有一个 alpha 值来指定颜色的透明度。上表中的颜色没有显示 alpha 值。

## The Alpha Value
**alpha value** 是 Quartz 用来决定怎么组合新绘制的对象到现有页面的 graphics state 参数。强度最强时，新绘制的对象是不透明的。强度最弱时，新绘制的对象是不可见的。Figure 4-2 显示了 5 个大矩形使用 1.0, 0.75, 0.5, 0.1和 0.0 的 alpha 值来绘制。因为大矩形渐渐变得透明，它会显示出一个在它之下的一个更小的，不透明的红色矩形。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/4color/a_comparison_of_large_rectangles_painted_using_various_alpha_values.png" width=80% />

你可以同时使页面上的对象和页面自身透明，在绘制前通过设置 graphics context 中全局的 alpha 值。Figure 4-3 比较了全局设置为 1.0 和 0.5 的不同结果。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/4color/a_comparison_of_global_alpha_values.png" width=80% />

在 normal blend mode (正是 graphics state 的默认值)，Quartz 通过以下公式组合 source color 的组成成分和 destination color 的组成成分:

```objc
destination = (alpha * source) + (1 - alpha) * destination
```

这里 `source` 新绘制颜色的一个组成成分，`destination` 是背景色的一个组成成分。这个方程会应用每个新绘制的形状或图片。

对于对象透明度，设置 alpha 值为 1.0 时指定你绘制的这个对象应该是不透明的；设置它为 0.0 时指定新绘制的对象为完全透明的。一个介于 0.0 与 1.0 的 alpha 值指定了一个部分透明的对象。你可以提供 alpha 值给任何接受颜色的例程。你也可以通过调用 `CGContextSetAlpha` 来设置全局的 alpha 值。如果你同时设置了的话，Quartz 会将你提供的 alpha 颜色组成成分乘以全局 alpha 值。

要允许自身是全透明的话，你可以使用函数 `CGContextClearRect` 显示清除 graphics context 的 alpha channel，只要这个 graphics context 是一个 window 或 bitmap graphics context。当给一个 icon 创建一个透明 mask 时你也许想要这么做，例如，要使一个 window 的背景透明。

## Creating Color Spaces
Quartz 支持颜色管理系统用于设备不依赖的标准颜色空间，也支持 generic, indexed, and pattern color spaces。**Device-independent color spaces** 以一种设备间可移植的方式来表示颜色。它们被用来从一个设备的本地颜色空间到另一个设备的颜色空间交换数据。在一个不依赖设备的颜色空间中的颜色在不同的设备上显示的时候是一样的，至少会尽设备能力允许的可能。因为这个原因，独立于设备的颜色空间是你代表颜色的做好选择。

有准确颜色要求的应用应该总是使用一个不依赖设备的颜色空间。一个通用的不依赖设备的颜色空间是 **generic color space**。Generic color spaces 让操作系统为你的应用提供最好的颜色空间。绘制到屏幕和打印相同的颜色到一个打印机开起来一样好。

> **重要**: iOS 不提供设备独立或 generic color spaces. 相反 iOS 应用必须使用设备颜色空间。

### Creating Device-Independent Color Spaces
要创建一个不依赖设备的颜色空间，你给 Quartz 提供一个特定设备的 white point referenc, black point reference, and gamma values。Quartz 使用这些信息将来你的源颜色空间的颜色转换到输出设备的颜色空间。

创建 Quartz 支持的不依赖设备的颜色空间如下:

* L\*a\*b\* 是一个 Munsell color notation system (一个通过 hue, value, 和 saturation——或 chroma ——值来指定颜色的系统) 的非线性变换。这个颜色空间使用颜色空间中的数据距离来匹配视觉上的颜色不同。L\* 组成承贷代表亮度值，a\* 组成成分代表从绿色到红色的值，b\* 代表从蓝到黄的值。这个颜色空间用来模仿人类大脑解码颜色的方式。使用函数 `CGColorSpaceCreateLab`。
* ICC 是来自 ICC 颜色 profile 的一个颜色空间，由 International Color Consortium 所定义。ICC profiles 定义了设备支持颜色的 gamut，和一些其他设备的特点，以便这个信息可以用来在设备间的颜色空间内做转换。设备的生产者通常提供一个 ICC profile。一些彩色监控器和打印机包含内嵌的 ICC profile 信息，和一些位图格式，如 TIFF。使用函数 `CGColorSpaceCreateICCBased`。
* Calibrated RGB 是一个不依赖设备的 RGB 颜色空间，它相对于 a reference white point 来表示颜色，这个 reference white point 是基于一个输出设备可以产生的最白的光。使用函数 `CGColorSpaceCreateCalibratedRGB` 创建。
* Calibrated gray 是一个不依赖设备的灰度颜色空间，它相对于 a reference white point 来表示颜色，这个 reference white point 是基于一个输出设备可以产生的最白的光。使用函数 `CGColorSpaceCreateCalibratedGray` 创建。

### Creating Generic Color Spaces
Generic color space 将颜色匹配交予系统。对于大部分情况，结果是可接受的。尽管这个名字隐含着另一层意思，每个 "generic" 颜色空间——generic gray, generic RGB, and generic CMYK——是一个不依赖设备的颜色空间。

Generic 颜色空间很容易使用；你不需要提供任何 reference point 信息。你通过以下常量使用函数 `CGColorSpaceCreateWithName`:

* `kCGColorSpaceGenericGray`, 指定 generic gray, a monochromatic 颜色空间，这个空间允许指定一个从绝对黑色到绝对白色范围的一个值。
* `kCGColorSpaceGenericRGB`, 指定 generic RGB, a three-component 颜色空间 (red, green and blue) 跟一个像素在屏幕上显示时相似。RGB 颜色的每个组成成分值是从 0.0 到 1.0
* `kCGColorSpaceGenericCMYK` 指定 generic CMYK，一个 four-component 颜色空间 (cyan, magenta, yellow, black)，模拟了在打印过程中墨水构建的过程。CMYK 颜色空间的每个颜色组成成分的值范围是从 0.0 到 1.0.

### Creating Device Color Spaces
设备颜色空间主要被 iOS 应用所使用，因为其他的选项不可用。在大部分情况下，一个 OS X 应用使用一个 generic 颜色空间而不是创建一个设备颜色空间。然而，一些 Quartz 例程会期望图片有一个设备颜色空间。例如，如果你调用 `CGImageCreateWithMask` 函数并指定一个图片作为 mask，这张图片必须使用设备 gray 颜色空间定义。

你通过调用以下函数之一来创建一个设备颜色空间：

* `CGColorSpaceCreateDeviceGray` 来创建一个依赖设备的 grayscale 颜色空间。
* `CGColorSpaceCreateDeviceRGB` 来创建一个依赖设备的 RGB 颜色空间
* `CGColorSpaceCreateDeviceCMYK` 来创建一个依赖设备的 CMYK 颜色空间

### Creating Indexed and Pattern Color Spaces
Indexed color space 一个最多 256 个条目的颜色表，和一个颜色表条目映射时所基于的基本颜色空间。颜色表中的每个条目指定了在 base color space 中的一个颜色。使用函数 `CGColorSpaceCreateIndexed` 来创建。

Pattern color spaces, 在本文的 **Patterns** 中有介绍。当使用 patterns 来绘制的时候使用。使用函数 `CGColorSpaceCreatePattern` 来创建。

## Setting and Creating Colors
Quartz 提供了一组函数来设置 fill color, stroke color, color spaces, and alpha. 这些颜色参数应用到 graphics state，也就是意味着一旦设置，这些参数会持续到被设置为另外的参数。

一个颜色必须有一个相关联的颜色空间。否则，Quartz 就不知道怎么翻译颜色值。并且，你需要为绘制目的地提供一个合适的颜色空间。对比 Figure 4-4 中左边的蓝色填充色，是一个 CMYK 填充色，而右边的蓝颜色，是一个 RGB 填充色。如果你在屏幕上看这个文档的话，你会看到这两个填充色之间的去边。在理论上颜色是一样的，但是只会在 RGB 颜色被用于一个 RGB 设备，CMYK 颜色被用于一个 CMYK 设备时才会看上去一样。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/4color/a_cmyk_fill_color_and_an_rgb_fill_color.png" width=80% />

你可以使用函数 `CGContextSetFillColorSpace` 和 `CGContextSetStrokeColorSpace` 来设置填充和 stroke 的颜色空间，或使用下表中的简便方法来为一个设备颜色空间设置颜色。

Funcation | Use to set color for
-------------- | ---------------------------
`CGContextSetRGBStrokeColor` <br />`CGContextSetRGBFillColor` | Device RGB。在 PDF-generation 的时候，Quartz 像颜色在相应的 generic color space 一样写入它们。
`CGContextSetCMYKStrokeColor` <br /> `CGContextSetCMYKFillColor` | Device CMYK (在 PDF-generation 的时候，保持设备 CMYK 颜色空间)
`CGContextSetGrayStrokeColor` <br /> `CGContextSetGrayFillColor` | Device Gray。在 PDF-generation 的时候，Quartz 像颜色在相应的 generic color space 一样写入它们。
`CGContextSetStrokeColorWithColor` <br /> `CGContextSetFillColorWithColor` | 任何颜色空间；你提供一个 `CGColor` 对象指定颜色空间。对于你需要重复使用的颜色使用这两个函数。
`CGContextSetStrokeColor` <br /> `CGContextSetFillColor` | 当前的颜色空间。不推荐按，相反，使用一个 `CGColor` 对象和函数 `CGContextSetStrokeColorWithColor` 和 `CGContextSetFillColorWithColor`

你通过 fill 和 stroke 颜色空间中的值来指定 fill 和 stroke 值。例如，一个 RGB 空间中的全饱满度的红色值为 (1.0, 0.0, 0.0, 1.0)。前三个数指定红色的强度，没有绿色，没有蓝色。第四个数是 alpha 值，用来指定颜色的不透明度。

如果你在你的应用中重用颜色，设置 fill 和 stroke 颜色的最有效的方式是创建一个 `CGColor` 对象，并将这个对象作为参数传递给 `CGContextSetFillColorWithColor` 和 `CGContextSetStrokeColorWithColor` 函数。你可以根据需要来保持这个 `CGColor` 对象。通过直接使用 `CGColor` 对象你可以提升应用的性能。

你通过调用 `CGColorCreate` 函数创建一个 `CGColor` 对象，创建时传递一个 `CGColorspace` 对象和一组指定颜色成分强度的浮点数。数据的最后一个指定 alpha 值。

## Setting Rendering Intent
rendering intent 指定了怎么映射源颜色空间中的颜色到一个 graphics context 目的颜色空间的 gamut 范围内。如果你不显示的设置 rendering intent，Quartz 为除位图图像外的所有绘制使用 relative colorimetric rendening intent。对于位图图像 Quartz 使用 perceptual rendering intent。

要设置 rendering indent，调用函数 `CGContextSetRenderingIntent`，传一个 graphics context 和以下常量之一：

* `kCGRenderingIntentDefault`. 使用默认的 rendering indent
* `kCGRenderingIntentAbsoluteColorimetric`. 将输出设备 gamut 外的颜色映射到输出设备 gamut 内的最接近的可能匹配。这可能会产生一个裁剪效果—— graphics context 中两个不同的颜色值被映射到输出设备的 gamut 中的相同值。当 graphics context 中使用的颜色同时在源和目的 gamut 内时是最好的选择，通常 logs 的情况或当 spot colors 被使用的时候就是这样。
* `kCGRenderingIntentRelativeColorimetric`. relative colorimetric 会移动所有颜色 (包括那些在 gamut 范围内的) 来消灭 graphics context 中的 white point 与输出设备中的 white point 之间的不同。
* `kCGRenderingIntentPerceptual`.  通过压缩 graphics context 的 gamut 以适应到输出设备的 gamut 范围内来保持视觉关系。Perceptual intent 比较适用于摄影图片和其他复杂丰富的图片。
* `kCGRenderingIntentSaturation`. 当转换颜色到输出设备的 gamut 时保持颜色的相对饱满度。结果是一张有亮度，饱满颜色的图片。Saturation intent 适合于重新产生细节不丰富的图片。如图表和图形。


----
# Transforms
Quartz 2D 绘制模型定义两套完全不同的坐标系空间：代表文档页面的 user space，和代表一个设备的本地分辨率的 device space。User space 中坐标是浮点数，这些浮点数与设备空间中的像素是不相关的。当你想要打印或显示你的文档时，Quartz 映射用户空间中的坐标到设备空间坐标。因此，在不同的设备上显示的时候你绝不需要重写你的一个用或写额外的代码来调整你应用的输出。

你可以通过在 **current transformation matrix** 上操作来修改默认的用户空间。在你创建一个 graphics context 之后，CTM 是 identity matrix。你可以使用 Quartz tranformation 函数来修改 CTM，结果是，修改了在用户空间的绘制。

这章：

* 提供了你可以用来进行 transformations 的函数概览
* 展示了怎么修改 CTM
* 描述了怎么创建一个 affine transform
* 展示了怎么决定是否两个 transforms 是否相等
* 描述了怎么做到 user-to-device-space transform
* 讨论了在 affine transform 背后的数据知识

## About Quartz Transformation Functions
你可以使用 Quartz  2D 内建的 transformation 函数轻易的 translate, scale, and rotate 你的绘制。只需要使用几行代码，你可以以任意顺序和组合应用这些 transformation。Figure 5-1 图示了 scalling and rotating 一张图片的效果。你应用的每个 transformation 会更新 CTM. CTM 总是用户空间和设备空间的映射关系。这个映射保证了你应用的输出在任何显示屏或打印机上看起来不错。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/applying_scaling_and_rotation.png" width=80% />

Quartz 2D API 提供了 5 个函数来允许你获取和修改 CTM。你可以 rotate, translate, scale the CTM，并且你可以追加一个 affine transformation 到 CTM 上。查看本文的 **Modifying the Current Transformation Matrix**.

Quartz 也允许你创建不会操作用户空间的 affine transformations，直到你决定要应用这个 transform 到 CTM。你使用其他函数集合来创建 affine transforms，这些 transforms 之后可做拼接。查看本文的 **Creating Affine Transforms**。

关于矩阵数学的内容啥也不知道你也可以使用任意集合中的函数。然而如果你想要理解 Quartz 在调用 transform 函数之一时是怎么做的话，查看本文的 **The Math Behind the Matrices**.

## Modifying the Current Transformation Matrix
在绘制图片前你操作 CTM 来 rotate, scale, translate 页面，因此也就是 transforming 你将绘制的对象。在你 transform CTM 之前，你需要保存 graphics state 以便你在绘制之后可以恢复它。你也可以使用一个 affine transform 来拼接 CTM。这四个操作的每一个——translation, rotation, scalling, and concatenation——在进行每个操作的 CTM 函数章节描述。

下面的一行代码绘制一张图片，假设你已经提供了一个有效的 graphics context，一个图片将被绘制到的矩形框的指针，和一个有效的 `CGImage` 对象。代码会绘制一张图片，如 Figure 5-2 中的公鸡。在你阅读接下来的部分，你将看到图片是怎么随着你的 transformations 而改变的。

```objc
CGContextDrawImage (myContext, rect, myImage);
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/an_image_that_is_not_transformed.png" width=80% />

**Translation** 移动坐标系的原点，移动的距离由你指定。你可以调用函数 `CGContextTranslateCTM` 来修改每个点的 x 和 y 坐标。Figure 5-3 显示了使用以下代码移动 x 轴 100 点，y 轴 50 点：

```objc
CGContextTranslateCTM (myContext, 100, 50);
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/a_translated_image.png" width=80% />

**Rotation** 通过你指定的角度来移动坐标系。你可以指定一个旋转角度调用函数 `CGContextRotateCTM`，以弧度为单位。Figure 5-4 显示了一张图片绕着原点旋转 -45°，原点在 window 的左下方。代码如下：

```objc
CGContextRotateCTM (myContext, radians(–45.));
```

图片被裁减了，因为旋转使得图片的部分移出了 context 的范围。你需要以弧度来指定旋转角度。

如果你计划进行任何旋转，写一个弧度例程很有用。

```objc
#include <math.h>

static inline double radians (double degrees) {return degrees * M_PI/180;}
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/a_rotated_image.png" width=80% />

**Scaling** 通过指定 x 和 y 因子来改变的坐标空间的 scale，从而有效的拉伸或缩小图片。x 和 y 因子的数量级决定了新坐标系相对于原有坐标系的大小。另外，通过使 x 因子取负，你可以沿着 x 轴翻转坐标系；相似的，你可以水平的翻转坐标，沿着 y 轴，通过使 y 因子取负。Figure 5-5 显示了一张图片的 x 值被 scaled 了.5，y 值被 scaled 了 .75，使用以下代码：

```objc
CGContextScaleCTM (myContext, .5, .75);
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/a_scaled_image.png" width=80% />

**Concatenation** 通过将两个矩阵相乘来组合它们。你可以追加几个矩阵形成一个单一的矩阵，这个矩阵包含这些举证的累积效果。你调用函数 `CGContextConcatCTM` 来组合一个 affine transform 和 CTM。Affine transforms，和创建它们的函数，在本文的 **Creating Affine Transforms** 中有讨论。

另一种达成累积效果的方式是进行两个或多个 transformations，在这些 transformations 之间不恢复 graphics state。Figure 5-6 展示了移动一张图片然后旋转它之后的结果，使用以下代码：

```objc
CGContextTranslateCTM (myContext, w,h);CGContextRotateCTM (myContext, radians(-180.));
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/an_image_that_is_translated_and_rotated.png" width=80% />

Figure 5-7 展示了一张图片被 translated, scaled, and rotated, 使用以下代码：

```objc
CGContextTranslateCTM (myContext, w/4, 0);
CGContextScaleCTM (myContext, .25,  .5);
CGContextRotateCTM (myContext, radians ( 22.));
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/an_image_that_is_translated_scaled_and_then_rotated.png" width=80% />

你进行多个 transformations 的顺序是很重要；如果逆转这个顺序你会得到不一样的结果。逆转用来创建 Figure 5-7 的 transformations 顺序，你会得到 Figure 5-8，代码如下：

```objc
CGContextRotateCTM (myContext, radians ( 22.));
CGContextScaleCTM (myContext, .25,  .5);
CGContextTranslateCTM (myContext, w/4, 0);
```

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/5transformation/an_image_that_is_rotated_scaled_and_then_translated.png" width=80% />

## Creating Affine Transforms
Quartz 中可用的 affine transform 函数操作在矩阵上，而不是 CTM。你可以使用这些函数来组建一个矩阵，之后通过调用 `CGContextConcatCTM` 应用到 CTM 上。affine transform 函数要么操作要么返回一个 `CGAffineTransform` 数据结果。捏可以组建可重用的简单或复杂的 affine transforms。

affine transform 函数和 CTM 函数进行一样的操作——translation, rotation, scaling, and concatenation. 下表列出了进行这些操作的函数和怎么使用它们的信息。注意对于每个 translation, rotation, and scaling 操作有两个函数。

Funcation | Use 
-------------- | --------
`CGAffineTransformMakeTranslation` | 从指定原点应该移动多远的 x 和 y 值组建一个新的 translation matrix
`CGAffineTransformTranslate` | 应用一个 translation 操作到一个已有的 affine transform 上
`CGAffineTransformMakeRotation` | 从一个指定坐标系应该旋转的弧度值构建一个新的旋转矩阵
`CGAffineTransformRotate` | 应用一个 rotation 造作到一个现有的 affine transform 上
`CGAffineTransformMakeScale` | 从一个指定应该拉伸或紧缩的 x 和 y 值构建一个新的 scaling 矩阵
`CGAffineTransformScale` | 应用一个 scaling 操作到一个现有的 affine transform 上

Quartz 也提供了一个 affine transform 函数来反转一个矩阵，`CGAffineTransformInvert`。Inversion 通常用来提供 transformed 对象的 reverse transformation。Inversion 在你需要恢复一个已经被矩阵 transformed 的值时很有用：invert 这个矩阵，然后将值乘以 inverted matrix，结果就是原来的值了。你通常不需要 invert transforms，因为你通过保存和恢复 graphics state 来 reverse transforming 的效果。

在一些情况下，你可能不需要 transform 整个空间，而仅仅是一个点或一小块。你通过调用函数 `CGPointApplyAffineTransform` 来操作一个 `CGPoint` 结构。你通过调用函数 `CGSizeApplyAffineTransform` 来操作一个 `CGSize` 结构。你可以通过调用函数 `CGRectApplyAffineTransform` 来操作一个 `CGRect` 结构。这个函数返回包含 transformed 矩形框四个顶点的最小矩形框。如果 affine transform 在矩形框上的操作只是 scaling 和 translation 的话，返回的矩形框与 transformed 顶点构成的矩形框一致。

你可以调用函数 `CGAffineTransformMake` 创建一个新的 affine transform，但是不像其他创建新 affine transform 的函数，这个要求你提供 matrix entries。要有效的使用这个函数，你需要理解矩阵数学。查看本文的 **The Math Behind the Matrices**。

## Evaluating Affine Transforms
你可以通过调用函数 `CGAffineTransformEqualToTransform` 来判断两个 affine transform 是否相等。相等的时候返回 `true`，否则返回 `false`。

函数 `CGAffineTransformIsIdentity` 对于检查一个 transform 是否是 *identity transform* 很有用。identity transform 不进行  translation, scaling, or rotation。应用这个 affine transform 到输入坐标总是返回输入的原坐标。Quartz 常量 `CGAffineTransformIdentity` 表示 identity transform。

## Getting the User to Device Space Transform
通常当你使用 Quartz 2D 绘制时，你只是在 user space 中工作。Quartz 负责 user space 与 device space 的转换。如果你的应用需要获取 Quartz 使用来转换 user space 到 device space 的 affine transform 的话，可以调用 `CGContextGetUserSpaceToDeviceSpaceTransform`。

Quartz 提供了一系列了简便方法来在 user space 和 device spcae 间做几何转换。你也许会发现这些函数比使用 `CGContextGetUserSpaceToDeviceSpaceTransform` 函数返回的 affine transform 更简单。

* Points. 函数 `CGContextConvertPointToDeviceSpace` 和 `CGContextConvertPointToUserSpace` 在这两个 space 间转换一个 `CGPoint` 数据
* Sizes. 函数 `CGContextConvertSizeToDeviceSpace` 和 `CGContextConvertSizeToUserSpace` 在这两个 space 间转换一个 `CGSize` 数据
* Rectangles. 函数 `CGContextConvertRectToDeviceSpace` 和 `CGContextConvertRectToUserSpace` 在这两个 space 间转换一个 `CGRect` 数据

## The Math Behind the Matrices
*因为 markdown 对数学公式的支持有限，这里就不翻译这一节了，具体请看原文档*


---
# Pattern
一个 *pattern* 是重复绘制到 graphics context 的一系列绘制操作。你可以像使用颜色一样使用 patterns。当你使用 pattern 绘制时，Quartz 将页面划分为一系列的 pattern cells，每个 cell 的大小是 pattern image 的大小，并使用你提供的回调来绘制每个 cell。Figure 6-1 展示了一个绘制到 window  graphics context 的 pattern。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_pattern_drawn_to_a_window.png" width=80% />

## The Anatomy of a Pattern
pattern cell 是一个 pattern 的基本成分。Figure 6-1 中显示的 pattern 的 pattern cell 在 Figure 6-2 中有显示。黑色的矩形框不是 pattern 的一部分，画出来是为了显示 pattern 结束的位置。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_pattern_cell.png" width=80% />

这个特定 pattern cell 的大小包括四个有色矩形框的区域和上面右面的空白，如图 Figure 6-3 所示。图中包围每个 pattern cell 的黑色矩形框并不是 cell 的一部分；它被绘制来表示 cell 的边框。当你创建一个 pattern cell 时，你定义 cell 的边界并在边界内绘制。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/pattern_cells_with_black_rectangles_drawn_to_show_the_bounds_of_each_cell.png" width=80% />

你可以指定水平方向和竖直方向上 Quartz 绘制每个 pattern cell 的开始时距离下一个的距离。Figure 6-3 中 pattern cell 绘制的时候，一个 pattern cell 的开始距离下一个 pattern cell 的距离正好是一个 pattern 的宽度，这样导致 pattern cell 紧挨着彼此。Figure 6-4 中的 pattern cells 在两个方向同时增加了空隙。你可以给每个方向指定不同的 **spacing values**。如果你使得 spacing 小于一个 pattern cell 的宽度或高度的话，pattern cell 会重叠。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/spacing_between_pattern_cells.png" width=80% />

当你绘制一个 pattern cell 时，Quartz 使用 **pattern space** 作为坐标系统。Pattern space 是一个抽象空间，在创建 pattern —— **pattern matrix** 时，通过你指定的矩阵来映射到 user space。

> **注意**: Pattern space 不同于 user space。untransformed pattern space 映射到基本的 (untransformed) user space，不管 CTM 的状态是什么。当你应用一个 transformation 到 pattern space，Quartz 只会应用 transform 到 pattern space。<br />对于一个 pattern 的默认坐标系的默认约定正是潜在的 graphics context 的那些约定。默认，Quartz 使用一个 x 轴正向向右 y 轴正向向上的坐标系。然而，UIKit 创建的 graphics context 使用一个不同的约定，y 轴正向向下。尽管这个约定通常通过拼接一个 transformation 到坐标系上，在这种情况下，Quartz 也会修改匹配的 pattern space 的约定

如果你不想 Quartz transform pattern cell，你可以指定 identity 矩阵。然而你可以通过提供一个 transformation 矩阵来实现有趣的效果。Figure 6-5 显示了 scaling Figure 6-2 中的 pattern cell 的效果。Figure 6-6 显示了旋转 pattern cell 的效果。Translating pattern cell 有点儿不易察觉。Figure 6-7 显示了 pattern 的原点，pattern cell 朝两个方向都有移动，水平和垂直，所以 pattern 不再毗邻 window。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_scaled_pattern_cell.png" width=80% />

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_rotated_pattern_cell.png" width=80% />

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_translated_pattern_cell.png" width=80% />

## Colored Patterns and Stencil (Uncolored) Patterns
**Colored patterns** 有与它们关联的内在颜色。改变了是用来创建它们的颜色轴，pattern 也便失去了它的意义。一个苏格兰式毯子 (如 Figure 6-8 中所示) 是一个 colored pattern 的例子。在一个 colored pattern 中的颜色是在 pattern cell 创建的过程中指定的，而不是 pattern 的绘制过程中的部分。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_colored_pattern_has_inherent_color.png" width=80% />

其它的 patterns 只是定义了它们的形状，因为这个原因它们可以被认为是 **stencil patterns**，uncolored pattern，甚至可以叫做 image mask。Figure 6-9 中显示的红色和黑色星星是同一个 pattern cell 的表现。cell 自身由一个形状——一个填充了的星星——组成。当这个 pattern cell 被定义的时候，没有颜色与之关联。颜色是在绘制的过程中指定的，而不是 pattern cell 创建的过程中。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_stencil_pattern_does_not_have_inherent_color.png" width=80% />

在 Quartz 2D 中可以创建两个类型的 pattern——colored 或 stencil 的。

## Tilling
**Tiling** 是渲染 pattern cells 到页面一部分的过程。当 Quartz 渲染一个 patten 到一个设备时，Quartz 也许需要调整 pattern 来适应设备空间。也就是说，在 user space 中定义的 pattern cell 因为 user space 单位与设备像素的不同在渲染的时候可能不能完美的适配。

必要的时候 Quartz 有三个 tiling 选项可以使用来调整 patterns，Quartz可以保持：

* The pattern，以轻微的调整 pattern cell 间的空隙为代价，不不会超过一个像素。这被称为 **no distortion**。
* Spacing between cells，以轻微的变形 pattern cell 为代价，但不会超过一个设备像素。这被称为 **constant spacing with minimal distortion**。
* Spacing between cells (和 minimal distortion option 一样)，以变形 pattern cell 为代价以便最快的 tilling cell。这被称为 **constant spacing**。

## How Patterns Work
从你设置一个填充和 stroke pattern 然后调用绘制函数的角度而言 Patterns 操作起来跟颜色差不多。Quartz  使用你设置的 pattern 作为 “画笔”。例如，如果你想要绘制一个有固定填充色的矩形框，你先调用一个函数，如 `CGContextSetFillColor`，来设置填充色。然后你调用函数 `CGContextFillRect` 来使用你指定的颜色填充矩形框。要使用 pattern 来绘制，首先你调用函数 `CGContextSetFillPattern` 来设置 pattern。然后你调用 `CGContextFillRect` 函数来使用你指定的 pattern 给矩形框被填充区域上色。使用颜色绘制与使用 patterns 的区别是你必须设置这个 pattern。你提供 pattern 和颜色信息给函数 `CGContextSetFillPattern`。在本文的 **Painting Colored Patterns** 和 **Painting Stencil Patterns** 你将看到怎么创建，设置和绘制 pattern。

下面是一个 Quartz 幕后怎么使用你提供的 pattern 来进行工作。当年你使用一个 pattern 来填充或 stroke 时，理论上 Quartz 会进行一下任务来绘制每个 pattern cell：

1. 保存 graphics state
2. 移动 current transformation matrix 到 pattern cell 的原点。
3. 拼接 pattern matrix 到 CTM
4. 裁剪 graphics context 到 pattern cell 的矩形框
5. 调用你的绘制回调来绘制 pattern cell
6. 恢复 graphics state

Quartz 会负责所有的 tiling，重复的渲染 pattern cell 到绘制空间直到整个空间被上色。你可以使用一个 pattern 填充或 stroke。pattern cell 可以你指定的任何大小。如果你想要查看 pattern，你应该确认 pattern cell 在绘制空间内是大小合适的。例如，如果你的 pattern cell 是 8 x 10 大小，并且你使用 pattern 来 stroke 一个宽度为 2 的直线，pattern cell 会被裁剪，因为它的宽为 10。在这种情况下，你可能识别不出来 pattern。

## Painting Colored Patterns
你需要进行绘制一个 colored pattern 的五步在下面的章节有描述：

1. **Write a Callback Function That Draws a Colored Pattern Cell**
2. **Set Up the Colored Pattern Color Space**
3. **Set Up the Anatomy of the Colored Pattern**
4. **Specify the Colored Pattern as a Fill or Stroke Pattern**
5. **Draw With the Colored Pattern**

你使用来绘制一个 stencil pattern 的步骤是相同的。两者之间的区别是你怎么设置颜色信息。你可以在 **A Complete Colored Pattern Painting Function** 查看这些步骤是怎么串在一起的。

### Write a Callback Function That Draws a Colored Pattern Cell
一个 pattern cell 看起来是怎样的完全取决于你。例如，下面的代码 Figure 6-2 中的 pattern cell。别忘了包围 pattern cell 的黑线并不是 cell 的一部分；它被绘制出来是为了显示 pattern cell 的边界比代码绘制的矩形框要大。稍后你指定 pattern 的大小到 Quartz。

你的 pattern cell 绘制函数是一个如下格式的回调：

```objc
typedef void (*CGPatternDrawPatternCallback) (void *info,CGContextRef context);
```

你可以随意命名你的回调。回调函数有两个参数：

* `info`，一个通用指针提供与 pattern 关联的数据。这个参数是可选的；你可以传递 `NULL`。传递给回调的数据跟你稍后提供的数据一样，当你创建 pattern 的时候。
* `context`，用来绘制 pattern cell 的 graphics context。

在代码中绘制的 pattern cell 是随意的。你的代码可以绘制任何对于你创建的 pattern 合适的内容。关于代码的详情很重要：

* pattern size 被申明好。在你编写你的绘制代码时，你需要记住 pattern 的大小。这里 pattern 的大小被申明为全局的。绘制函数没有具体的指定大小，除了在评论中。稍后，你指定 pattern size 到 Quartz 2D。查看本文的 **Set Up the Anatomy of the Colored Pattern**。
* 紧跟 `CGPatternDrawPatternCallback` 定义的原型是绘制函数
* 函数中的绘制代码设置了颜色，使得 pattern 是一个 colored pattern。

```objc
#define H_PATTERN_SIZE 16#define V_PATTERN_SIZE 18

void MyDrawColoredPattern (void *info, CGContextRef myContext) {
    CGFloat subunit = 5; // the pattern cell itself is 16 by 18
    CGRect  myRect1 = {{0,0}, {subunit, subunit}},
        myRect2 = {{subunit, subunit}, {subunit, subunit}},        myRect3 = {{0,subunit}, {subunit, subunit}},        myRect4 = {{subunit,0}, {subunit, subunit}};
        
    CGContextSetRGBFillColor (myContext, 0, 0, 1, 0.5);    CGContextFillRect (myContext, myRect1);    CGContextSetRGBFillColor (myContext, 1, 0, 0, 0.5);    CGContextFillRect (myContext, myRect2);    CGContextSetRGBFillColor (myContext, 0, 1, 0, 0.5);    CGContextFillRect (myContext, myRect3);    CGContextSetRGBFillColor (myContext, .5, 0, .5, 0.5);    CGContextFillRect (myContext, myRect4);
}
```

### Set Up the Colored Pattern Color Space
上面的代码使用颜色来绘制 pattern cell。你必须通过设置基本 pattern color space 为 `NULL` 来保证 Quartz 使用你的绘制例程中使用的颜色来绘制，如下面的代码所示。关于每行代码的详细解释紧随其后。

```objc
CGColorSpaceRef patternSpace;

patternSpace = CGColorSpaceCreatePattern (NULL);                          //1CGContextSetFillColorSpace (myContext, patternSpace);                  //2CGColorSpaceRelease (patternSpace);                                                   //3```

下面是代码所做的：

1. 传递参数 `NULL` 作为基本颜色空间，调用函数 `CGColorSpaceCreatePattern` 来创建一个适合于一个 colored pattern 的颜色空间。
2. 设置填充色的颜色空间为 pattern color space。如果你使用 pattern 来 stroke 的话，调用 `CGContextSetStrokeColorSpace`
3. 释放 pattern color space。

### Set Up the Anatomy of the Colored Pattern
关于一个 pattern 结构的信息被保持在一个 `CGPattern` 对象上。你通过调用函数 `CGPatternCreate` 来创建一个 `CGPattern` 对象。

```objc
CGPatternRef CGPatternCreate (void *info, 
                                                            CGRect bounds,
                                                            CGAffineTransform matrix,
                                                            CGFloat xStep,
                                                            CGFloat yStep,
                                                            CGPatternTiling tiling,
                                                            bool isColored,
                                                            const CGPatternCallbacks *callbacks );
```

`info` 参数是一个指针，指向你想要传递给绘制回调的数据。正是我们在本章的 **Write a Callback Function That Draws a Colored Pattern Cell** 中讨论的同一个指针。

你在 `bounds` 参数中指定 pattern cell 的大小。`matrix` 参数是你指定 pattern matrix 的地方，它会将 pattern 坐标系映射到 graphics context 的默认坐标系，如果你想要使用与 graphics context 相同的坐标系的话使用 identity matrix。`xStep` 和 `yStep` 参数在 pattern 坐标系中指定 cells 间的水平和垂直空隙。查看本文的 **The Anatomy of a Pattern** 来查看关于 bounds, pattern matrix, and spacing 的更多信息。

titling 参数可以是以下三个值之一：

* `kCGPatternTilingNoDistortion`
* `kCGPatternTilingConstantSpacingMinimalDistortion`
* `kCGPatternTilingConstantSpacing`

查看本文的 **Tiling** 来了解更多的信息。

`isColored` 参数指定了 pattern cell 是一个 colored pattern 还是 stencil pattern。如果你传递 `true`, 你的绘制 pattern 回调指定 pattern 的颜色，并且你必须设置 pattern color space 为 colored pattern color space (查看本文的 **Set Up the Colored Pattern Color Space**)。

传递给函数 `CGPatternCreate` 的最后一个参数是一个指向 `CGPatternCallbacks` 数据结构的指针。这个结构有以下三个字段：

```objc
struct CGPatternCallbacks {
    unsigned int version;    CGPatternDrawPatternCallback drawPattern;    CGPatternReleaseInfoCallback releaseInfo;
}
```

你设置 `version` 为 0。`drawPattern` 是一个指向你的绘制回调的指针。`releaseInfo` 字段指向一个回调，这个回调在 `CGPattern` 对象被释放的时候调用，用来释放你传递给你的绘制回调的 `info` 字段。如果你没有通过 `info` 字段传递任何数据的话，这个字段你设置为 `NULL`。

### Specify the Colored Pattern as a Fill or Stroke Pattern
通过调用合适的函数——`CGContextSetFillPattern` 或 `CGContextSetStrokePattern` 你可以使用你的 pattern 来填充或 stroking。对于接下来的填充或 stroking，Quartz 使用你的 pattern。

这些函数接受三个参数：

* graphics context
* 你之前创建的 `CGPattern` 对象
* color components 的数据。

尽管 colored pattern 提供了它们自身的颜色，你必须传递一个单一的 alpha 值来通知 Quartz 在绘制的时候 pattern 总的透明度是怎样的。Alpha 的值从 1 到 0。这些代码展示了怎么设置用来填充的 colored pattern 的透明度。

```objc
CGFloat alpha = 1;
CGContextSetFillPattern (myContext, myPattern, &alpha);
```

### Draw With the Colored Pattern
在你完成之前的步骤后，你可以调用任何 Quartz 2D 函数来绘制。你的 pattern 被用来上色。例如，你可以调用 `CGContextStrokePath`，`CGContextFillPath`，`CGContextFillRect` 或任何其它绘制函数。

### A Complete Colored Pattern Painting Function
下面的代码绘制一个 colored pattern。函数包含上面讨论的所有步骤。关于每行代码的具体解释紧随其后。

```objc
void MyColoredPatternPainting (CGContextRef myContext, CGRect rect) {
    CGPatternRef pattern;                                                                                       //1      
    CGColorSpaceRef patternSpace;                                                                   //2
    CGFloat alpha = 1,                                                                                             //3
                    width, height;                                                                                      //4
    static const CGPatternCallbacks callbacks = {0,  &MyDrawPattern,NULL}; // 5
    
    CGContextSaveGState (myContext);
    patternSpace = CGColorSpaceCreatePattern (NULL);                              //6
    patternSpace = CGColorSpaceCreatePattern (NULL);                              //7
    CGColorSpaceRelease (patternSpace);                                                       //8
    
    pattern = CGPatternCreate (NULL,                                                                //9
                                                    CGRectMake (0, 0, H_PSIZE, V_PSIZE),        //10
                                                    CGAffineTransformMake (1, 0, 0, 1, 0, 0),    //11
                                                    H_PATTERN_SIZE,                                            //12
                                                    H_PATTERN_SIZE,                                            //13
                                                    kCGPatternTilingConstantSpacing,             //14
                                                    true,                                                                    //15
                                                    &callbacks);                                                       //16
                                                    
    CGContextSetFillPattern (myContext, pattern, &alpha);                          //17
    CGPatternRelease (pattern);                                                                          //18
    CGContextFillRect (myContext, rect);                                                          //19
    CGContextRestoreGState (myContext);
}
```

代码做的内容如下：

1. 为稍后创建的 `CGPattern` 对象申明内存
2. 为稍后创建的 pattern 颜色空间申明内存
3. 申明 alpha 变量且设置其为 1.0，这指定了 pattern 的透明度为完全不透明。
4. 申明变量来保存 window 的宽和高。在这个例子中，pattern 在 window 的区域内绘制。
5. 申明和填充一个数据结果，传递 0 作为它的版本值，一个指向绘制回调函数的指针。这个例子并不提供一个 release info 回调，所以传递 `NULL`
6. 创建一个 pattern color space 对象，设置 pattern 的 base color space 为 `NULL`。当你绘制一个 colored pattern 时，pattern 在它的绘制回调中提供自己的颜色，这也是为什么设置 color space 为 NULL。
7. 设置填充颜色空间为你刚刚创建的颜色空间对象。
8. 释放 pattern color space 对象。
9. 传递 `NULL` 因为 pattern 并不需要任何额外的信息传递给绘制回调。
10. 传递一个 `CGRect` 对象来指定 pattern cell 的边界
11. 传递一个 `CGAffineTransform` 来指定怎么翻译 pattern space 到默认的 user space。这个例子传递了 identity matrix。
12. 传递水平 pattern size 作为每个 cell 之间的水平位置。在这个例子中，每个 cell 彼此相连。
13. 传递竖直 pattern size 作为每个 cell 竖直放置位置。
14. 传递常量 `kCGPatternTilingConstantSpacing` 来指定 Quarz 应该怎么渲染 pattern。更多信息，请参见本文的 **Tiling**
15. 传递 `true` 给 `isColored` 参数，指定 pattern 为一个 colored pattern。
16. 传递一个指向回调结构的指针，这个结构包含版本信息，一个指向绘制回调的函数。
17. 传递 context，你刚创建的 `CGPattern` 对象，和一个指向 Quartz 是用来指定所以用 pattern 透明度的 alpha 值。
18. 释放 `CGPattern` 对象。
19. 填充传递给 `MyColoredPatternPainting` 的矩形框。Quartz 使用你设置的 pattern 来填充矩形框。


## Parinting Stencil Patterns
绘制一个 stencil pattern 需要以下 5 步：

1. **Write a Callback Function That Draws a Stencil Pattern Cell**
2. **Set Up the Stencil Pattern Color Space**
3. **Set Up the Anatomy of the Stencil Pattern**
4. **Specify the Stencil Pattern as a Fill or Stroke Pattern**
5. **Drawing with the Stencil Pattern**

这些步骤实际上跟你是用来绘制一个 colored pattern 的步骤是一样的。两者的区别是你怎么设置颜色信息。你可以在 **A Complete Stencil Pattern Painting Function** 里查看这所有的步骤是怎么在一起的。

### Write a Callback Function That Draws a Stencil Pattern Cell
你为绘制一个 stencil pattern 所编写的回调，与之前编写一个 colored pattern cell 的形式一样。具体查看本文的 **Write a Callback Function That Draws a Colored Pattern Cell**。唯一的不同是你的绘制代码不指定颜色。Figure 6-10 中的 pattern cell 并不从它的绘制回调中获得它的颜色。颜色通过 pattern 颜色空间外设置。

<img src="https://raw.githubusercontent.com/0oneo/ImageArchive/master/drawingwithquartz2d/6patterns/a_stencil_pattern_cell.png" width=80% />

下面的代码绘制了 Figure 6-10 中的 pattern cell。代码简单的创建了一个 path 并填充这个 path。代码中没有设置颜色。

```objc
#define PSIZE 16 // size of the pattern cell

static void MyDrawStencilStar (void *info, CGContextRef myContext) {
    int k;    double r, theta;
    
    r = 0.8 * PSIZE / 2;
    theta = 2 * M_PI * (2.0 / 5.0); // 144 degrees

    CGContextTranslateCTM (myContext, PSIZE/2, PSIZE/2);
        CGContextMoveToPoint(myContext, 0, r);
    for (k = 1; k < 5; k++) {
        CGContextAddLineToPoint (myContext,
                        r * sin(k * theta),
                        r * cos(k * theta));
    }
    CGContextClosePath(myContext);
    CGContextFillPath(myContext);
}  
```

### Set Up the Stencil Pattern Color Space
Stencil patterns 要求你给 Quartz 设置一个 pattern color space 来绘制，如下面的代码所示，具体的解释紧随代码之后。

```objc
CGPatternRef pattern;CGColorSpaceRef baseSpace;CGColorSpaceRef patternSpace

baseSpace = CGColorSpaceCreateWithName (kCGColorSpaceGenericRGB);   //1
patternSpace = CGColorSpaceCreatePattern (baseSpace);               //2
CGContextSetFillColorSpace (myContext, patternSpace);                 //3
CGColorSpaceRelease(patternSpace);                                                   //4
CGColorSpaceRelease(baseSpace);                                                       //5
```

下面是代码所做的：

1. 这个函数创建一个通用的 RGB 空间。Generic color space 将颜色匹配交予系统。更多信息，请参见本文的 **Creating Generic Color Spaces**
2. 创建一个 pattern color space。你提供的颜色空间指定 pattern 的颜色是怎么表示的。稍后，当你给 pattern 设置颜色的时候，你必须使用 pattern color space 来设置。对于这个例子，你将需要使用 RGB 只来指定颜色。
3. 设置填充 pattern 时使用的颜色空间。你可以通过调用函数 `CGContextSetStrokeColorSpace` 来设置 stroke color。
4. 释放不用的 pattern color space
5. 释放 base color space

### Set Up the Anatomy of the Stencil Pattern
指定一个 stencil pattern 的结构和一个 colored pattern 是一样的——通过调用函数 `CGPatternCreate`。唯一的不同是给 `isColored` 参数传递 `false` 属性。更多信息查看 **Set Up the Anatomy of the Colored Pattern**。

### Specify the Stencil Pattern as a Fill or Stroke Pattern
你可以调用合适的函数 —— `CGContextSetFillPattern` 或 `CGContextSetStrokePattern`—— 来设置你的填充或 stroking 颜色。对于接下来的填充或 stroking，Quartz 使用你的 pattern。

这些函数有三个参数：

* graphics context
* `CGPattern` 对象
* 一组颜色值。

一个 stencil pattern 在绘制回调中并不提供颜色，所以你必须提供一个颜色来填充或 stroke 函数来通知 Quartz 使用什么颜色。下面的代码展示了给一个 stencil pattern 设置颜色。颜色数组中的值被 Quartz 在你之前设置的颜色空间翻译。因为这个例子使用 device RGB，颜色数据包含红蓝绿组成成分。最后一个成分指定了颜色的透明度。

```objc
static const CGFloat color[4] = { 0, 1, 1, 0.5 }; //cyan, 50% transparent

CGContextSetFillPattern (myContext, myPattern, color);
```

### Drawing with the Stencil Pattern
在你完成了之前的步骤之后，你可以调用任何 Quartz 2D 函数来绘制。你的 pattern 被用来上色。例如，你可以调用 `CGContextStrokePath`，`CGContextFillPath`，`CGContextFillRect`，或其他绘制函数。

### A Complete Stencil Pattern Painting Function
下面的代码包含一个函数绘制一个 stencil pattern。函数包含之前讨论的所有步骤。关于每行代码的具体解释紧随其后。

```objc
#define PSIZE 16

void MyStencilPatternPainting (CGContextRef myContext, const Rect *windowRect) {
    CGPatternRef pattern;
    CGColorSpaceRef baseSpace;
    CGColorSpaceRef patternSpace;
    static const CGFloat color[4] = { 0, 1, 0, 1 };                                          //1
    static const CGPatternCallbacks callbacks = {0, &drawStar, NULL};//2
    
    baseSpace = CGColorSpaceCreateDeviceRGB ();                             //3
    patternSpace = CGColorSpaceCreatePattern (baseSpace);            //4
    CGContextSetFillColorSpace (myContext, patternSpace);              //5
    CGColorSpaceRelease (patternSpace);
    CGColorSpaceRelease (baseSpace);
    pattern = CGPatternCreate(NULL, CGRectMake(0, 0, PSIZE, PSIZE), //6
                                CGAffineTransformIdentity, PSIZE, PSIZE,
                                kCGPatternTilingConstantSpacing,
                                false, &callbacks);
    CGContextSetFillPattern (myContext, pattern, color);                         //7
    CGPatternRelease (pattern);                                                                     //8
    CGContextFillRect (myContext,CGRectMake (0,0,PSIZE*20,PSIZE*20)); //9
}
```

下面是上面代码的解释：

1. 申明一个数据来持有一个颜色值，并设置颜色为 RGB 颜色空间中的不透明绿。
2. 申明并设置一个回调结构，传 0 作为版本参数和一个指向绘制回调函数的指针。这个例子并不提供一个 release info 回调，所以这个字段被设置为 `NULL`.
3. 创建一个 RBG 设备颜色空间。如果 pattern 被绘制到屏幕，你需要提供这种类型的颜色空间。
4. 从 RGB 设备颜色空间创建一个 pattern 颜色空间。
5. 设置填充颜色空间为你刚创建的 pattern color space。
6. 创建一个 pattern 对象。注意倒数第二个参数——`isColored`——是 `false`。Stencil pattern 并不提供颜色，所以你必须传递 `false` 给这个参数。所有其他的参数和 colored pattern 的参数相似。
7. 设置填充 pattern，传递之前申明的颜色数组。
8. 释放 `CGPattern` 对象。
9. 填充一个矩形框。Quartz 使用你刚设置的 pattern 来填充的矩形框。

----
# Shadows

## How Shadows Work

## Shadow Drawing Conventions Vary Based on the Context

## Painting with Shadows

----
# Gradients

## Axial and Radial Gradient Examples

## A Comparison of CGShading and CGGradient Objects

## Extending Color Beyond the End of a Gradient

## Using a CGGradient Object

## Using a CGShading Object

### Painting an Axial Gradient Using a CGShading Object

### Painting a Radial Gradient Using a CGShading Object

## See Also

----
# Transparency Layers

## How Transparency Layers Work

## Painting to a Transparency Layer

----
# Data Management in Quartz 2D

## Moving Data into Quartz 2D

## Moving Data out of Quartz 2D

## Moving Data Between Quartz 2D and Core Image in Mac OS X

