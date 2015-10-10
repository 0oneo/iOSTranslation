# Introduction to Animation Programming Guide for Cocoa
Cocoa 提供了设施来在有限的时间或无限的给某些类型的操作做动画。`NSAnimation` 类提供的基本动画支持专注于给你提供动画计时和管理。尽管 “animation” 这个词可能
使你想到动漫或其他形式的电影，动画对象不如说是给你的应用的用户界面的一部分做动画设计的。例如，你可以使用 `NSViewAnimation` 类 (一个 `NSAnimation` 子类) 来给视图或 window 的大小、位置、透明度创建平滑的过度。外表的动画显示让你创建一个更流畅的外表。

这篇文档描述了使用 Cocoa 动画对象时所涉及到的一些基本概念，也提供了样例代码来
说明在你自己的应用中怎么使用它们。

## Organization of This Document
这篇文档包含以下文章：

* **Using an NSAnimation Object** 描述了动画对象的基本特点，和你怎么自定义它们。
* **Animating Views and Windows** 描述了视图动画对象的使用，它们提供了更高级别的 API 来给视图和 window 的大小，位置，透明度做平滑的动画。

## See Also
样例代码提供了使用 Cocoa 动画类的样例：

* *Reducer* 实现了一个可以重用的 collapsible 视图 (使用 `NSViewAnimation`) 和一个继承 `NSAnimation` 的动画显示的 tab view
* *iSpend* 使用 `NSViewAnimation` 实现了一个可扩伸的视图

----
# Using an NSAnimation Object
`NSAnimaiton` 类为发生在有限时长的动画提供了高级行为。OS X 使用动画对象为 UI 元素实现了过度动画。你可以定义自定义动画对象来为你的代码实现动画。跟 `NSTimer` 相似，动画通知可以以不规律的间隔发生，允许你来创建看上去时快或时慢的动画。

下面的部分覆盖了创建一个自定义 `NSAnimation` 对象的基本步骤，并使用它来管理你的动画内容。如果你想要给你的视图和 windows 做动画，你应该查看 `NSViewAnimation` 类 (是 `NSAnimation` 的子类) 是否提供了你需要的行为。视图动画对象为对这事件改变视图大小和移动视图提供了高级的行为，它们在本文的 *Animating Views and Windows* 部分有描述。
 
## Creating and Configuring an Animation Timer
一个 `NSAnimation` 对象有几个重要属性：

* current progress —— 一个介于 0.0 与 1.0 的值，表示了动画完成的百分比。
* frame rate —— 当前每秒更新次数
* duration —— 动画发生的时长
* Animation curve —— 动画在它的过程中相对速度；例如，动画可以在开始的时候慢慢加速，然后在接近结束的时候慢慢减速，或整个过程保持相同的速度。
* blocking mode——动画运行过程中所处的模式，以应用响应用户行为的程度定义的

当你配置一个新的 `NSAnimation` 对象，你至少必须配置它的 duration，animation curve，frame rate，和 blocking mode 属性。你也应该赋一个 delegate 来监听动画的过程。当动画开始，结束，显式停止，或到达了过程中的标记点，动画对象发送一个消息给当前的 delegate。(查看本文的 *Setting and Handling Progress Marks* 来了解更多关于 progress mark 的信息) 。如果不想使用一个 delegate，你必须继承 `NSAnimation` 来获得进度信息；更多信息可以查看本文的 *Subclassing NSAnimation*。

下面的代码展示了一个样例方法，这个方法创建并配置一个标准的 `NSAnimation` 对象。创建动画的对象作为动画的 delegate，并狐狸任何进程消息。

```objc
- (id)init {
    self = [super init];
    if (self) {
        // theAnim is an NSAnimation instance variable.        theAnim = [[NSAnimation alloc] initWithDuration:10.0                                     animationCurve:NSAnimationEaseIn];        [theAnim setFrameRate:20.0];        [theAnim setAnimationBlockingMode:NSAnimationNonblocking];        [theAnim setDelegate:self];
    }
    return self;
}
```

`initWithDuration:animationCurve:` 方法是 `NSAnimation` 类的 designated 初始化方法。这个方法让你设置动画的两个属性。对于其他的属性，你可以使用默认值或使用适合的 accessor 方法来设置它们。默认的属性如下：

* 默认动画 curve 是 `NSAnimationEaseInOut`
* 默认 blocking mode 是 `NSAnimationBlocking`
* 默认 frame rate 是一个合理的值。frame rate 通常是 60 Hz，但是这个准确的值并不可靠。

一旦你准备好了一个 `NSAnimation` 对象供使用，你可以通过发送一个 `startAnimation` 消息给它来运行它。如果你需要在动画在它规划的时间前结束的话，给这个对象发送一个 `stopAnimation` 消息。`NSAnimation` 对象的 delegate (如果存在的话) 会受到消息来通知它这些事件，并告诉它动画已经按照规划的方式结束。

## Setting and Handling Progress Marks
`NSAnimation` 有 *progress marks* 的概念，*progress marks* 的值类型是浮点型 (类型是 `NSAnimationProgress`)，表示了动画完成的百分比。当你启动一个动画并它达到一个 progress mark 时 (准确的说，当前的进度等于 progress mark)，动画对象给它的 delegate 发送消息。delegate 然后可以更新自定义的进程指示，播放一个声音，或者完成一些其他适合动画这个点的效果。

> **重要**: 尽管你可以使用 progress mark 来以时间切分一个对象的动画，但这不是一个理想的方式来完成一个平滑的动画。一个推荐方式是继承 `NSAnimation` 并重绘每帧的改变；查看本文的 *Smooth Animations* 来了解更多的信息。

通常你在开始创建和初始化一个动画对象的时候设置这个对象。下面的代码展示了一种设置 20 间隔进度空隔的 progress marks。

```objc
- (void)awakeFromNib {
         NSAnimationProgress progMarks[] = {            0.05, 0.1, 0.15, 0.2, 0.25, 0.3, 0.35, 0.4, 0.45, 0.5,            0.55, 0.6, 0.65, 0.7, 0.75, 0.8, 0.85, 0.9, 0.95, 1.0  };
    
    int i, count = 20;
    // theAnim is an NSAnimation instance variable
    theAnim = [[NSAnimation alloc] initWithDuration:10.0 animationCurve:NSAnimationEaseInOut];
    [theAnim setFrameRate:20.0];    [theAnim setDelegate:self];
    
    for (i=0; i<count; i++)        [theAnim addProgressMark:progMarks[i]];
```

除了像上面样通过一个循环来设置 progress-mark，你也可以通过调用 `setProgressMarks:` 方法来在一个调用中设置它们，这个方法接受一个封装浮点数的 `NSNumber` 对象的数组。

当一个运行过程中的动画到达一个 progress mark，它发送 `animation:didReachProgressMark:` 给它的 delegate。delegate 应该以一种合适的方式来处理传入的 progress mark。下面的代码展示了一个 delegate 实现，该实现通过间隔的播放火车声来响应。

```objc
- (void)animation:(NSAnimation *)animation didReachProgressMark:(NSAnimationProgress)progress {
    if (animation == theAnim) {
        [[NSSound soundNamed:@"chug"] play];
    }}```

## Subclassing NSAnimation
尽管你只是可以使用 `NSAnimation` 对象就可以完成很多任务，但继承它是更普遍的方案。有 3 个主要原因要继承 `NSAnimaiton`：

* 通过绘制每帧的方式来实现平滑的动画。
* 当在主线程上以非阻塞模式执行一个动画的时候指定有效的 run-loop 模式
* 返回一个自定义 curve 值，不需要指定一个 delegate 来响应 `animation:valueForProgress:`

完成上述任务中前两个的步骤在以下部分有描述。要返回一个自定义 curve 值而不想实现一个 delegate 方法，你必须覆盖 `currentValue` 方法。查看 `NSAnimation` 类文档来了解更多信息。

### Smooth Animations
像 **Setting and Handling Progress Marks** 中说的一样，你可以设置一系列 progress marks 到一个 `NSAnimation` 对象，并让 delegate 实现 `animation:didReachProgressMark:` 方法来在每个 progress mark 来绘制一个对象。然而，这并不是最好的方式来给一个对象做动画。除非你设置很多 progress marks (每秒 30 或更多)，动画可能看上去是跳动的。

一个更好的方案是继承 `NSAnimation` 并覆盖 `setCurrentProgress:` 方法，像下面的代码一样。`NSAnimation` 对象在改变进度值的每帧之后调用这个方法。通过截获这个消息，你可以进行任何重绘或根据需要更新这个帧。如果你确实覆盖这个方法，确定在这个方法中调用 `super` 的实现来更新当前的进度。

```objc
- (void)setCurrentProgress:(NSAnimationProgress)progress {
     // Call super to update the progress value.    [super setCurrentProgress:progress];   
    // Update the window position.    NSRect theWinFrame = [[NSApp mainWindow] frame];    NSRect theScreenFrame = [[NSScreen mainScreen] visibleFrame];    theWinFrame.origin.x = progress *            (theScreenFrame.size.width - theWinFrame.size.width);    [[NSApp mainWindow] setFrame:theWinFrame display:YES animate:YES];
```

### Custom Run-Loop Mode Sets
一个使用 `NSAnimationNonblocking` 阻塞模式的 `NSAnimation` 对象跑在一个 run-loop mode 接受用户输入的进程主线程上。在它开始跑动画前，动画对象给它自己发送一个 `runLoopModesForAnimation` 消息来获得当前有效的 run-loop modes。默认情况下，这个方法返回 `nil`，这会告诉 `NSAnimation` 使用默认的 mode (`NSDefaultRunLoopMode`)，panel mode (`NSModalPanelRunLoopMode`) 和 event-tracking run-loop mode (`NSEventTrackingRunLoopMode`)。

你可以覆盖这个方法返回一个不同的 run loop modes 集合，这可以包含自定义的 modes。下面的代码给出了一个返回除 event-tracking mode (`NSEventTrackingRunLoopMode`) 外的默认 modes 数组

```objc
- (NSArray *)runLoopModesForAnimating {
    return  @[NSDefaultRunLoopMode, NSModalPanelRunLoopMode];
}
```

## Linking Animations
你可以连接两个动画对象，以便它们中的一个动画可以在另一个到达指定的动画标记时开始运行。`NSAnimation` 的这个特性很适合协调不同的效果。下面的代码说明了 ` startWhenAnimation:reachesProgress:` 方法是怎么被用来在一个动画到达它的中间点时开始另一个动画的。

```objc
- (IBAction)startAnim:(id)sender {
    // theAnim and theOtherAnim are variables of type NSAnimation.    [theOtherAnim startWhenAnimation:theAnim reachesProgress:0.5];    [theAnim startAnimation];
}
```

如果相反你想要在一个动画到达一个 progress mark 的时候停止一个动画，请使用 `stopWhenAnimation:reachesProgess:` 方法。你可以无限的连接动画，一个接着一个。然而，在任一给定时刻，只能有一个 "start" 和一个 "stop"。

如果你有一个 delegate 可以响应 `animation:didReachProgressMark:`消息，它必须区分多个动画，如下例：

```objc
- (void)animation:(NSAnimation *)animation didReachProgressMark:(NSAnimationProgress)progress {
    if (animation == theOtherAnim) {
        // Do an effect appropriate to progress mark.
    } else if (animation == theAnim) {
        // Do an effect appropriate to progress mark.
    }
}
```

---
# Animating Views and Windows
`NSViewAnimation` 类是 `NSAnimation` 的子类，提供了方便的方法来给视图或 window 对象某些方面做动画，包括以下：

* 改变 frame 的位置
* 改变 frame 的大小
* 改变对象的透明度，fade it in or out

## The View Animation Process
使用 view animation 对象的方式和你通常使用 `NSAnimation` 对象的方式有点儿出入。单一的 view animation 对象可以同时管理多个视图或 window 的动画过程。与其使用动画对象的方法来设置动画的属性，还不如为你想要修改的 view 或 window 创建动画属性字典。每个字典指定 action 的目标，和你想要使用的效果。要设置其他的因素，如 duration 和动画的 timing curve 的话，你继续使用 `NSAnimation` 的方法。

一个动画属性字典只有一个必须的值：目标对象。你使用 `NSViewAnimationTargetKey` 添加这个对象到字典。只有这个 key 存在的时候，并不会改变 view 或 window。要做修改的话，你必须包含一个或多个 keys 来指定想要的行为。

如果你选择的话，你可以对一个目标对象同时进行多个 actions。例如，你可以重新指定视图大小，改变它在屏幕上的位置，并且 fade it in or out。为了简单，下面的部分会分开展示这些 actions。要同时进行它们，简单的添加相关的 keys 到属性字典。

### Changing the Frame Rectangle
修改一个视图或 window 的 frame rectangle 让你重新指定对象的大小和位置。以视图为例的话，这意味着改变视图在它的父视图中的位置和大小。以 window 来说的话，这意味着改变 window 在桌面上的大小和位置。

Key | Value | Description
------ | -------- | ----------------
`NSViewAnimationTargetKey` | id | 被 resize 或 reposition 的 `NSView` 或 `NSWindow` 实例，必须
`NSViewAnimationStartFrameKey` | NSValue | 包含目标对象开始的 frame rectange。`NSValue` 兑现更应该包含一个编码了的 `NSRect` 数据类型。这个值通常等于视图或 window 当前的 frame。这个 key 是可选的并且默认是当前的 frame rectangle
`NSViewAnimationEndFrameKey` | `NSValue` | 包含目标对象结束时的 frame rectangle 。`NSValue` 对象应该包含一个编码了的 `NSRect` 数据类型。这个 key 是推荐的；如果没有出现的话，默认是目标对象当前的 frame rectangle

### Fading Objects In and Out
如果你想要隐藏一个视图或 window 的话，与其让这个对象突然消失，你不如使用一个视图动画来让这个对象渐渐地消失。相似的，你可以使用一个类似的动画来这个对象渐渐的出现。当渐渐显示一个视图的时候，视图的 ending frame rectangle 的大小必须不能为 0；如果是 0 的话，视图会保持隐藏。下表列出了渐现或渐隐应该使用的 key 和 value

Key | Value | Description
------ | ---------- | --------------
`NSViewAnimationTargetKey` | id | 标识需要修改的 `NSView` 或 `NSWindow` 对象。这个 key 是必须的
`NSViewAnimationEffectKey` | NSString | 包含下面的 string 常量：<br /> `NSViewAnimationFadeInEffect` 使得对象可见<br /> `NSViewAnimationFadeOutEffect` 影藏它。这些效果在动画的过程中改变对象的透明度

## A View Animation Example
下面的代码展示了一个 view animation 对象的基本使用。action 方法为两个不同的视图对象设置了属性字典，然后当 action 发生的时候跑这个动画。对于第一个视图对象，动画对象沿着每个轴移动原点 50 单位。对于第二个视图，动画对象缩小 frame 大小到 0 同时渐渐隐藏它知道它消失。动画使用一个自定义 timing curve 和时长但是使用默认的 blocking mode，这样默认会阻塞主线的用户输入直到动画结束。

```objc
- (IBAction)startAnimations:(id)sender {
     // firstView, secondView are outlets    NSViewAnimation *theAnim;    NSRect firstViewFrame;    NSRect newViewFrame;    NSMutableDictionary* firstViewDict;    NSMutableDictionary* secondViewDict;
    
    {          // Create the attributes dictionary for the first view.          firstViewDict = [NSMutableDictionary dictionaryWithCapacity:3];          firstViewFrame = [firstView frame];          // Specify which view to modify.          [firstViewDict setObject:firstView forKey:NSViewAnimationTargetKey];          // Specify the starting position of the view.          [firstViewDict setObject:[NSValue valueWithRect:firstViewFrame]                   forKey:NSViewAnimationStartFrameKey];          // Change the ending position of the view.          newViewFrame = firstViewFrame;          newViewFrame.origin.x += 50;          newViewFrame.origin.y += 50;          [firstViewDict setObject:[NSValue valueWithRect:newViewFrame]
              forKey:NSViewAnimationEndFrameKey];
    }
    
    {        // Create the attributes dictionary for the second view.        secondViewDict = [NSMutableDictionary dictionaryWithCapacity:3];    
        // Set the target object to the second view.        [secondViewDict setObject:secondView forKey:NSViewAnimationTargetKey];        // Shrink the view from its current size to nothing.        NSRect viewZeroSize = [secondView frame];        viewZeroSize.size.width = 0;        viewZeroSize.size.height = 0;        [secondViewDict setObject:[NSValue valueWithRect:viewZeroSize]                 forKey:NSViewAnimationEndFrameKey];    
        // Set this view to fade out        [secondViewDict setObject:NSViewAnimationFadeOutEffect                forKey:NSViewAnimationEffectKey];    }
    
    // Create the view animation object.    theAnim = [[NSViewAnimation alloc] initWithViewAnimations:[NSArray            arrayWithObjects:firstViewDict, secondViewDict, nil]];
    
    // Set some additional attributes for the animation.    [theAnim setDuration:1.5];    // One and a half seconds.    [theAnim setAnimationCurve:NSAnimationEaseIn];

    // Run the animation.    [theAnim startAnimation];    
    // The animation has finished, so go ahead and release it.    [theAnim release];
```