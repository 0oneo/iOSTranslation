本文来自 [Mike Ash](https://www.mikeash.com/) 的 [NSObject: the Class and the Protocol](https://mikeash.com/pyblog/friday-qa-2013-10-25-nsobject-the-class-and-the-protocol.html) ，主要是关于 Objective C 为什么会有叫 `NSObject` 的 class 和 protocol.

----


Reader Tomas Bouda 问：`NSObject` protocol 什么鬼？Cocoa 中有两个 NSObjects，一个 class 和一个 protocol. 为什么是两个了？ 它们存在的意义是什么了？

### 名称域（Namespaces）
首先，让我们看这两个实体是怎么能够同时存在的。Objetive-C 中的 classes 和 protocols 存在于完全不同的 namespaces。你可以拥有相同的名字的 class 和 protocol，这正是 NSObject 的情况。

如果你看看 Objective-C 这语言，你会发现没有地方你可以互换的使用 class 和 protocol。class 的名字可以使用作消息被发送的对象，在 `@interface` 的申明中，作为 type 名。 protocols 可以在一些同样的地方使用，但始终以不同的方式出现。在同样的名字的地方没有是 class 还是 protocol 的疑问。

### Root Classes
`NSObject` 是一个 root class. root class 没有 superclass。在 Objective-C 中，可以有多个 root class。

Cocoa 有多个 root class，除了 `NSObject`外，还有 `NSProxy` 和 一些其他的 class。这也是 `NSOjbect` protocol 存在的部分原因。`NSObject` 协议定义了 root class 必须实现的方法。

`NSObject` 和 `NSProxy` 都实现了 `NSObject`  protocol

### 关于 Proxies 的另一面 （An Aside About Proxies）
既然我们谈到了，那为什么有一个 `NSProxy` root class 了?

在某些情况下需要某些没有实现太多的 methods 的 class。正如 NSProxy 名字所暗示的，proxy 对象在这些情况下很适合。`NSObject` class 实现了很多不是 `NSObject` protocol 定义的方法，如 key-value coding。

当构建一个 proxy object 时，目标往往是不实现大部分 methods，这样就可以在 `forwardInvocation:`
 中转发这些没有实现的调用。Subclassing `NSObject` class 会引入很多可能影响的包袱。`NSProxy` 通过提供过一个更简单的 superclass 来避免的了这些。
 
### protocols
`NSObject` protocol 对于 root class 很有用的事实，在大部分的 Objective-C 编程中并不有趣，因为我们很少使用其他的 root classes。但是当我们写我们自己的 protocols 时，就变得很方面了。

```objc
@protocol MyProtocol

- (void)foo;

@end
```

现在你有一个指向实现了这个协议的对象：

```ojbc
 id<MyProtocol> obj;
```

你可以告诉这个对象 `foo`：

```objc
[obj foo];
```

然后，你不能进行如下调用：

```objc
[obj description];
```

同样你也不能 check for equality：

```objc
[obj isEqual: obj2];
```

通常，你不能要求这个对象做任何一个普通对象可以做的。有些时候，这没啥，但是又有些时候你真想要这个对象做个普通的对象。

这就是 `NSOjbect` protocol 出现的地时候了。Protocols 能够继承其它的 protocols。你可以让 `MyProtocol` 继承 `NSObject` 协议：

```objc
@protocol MyProtocol <NSObject>

- (void)foo;

@end
```

这就是说 conform to `MyProtocol` 的对象不仅 responde to `foo`，而且 respond to 任何定义在 `NSObject` protocol 中的 message 。由于每个 object 往往继承自 NSObject class 且 NSObject class comforms to NSObject，每个 object 就不用实现其他的要求了。

###Conclusion
有不同的 `NSOjbect` ，确实是很奇葩的地方，但是当你深入分析的话，会发现还是挺合理的。一个 `NSObject` 允许了多个 root class 存在，同时也是声明一个新 protocol 时要求一个对象能够像一个普通的的 NSObject 对象一样。`NSObject` class 实现了 `NSObject` protocol 又将一切串起来了。