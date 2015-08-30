本文来自 [Mike Ash](https://mikeash.com/) 的 [Friday Q&A 2013-06-14: Reachability](https://mikeash.com/pyblog/friday-qa-2013-06-14-reachability.html)。

应用开发中网络所扮演的角色前所未有的重要，Apple 的 reachability API 是一个很有价值的工具，能够使重网络的应用很好的响应真实环境中不断变化的网络。今天我将来介绍 reachability API 的概况，它是做什么的，怎样使用。主题来自读者 Manuel Diaz 的建议。

### Concept
Reachability 允许应用检查一个 remote server 是否有被访问的潜在可能。简单说，它告诉你电脑 (或 iDevice) 有一个网络连接。它也追踪不同网络的联通状态，当网络消失或出现的时候通知你。

想象你在没有信号的隧道中打开了一只 app。它试着加载数据，但是失败了，它给了你一个奇怪的提示。你穿过了隧道，但是你的 app 仍然显示着那个错误。你点击了 reload 的按钮，以为是信号丢失的原因导致数据加载失败。

现在假设同样的场景，但是现在的 app 使用了 reachability API。它通知没有网络连接，并第一时间就不试着去加载数据，而是提示需要网络连接的信息。一旦你离开隧道手机获得了信号，app 自动加载数据，你可以继续使用 app。

显然第二个方案好很多，它也不需要太多的工作量。

### Limitations
Network reachability 并不是万灵药，并不能替换你网络代码中的错误处理。

比较特别的是，reachability API 只考虑本地的环境。如果你的电脑连上了一个 Wi-Fi access point，但是这个 access point 连不上互联网，reachability 会返回 *yes*，表示你有一个 network connection。从它的角度来看，你确实有。同样，如果你有一个可访问互联网的连接，但是目标 server down 了， reachability 同样返回 *yes*，你有一个连接。

最后，任何先检查许可条件然后执行一个 request 内在本身一个 race condition。Reachability 可能会说网络连接是 up 的，但是在你的代码执行网络请求之前，它断开了。

上面所说的是你的代码应该在没有 reachability 时的能够正常的工作，但是采用 reachability 来增加可用性。

### The API
Reachability API 在 SystemConfiguration 框架中，叫做 `SCNetworkReachability`。它跟 Core Foundation 的 API 相似，你需要处理下 C。

使用这个 API 基本的 lifecycle 如下：

1. 使用一个给定的 remote host 创建一个 reachability 对象
2. 设置回掉，以便当有改变时它能够通知你
3. 在 runloop 或 dispatch queue 上调度 reachability 对象
4. 如果你想的话，直接查询 reachability 对象
5. 完成后，释放掉对象

#### Creation
使用 `SCNetworkReachabilityCreateWithName` 创建 reachability 对象。

```objc
NSURL *url = ...;
NSString *host = [url host];
SCNetworkReachabilityRef reachabilityRef = SCNetworkReachabilityCreateWithName(NULL, [host UF8String]);
```

#### Callback
通过 `SCNetworkReachabilitySetCallback` 来设置回掉。参数是一个 context 和一个 C 函数。因为在 Objective-C 中的代码中原生的 C 函数比较烦人。我将写一个调用 block 的跳板函数。

C callback 函数需要三个参数：reachability 对象，new reachability flags，an info pointer。Reachability 对象本身就可以被访问，info pointer 中有 block，block example：

```objc
void (^callbackBlock)(SCNetworkReachabilityFlags) = ^(SCNetworkReachabilityFlags flags) {
  BOOL reachable = (flags & kSCNetworkReachabilityFlagsReachable) != 0;
  NSLog(@"The network is currently %@", reachable ? @"reachable" : @"unreachable");
};
```

接下来准备好 context structure，它有五个字段，`version` 总是为 0，info 指针，`retain` 和 `release` 调用，一个 `copyDescription` 调用。我们只使用 `version`，`info` 和 `release`。info 指针总指向上面的 block。`CFBridgingRetain` 调用是为了使 ARC be happy with the type conversion， and to handle memory management correctly:

```objc
SCNetworkReachabilityContext context = {
  .version = 0,
  .info = (void *)CFBridgingRetain(callbackBlock),
  .release = CFRelease
};
```

真实的回掉函数只是从 `info` 中取出 block 并调用它：

```objc
static void ReachabilityCallback(SCNetworkReachabilityRef reachabilityRef, SCNetworkReachabilityFlags flags, void *info) {
  void (^callbackBlock)(SCNEtworkReachabilityFlags) = (__bridge id)info;
  callbackBlock(flags);
}
```

一切准备好之后，设置回掉就可以了：

```objc
SCNetworkReachabilitySetCallback(reachabilityRef, ReachabilityCallback, &context);
```

#### Scheduling
为了提供通知，reachability 对象必须在 runloop 或 dispatch queue 上调度。回调函数将会在指定的 runloop 或 queue 上被调用。

为了在主线程上收到通知并调用回调函数，只需在主线程的 runloop 上调度 reachability 对象：

```objc
SCNetworkReachabilityScheduleWithRunLoop(reachabilityRef, CFRunLoopGetMain(), kCFRunLoopDefaultMode);
```

或者在 main dispatch queue 上调度：

```objc
SCNetworkReachabilitySetDispatchQueue(reachabilityRef, dispatch_queue_get_main());
```

如果你不想回调在主线程上被调用的话，你可以传一个自定义的 dispatch queue 或者 global queue。

#### Flags
当你的回掉别调用后，它将收到 new reachability flags。你可以直接 query the flags：

```objc
SCNetworkReachabilityFlags flags = 0;
bool success = SCNetworkReachabilityGetFlags(reachabilityRef, &flags);
if(!success) // error getting flags!
```

可能的 flags 有一些，你该看啥了？

最重要的一个是 `kSCNetworkReachabilityFlagsReachable`。这个 flag is set when the destination appears to be reachable。

flags 提供了 reachability 描述的很多信息，如目前还没有连接，但会根据需要初始化。具体请查看文档。

对于 iOS 开发很相关的一个 flag 是 `kSCNetworkReachabilityFlagsIsWWAN`。这个 flag 是在手机通过 cellular connection 时设置的。你可以使用这个 flag 来更全面的决策，如下载什么样的数据，调节这些连接的慢和贵的特性。

**注意**：如果你有流量绝对不能从 cellular connection 走的话，检查 `kSCNetworkReachabilityFlagsIsWWAN` 并不是 100% 可靠的。系统可能认为可以通过其他的途径，但是稍后丢失了那个连接，而尝试使用 cellular network。如果你需要绝对不使用 cellular network，你可以使用 `NSMutableURLRequest` 的 `setAllowsCellularAccess:` 方法。

#### Cleanup
当不需要再使用 reachability 对象时，first unschedule it from the runloop, or clear the dispatch queue:

```objc
// runloop
SCNetworkReachabilityUnscheduleFromRunLoop(reachabilityRef, CFRunLoopGetMain(), kCFRunLoopDefaultMode);

// dispatch queue
SCNetworkReachabilitySetDispatchQueue(reachabilityRef, NULL);
```

最后，释放对象：

```objc
CFRelease(reachabilityRef);
```

### Conclusion
Reachability API 是一个有价值的工具，使得网络类型的 app 更简单和更好用。在发送请求前，通过检查 reachability status，你可以给用户提供更好的错误报告。通过观察 reachability 变更，一旦用户重新获得网络访问，你可以自动的重试网络请求。
