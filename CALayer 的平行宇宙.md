## CALayer 的平行宇宙

本文来自 [Joel Kraut](http://blog.spacemanlabs.com/author/ultrajoke/) 的 [CALayer’s Parallel Universe](http://blog.spacemanlabs.com/2011/08/calayers-parallel-universe/)

试过做 `UIView` 位置的动画没？很简单，使用 `UIView` 动画的类方法`animateWithDuration:animations:` 就可以做到。简单的在 animation block 中改变 view 的位置就可以了。

但是你试过在动画的过程中改变动画了？假设你使用动画在写一个 `UIView` 总是移到手指的应用，第一个动画正常工作，但时候剩下的 animation blocks 会导致动画始终是从上个动画结束位置，而不是从动画中的某个未知的位置。

### 为什么会这样了？
先简单说下，`UIView` 的动画方法中一个方法支持动画的一些选项，其中一个是 `UIViewAnimationOptionBeginFromCurrentState` —— 从名称上看，动画总是从 view 的当前状态开始，即 view 在屏幕上的位置开始。但是上述问题还是没有得到回答，为什么会这样了？背后到底发生了什么？

### 揭开幕布
你也许在某处听过 `UIView` are backed by `CALayers`。这句话的意思是 CALayer 是真正处理 render、compositing、animation 的幕后主角。而 `UIView` 只是一个相对复杂的 add-on，它知道怎么处理用户交互，打印，和它的子类所做的事。UIView 很多处理 view hierachy，display，colors，hit testing 的方法只是简单的调用 CALayer 的相似方法。

现在，你知道 CALayer 才是幕后英雄。我们来聊聊它是怎么完成它的工作。当一个 View 从 A 动画到 B 时到底发生了什么？

### 平行宇宙
一切并不神奇，只是技术而已。当你启动一个动画的时候，两件很不同的事会发生。 view 的 animated 属性根本没啥动画，它被立即设置为新的值。当你设置一个一年的动画，在你的下一代码中获取 view 的位置，你将得到一年后的 view 的位置。但是这实在是不合理——屏幕上的 view 显然不在一年后的位置上。

如果 view 的位置属性改变是即刻发生的，而 view 在屏幕上又是从一个位置移动到另一个位置，那么一定有些我们不知道的。秘密是：我们代码所交互的 view 并不是屏幕上的我们看到的 view。你引用的任何 view 都不曾在屏幕上，Core Animation 创建了一个平行的 view hierarchy。屏幕上你看到的只是一个双胞胎而已。

### 原理
Core Animation 所做的是一个底层的 Mode / View 的分离，就像我们熟悉的 MVC 模式一样。等等，难道我们正在说的不就是 view 吗？we're overloading the term here (不知道咋翻译). 现在我们讨论的关于一个对象的 model data 和 model data 的 view，而 model data 刚好是 UIView (晕了的有木有啊~_~#)。**model**正是你交流的 UIView，它包含了数据，如位置。 **view** 正是屏幕上平行的 CALayer，它是 data 的视觉展现。它可以做动画，view 可以按它自己的方式渲染数据。

上述原理知道就行，如果你无法访问平行的 view hierarchy 的话那就只是满足求知的兴趣。幸运的是你可以！CALayer 的 `presentationLayer` 方法可以带你去那儿。讲课了：一个 layer 的 “presentation layer” 正是上文中讲的 **view**。要在两个 hierarchy 中来回的话，可以使用 presentation layer 的 model layer，通过 CALayer 的 `modelLayer` 方法获取。

### Code
结论就是：presentation layer 反应正在屏幕上的视图，而不是我们习惯的 model layer。这样从当前位置做动画就简单很多了。代码示例如下：

```objc
- (void)viewDidLoad
{
    [super viewDidLoad];
    touchView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 40, 40)];
    touchView.backgroundColor = [UIColor redColor];
    [self.view addSubview:touchView];
    UITapGestureRecognizer *gr = [[UITapGestureRecognizer alloc] 
        initWithTarget:self action:@selector(tap:)];
    [self.view addGestureRecognizer:gr];
}

- (void)tap:(UITapGestureRecognizer*)gr
{
    CGPoint newPos = [gr locationInView:self.view];
    CGPoint oldPos = [touchView.layer.presentationLayer position];
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position"];
    animation.fromValue = [NSValue valueWithCGPoint:oldPos];
    animation.toValue = [NSValue valueWithCGPoint:newPos];
    touchView.layer.position = newPos;
    [touchView.layer addAnimation:animation forKey:@""];
}
```