本文来自 [Mike Ash](https://www.mikeash.com/) 的 [Objective-C Message Forwarding](https://www.mikeash.com/pyblog/friday-qa-2009-03-27-objective-c-message-forwarding.html)，主要介绍的是 Objective-C 的消息转发机制。

----

Yuji Tachikawa 建议说说 `@dynamic` properties 的工作机制，接下来我将谈谈这个，并将话题扩展到更普遍的消息转发机制。

### No Such Method
 
上周我介绍了下 Objective-C 消息发送机制（可查阅[[翻译: Objective C 消息发送]]），提到了当 selector 对应的 method 没有找到的时候会发生一些有趣的事情，这些事情导致消息转发。
 
什么是消息转发了？简单来说，它允许捕捉 (trapped) 未知的 (unknown) method，并作出响应。换句话说，任何时候一个未知的 method 被发送了，它被打包发送给你的代码，你想怎么处理就怎么处理。
 
消息转发特别强大，可以用来做实用，灵活的事情。

**Note**: 现在，你可能在想，“为啥要叫转发了 (forwarding) ?"，看上去对未知的 messages 采取任意的行为跟 “转发” 没啥大关系。叫这的主要原因是这个技术主要是为了允许对象让其他的对象替他们处理那些 message，因此叫做 “转发”。


 
### 发生了什么 (What Happens)
 
当你发送消息 `[foo bar]`，但 `foo` 没有实现 `bar` method 时会发什么了？ 当它实现了这个方法时，很简单：它查找合适的 method，然后调用。当没有找到 method 时，一系列复杂的事情接踵发生：
 
 1. **Lazy method resolution**. 这是通过发送 `resolveInstanceMethod:` (`resolveClassMethod:` 对于类方法的话) 给被询问的类。如果 method 调用返回 `YES`，runtime 会假设 unknown method 现在已经被添加到 class 的 method list 上了，unknown method 的发送过程也将重新启动 (restarted)
 2. **Fast forwarding path**. 这是通过发送 `forwardingTargetForSelector:` 给 target object 完成的，当然前提是它实现了这个 method。如果这个 method 返回的不是 `nil` 或 `self`，那么整个下消息发送的过程将以返回的值作为新的 target object 重新启动。
 3. **Normal forwarding path**.  首先 runtime 会发送 `methodSignatureForSelector:` 给 target object，以获取参数和返回值类型 。如果 method 的签名有返回，runtime 会创建一个 `NSInvocation` 对象来描述发送的消息，然后发送 `forwardInvocation:` 到这个对象。如果没有返回 method 的签名，runtime 发送 `doesNotRecognizeSelector:`

### Lazy Resolution
正如上周我们学到的一样，runtime 是通过查找 method (or IMP) 并调用来完成消息发送的。有时动态插入 IMP 比提前设置好它们要有用。这么做允许真正的快速 “转发”，因为在 method 被解析之后被调用正如正常的消息发送过程一样。缺点是不够灵活，你得提前有个 IMP 被插入 (plug in)，这意味着你得有预先定义好的参数和返回类型。

这类动态插入 IMP 的做法倒是挺适合 `@dynamic` properties。你得提前知道 method signature: 要么是一个参数无返回，要么不接受参数返回一个。properties value 的类型会变化，但是你可以覆盖通用的情况。因为 IMP 已经有发送 target object 的 selector 参数，它可以通过 selector 获得这个 property 的名字，并动态的查找。通过 `+ resolveInstanceMethod:` 插入到相应的 class 就可以了。

### Fast Forwarding
runtime 检查的下一件事是你是否想要把整个消息原封不动的发送到不同的 object。由于这是转发的普遍情况，runtime 允许我们轻易的完成这。

不知道为何，fast forwarding 基本没啥文档。除了 `NSObject.h` 中一段注释外，Apple 只在一个地方有所提及 (请自行搜索 New forwarding fast path， 原文中的链接已年久失修)。

这项技术很适合制造多层继承的假象，你可以按照以下代码重写：

```objc
   - (id)forwardingTargetForSelector:(SEL)sel { return _otherObject; }
```

这将导致任何 unknown method 被转发到 `_otherObject`， 这又导致从外面看你的 object 像融合了你的 object 和 otherObject 的行为。

### Normal Forwarding
前面的两个主要是允许转发更快的优化措施。如果你不想利用它们，那么更完整的转发机制就上场了。runtime 创建一个 `NSInvocation` 对象，这个对象封装了整个消息，包括 target object，selector ，所有的参数，它同样允许控制返回的类型。

在 runtime 构建 `NSInvocation` 前它需要一个 `NSMethodSignature`，于是它通过`- methodSignatureForSelector: `获取了一个。这些都是因为 Objective-C 的 C 语言特点要求的。为了将参数打包到 `NSInvocation`，runtime 需要知道参数的个数和类型。但是这些信息 C 语言运行环境没有提供，因此它必须以 C 语言”字节包“的的视角来进行最后的运行，并且以其他的形式获得类型信息。（so it has to do an end run around the C "bag of bytes" view of the world and get the information in another way）. (这里已经突破了我知道的领域，烦请知道 Mike Ash在说啥的同学提个issue注解下)。

一旦 `NSInvocation` object 创建，runtime 就调用 `forwardInvocation:` method，在这个 method 里你可以干任何你想的。

下面是一个简单的列子，想象你厌倦了写循环，于是你想更直观的操作数组，给 `NSArray` 添加了如下 category：

```objc
    @implementation NSArray (ForwardingIteration)
    
    - (NSMethodSignature *)methodSignatureForSelector:(SEL)sel
    {
        NSMethodSignature *sig = [super methodSignatureForSelector:sel];
        if(!sig)
        {
            for(id obj in self)
                if((sig = [obj methodSignatureForSelector:sel]))
                    break;
        }
        return sig;
    }
    
    - (void)forwardInvocation:(NSInvocation *)inv
    {
        for(id obj in self)
            [inv invokeWithTarget:obj];
    }
    
    @end
```

然后你写下如下代码：

```objc
  [(NSWindow *)windowsArray setHidesOnDeactivate:YES];
```

我不推荐写这种代码。问题在于转发不会任何 `NSArray` 已经实现的方法，最后你会 capture some but not others（有点儿喜欢这种表达了，so，就原文了）。一个更好的方式是通过继承 `NSProxy` 写一个"跳板"类。

`NSProxy` 基本是一个显示设计为作代理的类，它实现了最小集合的 methods，其他的让开发者自行决断。这就以为着一个实现转发的子类可以 capture 任何消息了。

使用 `NSProxy` 来完成上述代码，你需要写一个 `NSProxy` 的子类，这个子类初始化后会指向一个 `NSArray` 实例，然后添加一个 stub method 到 `NSArray` 返回一个 `NSProxy` 子类的实例，如下:

```objc
   @implementation NSArray (ForwardingIteration)
    - (id)do { return [MyArrayProxy proxyWithArray:self]; }
    @end
```

然后你使用如下调用：

```objc
    [[windowsArray do] setHidesOnDeactivate:YES];
```

关于怎么写“跳板”(trampolines)类来 capture 消息，然后让它们做些比较有意思的事情，已经在 “Higher-Order Messaging” (原文中链接已失联，大家自行 google) 中被探讨。

### Declarations
另一个 Objective-C 的 从 C 语言来的特征，就是编译器需要知道方法的签名是什么，即便是那些你只想转发的。为了更直观的说明，想象你写了个类使用转发来生产整数，你如下写：

```objc
    int x = [converter convert_42];
```

显然上述代码并不实用，但你确实可以那样做。此项技术更有用的变种是可以有的。

问题在于编译器对 `convert_42` 方法一无所知，因此它就不知道该它返回什么，于是它将产生一个令人讨厌的警告，同时假设返回 `id` 类型。要修正也很容易，在某处声明下就行：

```objc
   @interface NSObject (Conversion)
    - (int)convert_42;
    - (int)convert_29;
    @end
```

再次声明，这里确实没啥用。当你遇到更实用的转发方案时，这可以帮你跟编译器和平相处。例如如果你用转发来伪造多继承，
使用一个类别（Category）来描述其他类的所有 method，当作应用了多个继承类。这样编译器知道它有两个系列的方法。其中一个系列是通过转发来设置及获取，这对编译器来说没什么。

### Conclusion
消息转发是一项牛逼的技术，使得 Objective-C 的表达能力增强不少。Cocoa 将其用于如 `NSUndoManager` 和 distributed objects上，同时它可以让你在你的代码中作很多实用的事情。