# 介绍

并发是多个事情同时发生的概念。随着 CPU 核的增加，程序员需要新的方式去利用它们。尽管像 OS X 和 iOS 的操作系统允许并行的跑多个程序，但是大部分程序跑在后台，做着需要很少持续 CPU 时间的 task。正是前台应用获取了用户的注意和保持电脑处于忙的状态。如果应用有很多 task 要做，但只让部分核处于忙的状态，那么其他核的资源就浪费了。

以前应用引入多线程需要创建一个或多个线程。不幸的是写多线程代码很具挑战。线程是必须手动管理的底层技术。考虑到系统不同的负载和底层硬件，应用的最优线程数会动态变化，实现一个正确的线程方案变得异常困难。另外，线程同步机制通常带来软件的复杂性，并冒有使软件架构不能保证性能提升。

OS X 和 iOS 都采用了更异步执行并发 task 的尝试，相对于传统的基于线程的系统和应用。应用只需要定义 task，然后让系统来完成它们，而不是直接创建线程。通过让系统来管理线程，应用获得通过原始线程方案无法获得的高扩展性。应用开发程序员同时页获得了一个更简单且高效的编程模型。

## 专用名词
在进入讨论并发的话题之前，有必要定一些相关的专有名词，以防止误解。对 UNIX 或 老点儿的 OS X 技术比较熟的开发者可能发现  "task", “process”，“thread” 这些词在这篇文档中有所不同。

* The term *thread* is used to refer to a separate path of execution for code. The underlying implementation for threads in OS X is based on the POSIX threads API.
* The term *process* is used to refer to a running executable, which can encompass multiple threads.
* The term *task* is used to refer to the abstract concept of work that needs to be performed.

---
# Concurrency and Application Design
计算机的早期时间，单位时间内它可以做的工作是由 CPU 的 clock speed 决定的。很多因素限制单核的最大速度，最后通过增加核心的数量来提供速度。随之而来的问题是怎么利用这些核。

为了利用多核，计算机需要软件能够同时做多件事。对于像 OS X 或 iOS 这样的现代操作系统，同时能有上百个程序在跑，在不同的核调度是可能的。但是了，大部分程序是 system daemons 或 background applications，这些程序消耗很少的资源。然而对于每个 application 真正需要的一种方法有效的使用多余的核。

传统的使用多核的方式是创建多个线程。然而随着核数目的提高，线程方案有其自身的问题。最大的问题是 threaded code does not scale very well to arbitrary numbers of cores (我的理解两者不成比例)。你不能创建与核心等量的线程，然后期望程序跑得很好。应用自身去计算使用多少核心是很高效的本身是一件很有挑战的事。即使知道了数目，给这么多线程编写代码也是很有挑战的。

总的来说，应用程序需要一种方式来利用可变的核心。一个应用的进行的工作也需要根据变化的系统情况来自动伸缩。方便必须足够简单，不增加利用这些核心做工作的总量。Apple 的操作系统了这样的解决方案，这章将会讲讲构成该方案的技术，以及一些你可以使用的 design tweaks。

## The Move Away from Threads
尽管线程已经存在多年，也还有人在用，但是它们没有解决可伸缩执行多个任务的普遍问题。使用线程的话，实现可伸缩方案的负担落在了开发者自生身上。你必须决定使用多少个线程，并根据系统条件的变化动态调节。另一个问题是你的应用承担着创建和维护线程的大部分成本。

OS X 和 iOS 采用了 *asynchronous design approach* 来解决并发的问题，而不是依赖于线程。异步函数在操作系统中已经存在多年，并被使用来启动需要长时间的任务，如从硬盘中读取数据。当被调用时，一个异步函数在幕后会做些工作来启动一个任务，并在任务真正启动前返回。往往，这些工作设计到获得一个后台线程，在这个线程上执行上述任务，当任务完成的时候发送一个通知给调用者(通常通过回掉函数)。在过去，如果某个你想用的异步函数不存在的话，你就需要编写你自己的异步函数和创建你自己的线程。但是现在，OS X 和 iOS 提供了允许你执行异步任务，但不需要你管理任何线程的技术。(用不用，你丫自己看着办吧，from @0oneo)

其中的一个启动异步任务的技术叫做 *Grand Central Dispatch (GCD)*。这项技术将你经常在自己应用中写的管理线程的代码提出来，移到系统的层级里。你所需要做的是定义你的任务，将这些任务添加到相应的分发队列中 (dispatch queue)。GCD 负责创建需要的线程，并在这些线程上规划这些任务。因为线程管理现在是系统的一部分，GCD 提供了管理、执行、提供比传统的线程更好性能的完整方案。

*Operation queues* 是行为跟分发队列 (dispatch queues) 非常像的 Objective-C 对象。你定义自己想要执行的任务，并把它们添加到 operation queue 中，operation queue 会替你负责线程管理，保证任务在系统上执行得尽量快与高效。

### Dispatch Queues
分发队列是基于 C 的一个执行自定义任务的机制。一个 *dispatch queue* 要么串行 (serially) 要么并行 (concurrently) 地执行任务，但始终是 first-in，first-out 的顺序 (换句话说，一个分发队列总是按照进入队列的顺序从队列中取出执行任务)。串行分发队列任一时刻总是只执行一个任务，在取出并执行一个新任务之前总是等待。相比之下，一个并发分发对象总是尽可能多的执行任务，并不等待已经启动的任务结束。

分发队列有些其他好处：

* 它们提供了直接并简单的编程接口
* 它们提供了自动全面的线程池管理
* 它们提供了性能优化
* 内存使用更高效 (because thread stacks do not linger in application memory)
* They do not trap to the kernel under load.
* 异步的分发一个任务给一个分发队列不会导致死锁
* 在资源 contention 的时候可以自由伸缩
* 串行的分发队列提供了一个比 locks 和 其他同步原语更高效的方式

你提交给分发队列的任务必须封装在一个函数或 block 对象中。 `block objects` 是 OS X v10.6 和 iOS 4.0 引入的一个跟函数指针概念相似的 C 语言特性，但相对于函数指针，它有其他优点。除了在 blocks 自身的词法域定义 blocks 外，你通常可以在另一个函数或方法中定义 blocks，这样 blocks 就可以访问函数或方法内的变量了。当把 blocks 提交到分发队列时，blocks 同样可以从原有的 scope 中移出，并拷贝到 heap 中。所有这些 semantics 使得使用较少代码实现非常动态的任务变得可能。

### Dispatch Sources
Dispatch source 是一种基于 C 的异步处理特定系统事件的机制。一个 dispatch souce 封装了一个特定系统事件类型的信息，并提交一个特定的 block 对象或函数给一个分发队列当事件发生的时候。你可以使用 dispatch sources 来监听以下系统事件：

* Timers
* Signal handlers
* Descriptor-related events
* Process-related events
* Mach port events
* Custom events that you trigger

### Operation Queues
一个 operation queue 是一个并发分发队列的 Cocoa 等同物，由 `NSOperationQueue` 类实现。尽管分发队列总是以 first-in ，first-out 的顺序执行任务，operation queues 在决定任务的执行顺序的时候会考虑其他的因素。你可以配置任务简的依赖性在定义你的任务的时候，通过使用依赖性可以为任务创建比较复杂的执行顺序图。

你提交给 operation queue 的任务必须是 `NSOperation` 类的实例。一个  *operation object* 是一个你需要执行的任务和任务所需数据 Objective-C 封装的对象。因为 `NSOperation` 类本质上是一个抽象基类，你通常需要自定义子类来执行你的任务。然而，Foundation Framework 包含一些具体的子类你可以可以来执行你的任务。

Operation 对象会产生 KVO 通知，你可以使用此来监听你的任务的进度。尽管 operation queue 总是并发的执行 operations，你也可以通过定义 operations 间的依赖性来保证任务串行的执行。

## Asynchronous Design Techniques
在你考虑重新设计你的代码来支持并发的时候，你应该问下你自己这样做是否值得。并发可以通过让你的 main thread 专门响应用户事件来保证你的应用的响应性；可以使你的代码给定时间内做更多的工作，通过使用多个核心。然而，并发也会增加负载，提供代码的整体复杂性，使得代码难写和调试。

除了增加复杂性外，并发并不是一个你在应用的产品周期最后可以移接的特性。正确的使用它需要仔细的考虑你的应用所做的任务和这些任务需要的数据结构。做的不对的时候，反而会降低你代码的效率和响应性。因此，在设计开始的时候很有必要花些时间来设定你的目标，想一个你需要执行的方案。

每一个应用有不同的要求和不同的任务需要执行。几乎不可能有一个文档来告诉你怎么设计你的应用和相关的任务。不过，下面的部分会提出一些引导来帮助做出好的决策，在设计的过程中。

### Define Your Application’s Expected Behavior
在你开始考虑给你的应用添加并发之前，你应该考虑你视什么为你应用的正确行为。理解你应用的期望行为给了你稍后验证你的设计可能，同样给了你关于引入并发可能带来的性能提升的想法。

你应该做的第一件事是遍历应用要做的任务和每个任务所需要的结构。这些任务可能包含用户行为引起的，也可能是 timer 引起的。

之后列出优先级高的任务，细分认为到小的步骤。在这个层级，你应该主要关注你对数据结构的修改和这些对象的修改怎么影响全局状态。你应该注意到不同的对象间的依赖性。对于那些修改对象不依赖任何其他对象的操作，你可以考虑将这些修改并发化。

### Factor Out Executable Units of Work
从你对应用任务的理解，你应该已经可以标识出那些地方你可以使用并发来优化你的代码。如果改变任务执行的步骤会影响最终的结果的话，你继续保持这些步骤的顺序，如果改变步骤不影响最终的结果的话，你可以考虑并发化这些步骤。这两种情况，你定义工作的单元来代替你任务中需要执行的步骤。正是这些你使用 block 或 operation 封装的工作单元被分发给相应的队列。

对于每个你标识出来的工作单元，不必太担心工作量的大小。尽管使一个线程不断的跑起来有一定的开销，分发队列和 operation queue 的一个好处就是这些开销比传统的线程要低很多。因此，使用队列比使用线程可以更高效的执行这些比较小的工作单元。当然你总是应该度量真正的性能数据，然后调整工作单元的大小。但是还是那句话，开始的时候，没有任务应该被视为太小。

### Identify the Queues You Need
现在你的任务已经被分解为不同的工作单元，使用 block 或 operation
对象封装了它们，你需要定义执行任务的队列。对于一个给定的任务，你需要考虑你创建的 blocks 或 operation 对象和它们正确完成任务需要的执行顺序。

如果你使用 blocks 来完成你的任务，你可以添加 block 到串行或并行分发队列。如果特定的顺序是需要的，你将总是将总是将 blocks 添加到串行分发队列。如果顺序不重要，你可以将 blocks 添加到并行分发队列，或根据你的需要，把它们添加到多个不同的分发队列中。

如果你通过 operation 对象来实现你的任务，队列的选择往往没配置这些对象有趣。要串行的执行这些任务，你必须配置这些对象间的依赖性。依赖性是得一个 operation 会等待它依赖的 operations 完成执行，然后再执行。

## Tips for Improving Efficiency
除了重构你的代码成小的任务，将任务加到队列，还有其他的方式使用队列来提高代码的整体效率：

* **直接在你的任务中计算如果内存使用是一个因素的话**。CPU core 的寄存器和 Cache 比内存快多了。
* **尽早标识出串行的任务，尽可能使它们更并发**。
* **避免使用 locks**
* **尽可能的依赖系统框架**

## Performance Implications	
Operations 队列，分发队列，dispatch sources 被提供来使并发执行代码更简单。然而，这些技术病不保证应用效率和响应性的提高。高效的阿满足你的需求，不给应用的资源添加过重的负担依然是你的责任。例如，尽管你可以创建 10,000 个 operation 对象，并将它们提交给 operation 队列，但是这么做会使你的应用创建大量的内存，最终降低应用的性能和体验。

在引入任何并发到你的代码之前，不论是通过队列还是线程，你都应该收集度量的一些影响应用当前性能的基本标准。在引入了这些机制后，你需要重新收集这些信息，然后对比。

## Concurrency and Other Technologies
重构你的代码成原子任务是尝试提高应用并发性的最好方式。然后这种设计尝试并不能满足所有的场景。依赖于你的任务，你可能需要其他的选择来提供应用的并发性。

### OpenCL and Concurrency
OS X 中 `Open Computing Language (OpenCL)` 一个基于标准的技术，用来在 GPU 上进行一般目的计算。如果你有定义好的计算需要在大集合数据上操作，OpenCL 是不错的技术。例如，你也许用 OpenCL 在图像的像素上进行滤镜操作，或者在多个值上进行一次复杂的数学计算。换句话说，OpenCL 专为可以并行操作的大数据集合服务。

尽管 OpenCL 很适合进行大量数据并行操作，但它不适合于一般目的的计算。需要大量的努力来准备和转移数据和 the required work kernel (不知道咋翻译) 到显卡上，以便显卡可以计算。同样需要大量的努力才能从 OpenCL 获取操作结果。

### When to Use Threads
尽管 operation 队列和分发队列是进行并发更受喜爱的方案，但他们不是万灵药。根据你的情况，你可以任然需要创建自定义的线程。当你确实需要创建线程的时候，你应该尽量创建少的线程。同时你只应该用线程解决那些用其他方式解决不了的问题。

线程任然是一种实现代码必须实时跑的方法。分发队列尽最大努力保证任务尽快跑掉，但是没有解决实时的问题。如果你需要后台跑的代码有更可以预测的行为的话，线程可能仍然是更好的选择。

---
# Operation Queues
Cocoa operations 是一种以面向对象的方式来封装你需要异步执行的任务。Operations 被设计成要么跟 operation 队列一起工作，要么自己工作。以为是基于 Objective-C，operations 可同时在 OS X 和 iOS 中使用。

## About Operation Objects
一个 *operation object* 是一个 `NSOperation` 类的实例，你是用来封装需要执行的任务。`NSOperation` 是抽象基类，如果你要做啥具体的任务，必须要通过其子类来完成。尽管是抽象基类，`NSOperation` 仍然提供了重要的基础设施，以减少子类的工作量。另外，`Foundation Framework` 提供了两个具体的子类。

Class | Description
-------- | -----------------
`NSInvocationOperation` | A class you use as-is to create an operation object based on an object and selector from your application. You can use this class in cases where you have an existing method that already performs the needed task. Because it does not require subclassing, you can also use this class to create operation objects in a more dynamic fashion.
`NSBlockOperation` | A class you use as-is to execute one or more block objects concurrently. Because it can execute more than one block, a block operation object operates using a group semantic; only when all of the associated blocks have finished executing is the operation itself considered finished.
`NSOperation` | The base class for defining custom operation objects. Subclassing NSOperation gives you complete control over the implementation of your own operations, including the ability to alter the default way in which your operation executes and reports its status.

所有的 operation 都支持以下特性：

* 支持在 operations 间建立基于图式的依赖关系
* 支持一个可选的 completion block，该 block 在任务结束后执行。
* 支持通过 KVO 观察任务执行的状态
* 支持取消的语义操作，当任务在执行的时候可以取消任务

operation 被设计来提高应用的并发性。Operation 也是一种组织并封装应用的行为程简单分离的块的好方式。不用在主线程上跑些代码，你可以把一个或多个 operation 提交到队列，让相应的工作并行起来。

## Concurrent Versus Non-concurrent Operations
尽管你通常通过把它们添加到 operation 队列来执行 operation，但是这样做并不是必须的。你可以通过手动的调用 `start` 方法来执行一个 operation 对象，但是这样做并不保证 operation 跟其他的代码是并行执行的。`NSOperation` 类的方法 `isConcurrent` 告诉你 operation 相对于调用 `start` 方法的线程是否是异步的。默认返回 `NO`，表示 operation 同步的跑在调用的线程上。

如果你需要实现一个 `concurrent operation`，也就是说，相对于调用线程而言是异步执行的，你必须写额外的代码异步的启动 operation。例如，你可能创建一个线程，调用异步系统函数，或者任何保证 `start` 方法启动任务，并立即返回，在所有的可能下，都是在任务结束前返回的。

大部分开发者应该绝不需要实现 concurrent operation 对象。如果你总是将 operation 添加到队列中，你不需要实现 concurrent operation。当你提交一个 nonconcurrent operation 到 operation 队列的时候，队列自身会创建一个跑 operation 的线程。因此，添加一个 nonconcurrent operattion 到 operation 队列仍然导致了 operation 的异步执行。定义 concurrent operation 的能力只在你不想把添加 operation 到队列的时候有需要。

## Creating an NSInvocationOperation Object
`NSInvocationOperation` 类是 `NSOperation` 的具体子类，运行时调用你指定对象上制定方法。使用这个类可以减少大量的自定义 operation 的需求；尤其是你需要修改的应用已经有一些方法做了些必要的任务的时候。

创建 invocation operation 对象很直观，传入指定的对象和 selector 就可以了，见下例：

```objc
@implementation MyCustomClass 

- (NSOperation*)taskWithData:(id)data {
NSInvocationOperation* theOp = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(myTaskMethod:) object:data];
return theOp;
}

// This is the method that does the actual work of the task.
- (void)myTaskMethod:(id)data {
// Perform the task.
}

@end
```

## Creating an NSBlockOperation Object
`NSBlockOperation` 同样是 `NSOperation` 的具体子类，用来封装一个或多个 block 的。这个类给那些已经使用了 operation 队列并不想创建分发队列的应用提供了面向对象的封装。你可以使用那些分发队列没有的一些特性，如 operation 依赖，KVO 通知等

当你创建一个 block operation 的时候，通常在初始化的时候你添加一个 block；之后你可以添加多个 block。当你执行一个 `NSBlockOperation` 对象的时候，该对象会把所有的 blocks 提交给默认优先级的并发分发队列。对象会等待所有的 block 执行完毕。当最后的一个 block 执行完后，对象会置自己的状态为完成。因此，你可以使用一个 block operation 来追踪一组执行的 blocks，就像使用一个线程 join merge 多个线程执行的结果。因为 block operation 跑在独立的线程上，应用的其他线程可以继续执行它的工作的同时可以等待 block operation 的完成。

```objc
NSBlockOperation* theOp = [NSBlockOperation blockOperationWithBlock: ^{
NSLog(@"Beginning operation.\n");
NSLog(@"Beginning operation.\n");
}];
```

创建了 block operation 对象之后，你可以使用 `addExecutionBlock:` 方法添加更多的 blocks。如果你需要串行的执行 blocks，你必须直接把 blocks 提交给指定的分发队列。

## Defining a Custom Operation Object
如果 block operation 和 invocation operation 都不能满足应用的需求，你可以直接实现 `NSOperation` 的子类，做你想做的。`NSOperation` 类对所有 operation 对象提供了一般的子类入口，也提供了大量的基础架构来处理依赖管理和 KVO 通知。然而，仍然有些时候你许不要补充已经存在的基础设施来保证你的 operations 的正确的行为。额外的工作依赖于你是需要实现一个 nonconcurrent 还是 concurrent operation。

定义一个 nonconcurrent operation 比 顶一个 concurrent operation 简单得多。对于 nonconcurrent 而言，所有你需要做的是 main task 和合理的响应 cancellation 事件；已经存在的基础设施已经帮你把其他的工作做掉。对于一个 concurrent operation 而言，你必须使用自定义的代码替换掉现有的基础设施。下来的部分将要说明怎么实现这两种类型。

### Performing the Main Task
最低要求的话，每个 operation 都应该实现以下的方法：

* 一个自定义的初始化方法
* `main`

你需要一个自定义的初始化方法将你的 operation 对象置于已知的状态，一个 `main` 方法来进行的你的任务。你当然可以根据需要实现额外的方法，如下：

* 你打算从你的 `main` 方法调用的自定义方法
* 设置数据值和获取结果的 accessor
* `NSCoding`中允许你 archive 和 unarchive operation 对象的方法

下例展示了一个自定义 `NSOperation` 的启动模板 (代码中没有展示在呢么处理 cancellation，但展示了你通常需要的方法) 

```objc
@interface MyNonConcurrentOperation : NSOperation
@property id (strong) myData;
- (id)initWithData:(id)data;
@end

@implementation MyNonConcurrentOperation

- (id)initWithData:(id)data {
if (self = [super init]) {
myData = data;
}
return self;
}

-(void)main {
@try {
// Do some work on myData and report the results.
}
@catch(...) {
// Do not rethrow exceptions.
}
}
@end
```

### Responding to Cancellation Events
在 operation 开始执行之后，它要么持续执行到任务完成，要么到显式的被你的代码取消运行。Cancellation 可以发生在任何时候，甚至在 operation 开始运行之前。尽管 `NSOperation` 类给用户提供了一种方式来取消一个 operation，但是否识别取消事件是根据必要性而自愿的。如果一个 operation 被错误的停止了，可能就没有办法回收已经分配的资源。所以，operation 对象应该在执行的过程中检查 cancellation 事件，然后优雅的退出当事件发生的时候。

operation 对象要支持 cancellation，operation 所有需要做的只是在你的自定义代码中时而调用下 `isCancelled` 方法，如果方法返回 `YES` 的时候，立即返回就可以。不管 operation 执行时长多久，或者是否你继承自 `NSOperation`，或者直接使用 `NSOperation` 的具体子类，你都有必要支持 cancellation。`isCancelled` 方法本身很轻，可以被频繁的被调用而没有任何大的性能影响。当你设计你的 operation 对象的时候，你应该考虑在你代码中的以下地方调用 `isCancelled` 方法：

* 在真正开始进行的工作之前
* 每次循环至少一次，如果单次循环确实很长的话，可以多次检查
* 在你代码中任何可以轻易终止一个 operation 的地方

下例是一个在 operation 对象的 main 方法中怎么响应 cancellation 事件非常简单的例子。 在这里例子中，每次 while 循环 `isCancelled` 方法被调用，允许一个较快的退出，在工作开始之前。

```objc
- (void)main {
@try {
BOOL isDone = NO;

while (![self isCancelled] && !isDone) {
// Do some work and set isDone to YES when finished
}
@catch(...) {
// Do not rethrow exceptions.
}
}
}
```

尽管上述代码中没有包含清理资源的代码，但是你自己的代码中应该清理任何你分配的资源。

### Configuring Operations for Concurrent Execution
operation 对象默认是同步执行的，也就是说，它们在调用 `start` 方法的线程上执行相应的任务。因为 operation 队列给 nonconcurrent operation 提供了线程，所以大部分 operations 是异步在跑。然后，如果你计划手动执行 operations，并且想要它们能够并发的跑起来的话，你必须采取合适的行为来保证它们那样做。你通过定义你的 operation 对象为 concurrent operation 来完成目的。

下表列出了你需要 override 的方法，当你实现 concurrent operations 的时候

Method | Description
----------- | -----------------
`start` | **必须** 必须 override，并以自定义的实现来替换原有行为。要手动执行 operation，你调用 `start` 方法。因此，这个方法的实现是你的 operation 的启动点，也是你准备线程或其他执行你的任务的执行环境，你的实现任何时候不要调用 super
`main` | **可选** 通常这个方法被用来实现跟 operation 相关的任务。尽管你可以在 `start` 方法中实现任务，但是通过 `main` 方法很好的将任务环境准备和任务代码做了分隔。
`isExecuting` <br> `isFinished` | **必须** Concurrent operations 负责准备好执行环境，并向它的用户报告自己环境的状态，因此，一个 concurrent operation 必须维持一些状态信息，以便知道什么时候它在执行任务，什么时候任务已经完成。然后它必须通过这些方法来报告这些状态。<br> 这些方法的实现必须是线程安全的。当改变被这些方法报告的值的时候，你必须产生合适的期望的 key path 的 KVO 通知。
`isConcurrent` | **必须** 要标识一个 operation 为 concurrent operation，override 这个方法，返回 `YES`

这节剩余的部分展示 `MyOperation` 类的样例实现，展示了实现一个 concurrent operation 的基础代码。 `MyOperation` 只是简单在它创建的线程上执行 `main` 方法。`main` 方法的具体内容在这里是不相关的。样例的意义在于展示实现一个 concurrent operation 时需要的结构是怎样的。

下例中展示了 `MyOperation` 类的接口与实现，`isConcurrent`，`isExecuting`，`isFinished` 方法相当直接。

```objc
@interface MyOperation : NSOperation {
BOOL        executing;
BOOL        finished;
}

- (void)completeOperation;
@end

@implementation MyOperation
- (id)init {
self = [super init];
if (self) {
executing = NO;
finished = NO;
}
return self;
}

- (BOOL)isConcurrent {
return YES;
}

- (BOOL)isExecuting {
return executing;
}

- (BOOL)isFinished {
return finished;
}
@end
```

下面的代码展示了 `MyOperation` 的 `start` 方法实现。方法的实现很小，只展示了你必须实现的部分。这里来说的话，就是启动了一个线程，配置它调用 `main` 方法。这个方法同样更新 `executing` 状态，并产生 `isExecuting` key path 的 KVO 通知。完成配置后，方法简单的返回，让线程执行真正的任务。

```objc
- (void)start {
// Always check for cancellation before launching the task.
if ([self isCancelled]) {
// Must move the operation to the finished state if it is canceled.
[self willChangeValueForKey:@"isFinished"];
finished = YES;
[self didChangeValueForKey:@"isFinished"];
return;
}

// If the operation is not canceled, begin executing the task.
[self willChangeValueForKey:@"isExecuting"];
[NSThread detachNewThreadSelector:@selector(main) toTarget:self withObject:nil];
executing = YES;
[self didChangeValueForKey:@"isExecuting"];
}
```

下面的代码展示了 `MyOperation` 剩下的实现。从上例中可以看出 `main` 方法是线程的入口，它执行了 operation 相关的任务，当工作完成后调用自定义的 `completeOperation` 方法。`completeOperation` 方法会 `isExecuting` 和 `isFinished` 的 KVO 通知。

```objc
- (void)main {
@try {
// Do the main work of the operation here.

[self completeOperation];
}
@catch(...) {
// Do not rethrow exceptions.
}
}

- (void)completeOperation {
[self willChangeValueForKey:@"isFinished"];
[self willChangeValueForKey:@"isExecuting"];

executing = NO;
finished = YES;

[self didChangeValueForKey:@"isExecuting"];
[self didChangeValueForKey:@"isFinished"];
}
```

即使 operation 被取消了，你总是应该通知 KVO 观察者你的 operation 已经结束了它的工作当一个 operation 依赖于其他的 operation 结束时，它会监听这些对象的 `isFinished` key path。只有当被监听的对象报告它们已经结束工作，依赖于它们的 operation 才会开始跑起来。所以如果没能成功产生一个结束的通知，会导致其他的 operation 不能执行。

### Maintaining KVO Compliance
`NSOperation` 类对以下 key path 是 KVO 的：

* isCancelled
* isConcurrent
* isExecuting
* isFinished
* isReady
* dependencies
* queuePriority
* completionBlock

如果你覆盖了 `start` 方法或大幅度的自定义一个 `NSOperation` 对象，而不是覆盖 `main` 方法，你必须保证你的自定义对象仍然保持着这些 key path 的 KVO 兼容性。当你覆盖 `start` 方法的时候，你应该关心的 key path 是 `isExecuting` 和 `isFinished`。这些 key paths 是重写 `start` 方法最常影响到的。

如果你想要实现依赖特性，以支持除 operation 对象外的对象，你需要覆盖 `isReady` 方法，在你的自定义依赖满足前强制返回 `NO` (如果你的自定义依赖管理默认 `NSOperation` 的依赖管理，你必须调用 super 的 `isReady` 方法)。当你的 operation 的 readiness 状态发生改变的时候，发出 `isReady` key path 的 KVO 通知。除非你覆盖了 `addDependency:` 或者 `removeDependency:` 方法，你不应该担心产生 `dependencies` key path 的 KVO 通知。

尽管你可以 `NSOperation` 其他 key path 的 KVO 通知，但是你不大可能需要那么做。如果你要取消一个 operation，调用 `cancel` 方法。同样，你很少需要改变 operation 的优先级信息。最后，除非你的 operation 具有动态改变并发状态的能力，你基本不需要关心 `isConcurrent` key path 的 KVO 通知。

## Customizing the Execution Behavior of an Operation Object
operation 对象的配置发生在你创建它们和添加它们到队列之间。下面描述的配置适用于所有的 operation 对象，不管是 `NSOperation` 现有的子类还是你自定义的子类。 

### Configuring Interoperation Dependencies
依赖性是一种串行 operation 对象的方法。一个依赖其他 operations 的 operation 不能执行，除非它依赖的那些 operations 都执行完了。因此，你可以使用依赖性创建 one-to-one 的依赖关系或者复杂的依赖关系图。

调用 `addDependency:` 方法可以创建依赖关系。这个方法创建单向的依赖关系，receiver 依赖于参数指定的 operation。这个依赖意味着当前的 operation 不能开始执行，直到它依赖的 operation 结束执行。依赖不限于同一个队列。Operation 对象管理着它们自己的依赖，所以创建不同的 operations 间的依赖关系并将它们添加到不同的队列中是很完全可以接受的。不可接受的是，operations 间的环形依赖。这样做是一个程序员的错误，导致没有operation 会被执行。

当一个 operation 的所有依赖都结束执行时，通常该 operation 变成准备执行。(如果你自定义了 `isReady` 方法的话，operation 的 readiness 就依赖于你设定的方案了)。如果 operation 对象在队列中，队列可能随时启动执行 operation。如果你计划手动执行 operation，那么由你来决定啥时候调用 operation 的 `start` 方法。

**注意(前方高能)**： 你总是应该在运行你的 operation 或添加他们到队列之前配置 operation 的依赖。在那之后配置依赖也许不能组织 operation 开始执行。

依赖是基于 operation 在状态发生改变时发送 KVO 通知的。如果你自定义 operation 对象的行为，为了避免引起依赖性相关的问题，你也许需要在你自定义的代码中发出合适的 KVO 通知。

### Changing an Operation’s Execution Priority
对于一个加入到队列中的 operation 而言，它们的执行顺序首先由它们的 readiness 状态来决定，然后是他们的相对优先级。operation 的 readiness 由它所以来的 operation 来决定，而 priority 则是它自身的一个属性。默认，所有新建的 operation 对象有一个 normal 的 priority，但是你可以通过调用对象的 `setQueuePriority:` 方法来修改这个属性。

prioroty 只适用于同一个队列上的 operations。如果你的应用有多个队列的话，不同队列中的 operations 的优先级是相互独立的。因此，一个不同的队列中的低优先级的 operation 可能比另一个队列中高优先级的 operation 先执行。

优先级并不是依赖性的替换品。优先级只是决定一个队列中那些处于 ready 状态的 operations 的执行顺序。例如，如果一个队列同时包含一个高优先级和低优先级的 operation，并且它们都处于 ready 状态，队列会先执行高优先级的 operation。然而，如果高优先级的 operation 没有 ready，低优先级的 operation 已经 ready，那么低优先级的 operation 先执行。

### Changing the Underlying Thread Priority
自 OS X v10.6 来，配置一个 operation 的潜在线程的优先级是可能的。线程的调度策略又是由系统的内核决定的，但是一般高优先级的线程相对于低优先级的线程被给予更多的机会。在一个 operation 对象中，你指定线程的优先级为介于 0.0 到 1.0 之间的一个值，0.0 表示最低优先级，1.0 则最高。如果你没有显示的制定线程的优先级，operation 将跑在优先级为 0.5 的线程上。

调用 operation 的方法 `setThreadPriority:` 来设置 operation 的线程优先级，当然需要在添加 opration 到队列之前。当到达执行该 operation 的时候，默认的 `start` 方法会使用你指定的值改变当前线程的优先级。这个新的优先级值只在该 operation 对象的 `main` 方法执行期间有效。其他的代码 (包括该 operation 对象的 completion block) 都执行在默认优先级下。如果你创建了一个 concurrent operation，因此你必须 override  `start` 方法，你必须自己配置线程的优先级。

### Setting Up a Completion Block
自 OS X v10.6 来，一个 operation 对象可以执行一个 completion block 当它主要的任务结束执行后。你可以使用 completion block 来执行任何你认为不是主要任何的代码。例如，你也许会利用这个 block 来通知那些关心 operation 已经完成的对象。一个 concurrent operation 对象可能使用这个 block 来产生它最后的 KVO 通知。

调用 `NSOperation` 的 `setCompletionBlock:` 方法来设置 completion block。

## Tips for Implementing Operation Objects
尽管 operation 对象很容易实现，但在你写代码的时候有些事情你还是得知晓。下面将描述一些你写 operation 对象的代码时要考虑的一些因素。

### Managing Memory in Operation Objects
下面的部分描述了在 operation 对象中正确管理内存的关键因素。

#### Avoid Per-Thread Storage
尽管大部分 operations 跑在一个线程上，对于非并发的 operations 而言，执行 operation 的线程通常由 operation 队列提供。如果队列提供了一个线程给你，你应该认为线程是属于队列的，你的 operation 不应该瞎动它。准确的说，你绝不应该关联数据到不是你创建和管理的线程上。队列管理的线程会根据系统和应用的需要新建和消亡。因此，使用 per-thread storage 在 operations 间传递数据是不可靠的，且很有可能失败。

单就 operation 对象的例子，你应该在任何情况都没有原因使用 per-thread storage。当你初始化一个 operation 对象的时候，你应该提供一个 operaton 执行它任何所需要的所有信息。因此，一个 operation
对象本身提供了它需要的 storage。所有进入和出去的数据应该被保存直到它可以被集成到你的应用中或不在需要的时候。

#### Keep References to Your Operation Object As Needed
你不应该有你可以创建 operation 并忘记它们，就因为 operation 对象异步的执行。他们仍然只是一些对象，由你来管理它们的引用。如果你需要从一个完成运行的 operation 获取结果数据的话，引用它们就更重要了。

你应该保存 operations 对象的引用的原因是之后你可能没有机会找队列要这个对象。队列努力尽快的分发并执行 operation。许多情况下，队列在添加 operation 之后立即就开始执行 operation 了。你的代码回头想从队列中活得 operation 对象的引用时，operation 对象可能已经结束执行并从队列中移除了。

### Handling Errors and Exceptions
因为 operations 本质上是应用中分散的实体，它们自己负责处理产生的错误和异常。自 OS X v10.6 起，`NSOperation` 提供的默认 `start` 方法并不捕获异常 (OS X v10.5，`start` 方法确实捕获并 suppress 异常)。你自己的代码总是应该直接捕获并 suppress 异常。你也应该检查错误代码并根据需要通知应用的其他部分。如果你 override `start` 方法的话，你可以简单的捕获任何异常来阻碍它们潜在线程的 scope。

在错误情况的类型中，你应该准备好处理以下类型：

* 检查并处理 UNIX errno-style 的 error codes
* 检查方法或函数返回的显示 error code
* 捕获你的代码或系统框架抛出的异常
* 捕获 `NSOperation` 自身抛出的异常，包含以下情况：
* 当一个 operation 没有 ready 的时候，调用了 `start` 方法
* 当一个 operation 正在执行或结束执行，它的 `start` 方法又被调用
* 当一个 operation 正在执行或结束执行，你尝试设置它的 completion block
* 当你尝试获取一个被取消了的 `NSInvocationOperation` 对象的结果

如果你的自定义代码遇到一个异常或错误，你应该采取任何需要的步骤将错误传送应用其它的代码。`NSOperation` 类没有提供显示的方法来传递 error code 或 exception 到应用的其他的部分。因此，如果这些信息很重要，你必须提供必须的代码。

## Determining an Appropriate Scope for Operation Objects
尽管可以添加随意多的 operations 对象到队列，但这么做通常不大实际。跟其他对象一样，operation 对象消耗内存，执行的时候有开销。如果每个 operation 的工作只是很小的一部分，你可能会发现分发它们消耗的时间都比它们完成任务的时间还多。如果你的应用本身已经内存很紧张，你会发现创建过多的 operations 会使性能更糟糕。

有效使用 operations 的关键点在于找到工作量大小与保持计算机处于忙状态之间的平衡点。试着保证 operation 做合理的工作量。例如，100 个 operation 对象处理 100 个值，可以考虑 10 个 operations，每个处理 10 个值。

你同样要避免一次添加过多的 operations 到队列中，或避免添加 operations 的速度快过消耗它们的速度。与其一次大量的添加 operations，不如分批量的添加。当一批 operations 完成执行后，在 completion block 中通知你的应用创建另一批。当你有很多工作要做的时候，你想要的保持队列有足够的 operations 来保持计算机处于忙碌的状态，而不是一次添加所有的 operations 消耗完计算机的内存。

当然添加 operations 的数量和单个 operation 所做工作量的大小是一个变量，由你的应用来决定的。你应该使用 Instruments 来帮助你平衡性能和速度。

## Executing Operations
最终，你的应用需要执行 operations 为了处理关联的工作。在这部分，你将学会不同的方法来执行 operations，在运行的时候怎样操作 operations 的执行。

### Adding Operations to an Operation Queue
到目前为止，执行 operation 最简单的方式是添加它们到 operation 队列中，operation 队列是 `NSOperationQueue` 类的实例。你的应用负责创建和维护它需要使用的 operation 队列。应该可以有任意数量的队列，但任一时刻多少 operation 可以执行是有实际的限制的。operation 队列跟系统协同限制并发的 operations 的数量，这个数量对于系统的负责和可用的 cores 是合适的。因此，创建更多的队列不意味着你可以执行更多的 operations。

创建队列跟创建其他的对象是一样的：

```objc
NSOperationQueue* aQueue = [[NSOperationQueue alloc] init];
```

调用 `addOperation:` 方法添加 operations 到队列。自 OS X v10.6 起，你可以使用 `addOperations:waitUntilFinished:` 方法添加一组 operations，或者调用 `addOperationWithBlock:` 方法直接添加 block 到队列中。这些方法将 operation (or operations) 排好队，并通知队列它应该开始处理它们了。大部分情况，operations 在添加到队列之后马上会被执行，但队列可能会因为某些原因延迟执行队列中的 operations。具体的说 (specifically，咋翻译了？)，operation 可能会因为它依赖的 operations 还没有结束运行而被 delay 运行。也有可能是因为队列自身被暂停了，或并发的 operations 数达到了队列最大的限制数。以下的列子展示添加 operation 到队列的基本语法：

```objc
[aQueue addOperation:anOp]; // Add a single operation
[aQueue addOperations:anArrayOfOps waitUntilFinished:NO]; // Add multiple operations
[aQueue addOperationWithBlock:^{
/* Do something. */
}];
```

尽管 `NSOperationQueue` 类是给并发执行 operations 设计的，但强制一个队列一次只能执行一个 operaiton 是可能的。调用 `setMaxConcurrentOperationCount:` 方法来设置队列并发执行 operation 的最大数。传一个 1 给这个方法的话将使队列每次只能执行一个 operation。尽管任一时刻只有一个 operation 执行，但 operation 的执行顺序仍然基于其他的因素，如 operation 的 rediness 和它的优先级。因此一个串行的 operation 队列并不提供一个串行 dispatch queue 同样的行为。如果 operation 执行顺序对你很重要的话，你应该使用依赖性来建立执行顺序。

### Executing Operations Manually
尽管 operation 队列是执行 operation 最方便的方式，没有队列我们也可能执行 operation。如果你选择手动的执行 operation，在你的代码中你需要有些预防措施。特别是，operation 必须是 ready to run，你必须通过调用 `start` 方法来启动执行。

一个 operation 对象的 `isReady` 方法返回 `YES` 的时候才被认为可以执行。`isReady` 方法被集成到 `NSOperation` 的依赖管理系统中了，提供了一个 operation 对象的依赖的状态。当一个 operation 的依赖被清楚了之后，它就可以开始执行了。

你总是通过调用 `start` 方法来手动开始执行一个 operation。使用 `start` 方法，而不是 `main` 方法是因为 `start` 方法在开始执行 operation 前做了些安全检查。特别是，默认的 `start` 方法会产生必须的 KVO 通知以使 operation 正确的处理它的依赖。这个方法避免当 operation 已被 cancled 时执行它，并跑出异常。

如果你的应用自定义了 concurrent operation 对象。你同样需要考虑调用 `isConcurrent` 方法以决定怎么执行你的 operation。该方法返回 `NO` 的时候，你的代码需要决定是否同步在当前线程执行 operation 还是新起一个线程。然后，是否做这样的检测完全由你决定。

下例展示在手动执行 operation 前你需要做的检测。如果该方法返回 `NO`，你可以规划一个 timer 晚点儿调用这个方法。直到这个方法会 `YES` 。

```objc
- (BOOL)performOperation:(NSOperation*)anOp {
BOOL        ranIt = NO;

if ([anOp isReady] && ![anOp isCancelled]) {
if (![anOp isConcurrent]) {
[anOp start];
}  else {
[NSThread detachNewThreadSelector:@selector(start) toTarget:anOp withObject:nil];
}
ranIt = YES;
} else if ([anOp isCancelled]) {
// If it was canceled before it was started, move the operation to the finished state.
[self willChangeValueForKey:@"isFinished"];
[self willChangeValueForKey:@"isExecuting"];
executing = NO;
finished = YES;
[self didChangeValueForKey:@"isExecuting"];
[self didChangeValueForKey:@"isExecuting"];

// Set ranIt to YES to prevent the operation from
// being passed to this method again in the future.
ranIt = YES;
}
return ranIt;
}      
```

### Canceling Operations
一旦 operation 加入到 operation 队列，一个 operation 对象实际上就被 operation queue 所拥有，并且不能被移除。将 operation 移除队列的唯一方式是取消它。你可以通过调用 `cancel` 方法取消单个 operation，或者调用 `cancelAllOperations` 取消队列中所有的 operation 对象。

你应该在你确定你不再需要他们的时候取消 operation。发出一个 cancel 命令使一个 operation 对象进入 "canceled" 状态，这也会阻止它继续运行。因为一个被取消的 operation 仍然被认为是 "finished"，那些依赖于它的对象将收到 KVO 通知，以便清除那个依赖。因此，往往是取消队列中所有的 operation 来响应重大事件，像退出应用，或用户特定要求取消，而不是取消单个 operation。

### Waiting for Operations to Finish
为了最好的性能，你应该把 operation 设计得尽可能的异步，是你的 APP 在 operation 跑的时候可以自由的做其他活。如果创建 operation 的代码同样处理 operation 的结果，可以调用 `NSOperation` 的 `waitUntilFinished` 方法来 block 住当前代码，直到 operation 结束。一般能的话尽量避免调用这个方法。

除了等待单个 operation 结束外，你也可以通过调用 `NSOperationQueue` 的 `waitUntilAllOperationsAreFinished` 方法来等待队列中所有其他的 operations 结束。当等待整个队列结束时，注意你的应用的其他线程仍然可以添加 operation 到队列，因此会加长等待。

### Suspending and Resuming Queues
如果你想要暂停 operations 的执行，你可以 `NSOperationQueue` 的 `setSuspended:` 方法。暂停一个队列不是导致正在运行的 operation 暂停。它只是简单组织新的 operation 被调度执行。

---
# Dispatch Queues
Grand Central Dispatch (GCD) dispath queues 是执行任务的强有力工具。dispatch queues 允许你同步或异步的执行任何 block 代码。你几乎可以使用 dispatch queues 来执行以前你在单独线程上跑的任务。dispatch queues 的优点是它们简单易用，相对于线程代码高效得多。

这章将介绍 dispatch queues，怎么使用它们来执行应用中一般的任务。

## About Dispatch Queues
Dispatch queues 是进行同步和异步任务的简单方式。 一个 *task* 是你的应用需要进行的简单任务。例如，你可以定义一个 task 来进行一些计算，创建或修改一个结构，从一个文件处理一些数据，或任意数量的事情。你可以通过定义一个函数或 block 来定义 task， 并将 task 提交给 dispatch queue。

一个 dispatch queue 是一个 object-like 的结构，管理你提交给它的任务。所有的 dispatch queue 是 first-in，first-out 的数据结构。因此，所有你提交给 dispatch queue 的任务都是已它们被添加的顺序启动执行。GCD 自动给你提供了一些 dispatch queues，但是你可以创建其他的以满足特定目的。

type | description
------ | ----------------
Serial | 串行队列 (也被称为 `private dispatch queues`) 以任务添加的顺序每次执行一个任务。当前执行的任务在一个 dispatch queue 管理的独立线程上。串行队列经常被用来同步访问特定的结构。<br /> 你可以根据你的需要创建足够多的串行队列，不同的队列间并发的执行任务。换句话说，如果你创建 4 个串行队列，每个队列每次只执行一个任务，但是最多 4 个任务仍可能并发的执行。
Concurrent | 并发队列 (也被称为 global dispatch queue 的类型) 并发的执行一个或多个任务，但是任务仍是以它们添加到队列中的顺序为准。正在被执行跑在 dispatch queue 管理的线程上。同时执行任务的是可变的，依赖于系统的条件。<br /> 自 iOS 5 后，你可以通过指定队列类型为 `DISPATCH_QUEUE_CONCURRENT` 来创建并发队列。另外，有四个预先定义好的全局并发队列给你的应用使用。
Main dispatch queue | main dispatch queue 是一个全局的串行队列，在应用的主线程上执行任务。这个队列同主线程的 run loop 一同工作，在执行队列中的任务和 run loop 上的事件源上来回交替。因为任务是在主线程上执行，main queue 经常备用作应用关键的同步点。<br /> 尽管你不需要创建 main queue，但你要保证合理的使用 main queue。

当你要给你的应用添加并发性的时候，分发队列相对于线程提供了几个优点。最直接的优点是 work-queue 变成模型的简单。使用线程的话，你需要编写任务和线程创建管理的代码。dispatch queues 让你专注于你需要执行的任务，不需要关注线程的创建与管理。系统会负责线程的创建与管理。这样做的好处是系统能够更高效的管理线程。系统可以根据可用的资源、当前的环境动态的调整线程数。另外，系统通过能够比你创建线程更快的跑你的任务。

尽管你认为重新给 dispatch queue 写代码会更难，但实际上给 dispatch queue 写代码比给线程写容易多了。编写代码最关键的点是把你的任务设计成 self-contained，能够异步的执行。dispatch queue 的另一个优点是可预测性。如果你有不同线程上的两个任务需要访问同一份数据，任一线程都可能第一个修改共享的资源，这时你需要一个锁来保证两个线程不能同时修改共享的资源。使用 dispatch queue 的话，你可以将两个任务添加到串行队列中来保证同一时刻只有一个线程修改共享的数据。这种基于队列的同步机制比锁高效多了，因为锁总是需要昂贵的 kernel trap in both contested and uncontested cases，而 dispatch queue 主要在应用的进程空间，只在比较的时候调用到 kernel。

尽管你指出串行队列中的两个任务并不是并发的执行，但是你知道如果两个线程同时请求一个锁的时候，线程提供的任何并发性也就是丧失了。最重要的是，线程模型需要创建两个线程，同时消耗了 kerne 空间和 user-space 的内存。dispatch queue 不像线程一样消耗内存，做线程做的同样的事，且不阻塞。

关于 dispatch queue 需要记住的一些关键点如下：

* dispatch queue 相对于其他的 dispatch queue 而言是并发的执行它们的任务。任务的串行是相对于一个串行队列而言的。
* 系统决定任一时刻并行任务的数量。因此一个 100 队列中的 100 个任务也许并不是是并发的执行。
* 系统会考虑队列的优先级。
* 队列中的任务必须在它被添加到队列的时候已经准备好执行。
* private dispatch queue 是基于计数的对象。除了在你的代码中持有这个队列外，依附到 dispatch queue 上的 dispatch source 也会增加它的引用计数。因此，你必须要保证所有的 dispatch source 被取消，所有的 retain 调用和 release 调用是平衡的。

## Queue-Related Technologies
除了 dispatch queue 外，Grand Central Dispatch 提供一些使用队列的技术，来帮助你管理你的代码。下表列出这些技术。

Technology | Description
---------------- | -----------------
Dispatch Group | 一个 dispatch group 是一种监听一个 block 集合的完成 (你可以根据你的需要同步或异步监听)。Groups 给依赖其他任务完成的代码提供了一个有效的同步机制。
Dispatch semaphores | 一个 dispatch semaphore 跟一个传统的 semaphore 相似，但一般跟高效。Dispatch semaphores 只在调用的线程因为该 semaphore 不可用需要被阻塞的时候调用到 kernel，如果 semaphore 可用的话，没有 kernal 调用。
Dispatch sources | 一个 dispatch source 产生通知以响应指定类型的系统事件。你可以使用 dispatch source 来监听诸如进程通知、信号、descriptor 事件。当一个事件发生的时候，dispatch source 异步的提交任务代码到指定的 dispatch queue上。

## Implementing Tasks Using Blocks
Block 对象是基于 C 的语言特性，你可以在 C，Objective-C，C++ 代码中使用。Blocks 使得定义自包含的任务单元变得容易。尽管它们看起来像函数指针，block 实际上是用一个模仿对象的结构所表示，这个结构由编译器创建和管理。编译器将你提供的代码打包封装，使得它可以存在 heap 上，并在应用中传递。

一个使用 block 的好处是 block 可以捕获包围它域中的变量。当你在函数或方法中定义 block 时，它跟传统的代码块在某些地方很像。例如，一个 block 可以读取定义在 parent scope 的变量的值。被 block 访问的变量被复制到 block 的在 heap 中的数据结构，以便 block 稍后可以访问它们。当 block 被提交到 dispatch queue 中时，这些值通常是只读的。然而，同步执行的同样可以使用那些有 `__block` 前缀的变量，来返回值给它的 parent scope。

定义 block 的语法跟定义函数指针的语法相似。两者的主要区别是名字的前缀，一个是 ^，一个是 *。如下列：

```objc
int x = 123;
int y = 456;

// Block declaration and assignment
void (^aBlock)(int) = ^(int z) {
printf("%d %d %d\n", x, y, z);
};

// Execute the block
aBlock(789);   // prints: 123 456 789
```

下面是当你定义你的 blocks 时需要考虑的关键引导的概括：

* 对于你计划异步执行的 block，从 parent 函数或方法中捕获原子类型的变量是安全的。然而你不应该捕获随着 calling context 被分配和删除的大数据结构或其他基于指针类型的变量。当 block 执行的时候，这些被指向的内存可能以及给你被释放。当然，你可以自己分配内存，显式的让 block 来管理这段内存
* dispatch queue 会 copy 添加的 block，完成执行后释放 block。换句话说，你不需要显式的复制 block
* 尽管队列相对于原生线程在执行小任务时更高效，但是创建 block 并在队列上执行它们仍然是有负载的。如果一个 block 做的活太小。可能执行任务比分发它到队列执行消耗的资源更少。分别这些方式还是用工具来测量数据对比。
* 不要依赖线程来做 blocks 间的通讯。不同的 blocks 可能会在不同的线程上执行。
* 如果你的 block 创建了大量的 Objective-C 的对象，你也许想要包含你 block 的部分代码到 `@autorelease` 块中来处理这些对象的内存管理。尽管 GCD dispatch queue 有它自己的 autoreleasing pool，但它没有保证什么时候 pool 会被 drained。如果你的应用的内存很紧张，创建 autoreleasing pool 允许你更频繁的释放那些 autoreleasing 对象的内存。

## Creating and Managing Dispatch Queues
在你添加 task 到一个队列之前，你必须决定队列的类型和你打算怎么使用它。Dispatch queue 要么串行要么并行的执行任务。另外，如果你想用 dispatch queue 做些特别的事情的话，你可以相应的配置它的属性。下面的部分将给你展示怎样创建 dispatch queues 和配置它们。

### Getting the Global Concurrent Dispatch Queues
如果你有多个任务需要并发的执行的话，一个并发的 dispatch queue 会有很用。一个并发 dispatch queue 仍是一个 first-in，frist-out 的队列，一个并发队列会在前面的任务尚未结束就取出新的任务执行。任一时刻并发队列跑的任务数是可变的，会随着应用的状态动态的改变。许多因素会影响并发队列执行的任务数，包括可用的 cores，其他 processors 目前的工作量，其他串行队列中任务的数量和优先级。

系统给每个应用提供了 4 个并发队列。这些队列对于应用是全局的，它们只是优先级有所区别。因为它们是全局的，你不用显式的创建它们。你可以通过调用 `dispatch_get_global_queue` 函数来获取这些队列。

```objc
dispatch_queue_t aQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

除了获取默认的并发队列，你也可以传递 `DISPATCH_QUEUE_PRIORITY_HIGH` 和 `DISPATCH_QUEUE_PRIORITY_LOW` 常量来获取高或低优先级的队列，或者通过 `DISPATCH_QUEUE_PRIORITY_BACKGROUND` 来获取 background 队列。和你期望的一样，在高优先级的队列中的任务比在低优先级队列中的任务优先执行。同样，默认队列中的任务总是在低优先级队列中的任务前执行。

尽管 dispatch queues 是基于引用计数的对象，你不需要 retain 和 release 全局的并发队列。因为它们是全局的，针对这些队列的 retain 和 release 调用是被忽略的。因此，你不需要持有这些对象的引用。当你需要它们的时候你只需要调用 `dispatch_get_global_queue` 函数就可以。

### Creating Serial Dispatch Queues
当你想按指定的顺序执行任务的时候串行队列将很有用。任一时刻串行队列只执行一个任务，总是从队列的头部取出任务。你也许会用串行队列而不是锁来保护共享资源或可变数据结构。不像锁，一个串行队列保证任务总是按照可预测的顺序执行。只要你异步的提交任务给一个串行队列，队列永远不会死锁。

不像并发队列，已经给你创建好了，你必须显式创建管理任何有要用的串行队列。你可以创建任意数量的串行队列，但是你应该避免创建过多只是为了同时执行尽可能多的任务。如果你想尽可能多的执行任务，你应该提交这些任务给全局并发队列。当你创建串行队列的时候，试着找到每个队列存在的目的，例如保护资源或同步应用的一些行为。

下例展示了创建自定义串行队列的步骤。`dispatch_queue_create` 函数需要两个参数：队列名称和队列属性。debugger 和 performance tools 会展示队列名称，帮助你追踪任务是怎么被执行的。队列属性为 ` DISPATCH_QUEUE_SERIAL (or NULL)` 时创建串行队列，`DISPATCH_QUEUE_CONCURRENT` 时创建并发队列。

```objc
dispatch_queue_t queue;
queue = dispatch_queue_create("com.example.MyQueue", NULL);
```

除了你创建的自定义串行队列，系统会自动创建一个串行队列，并绑定到应用的主线程上。

### Getting Common Queues at Runtime
Grand Central Dispatch 提供了让你访问一些通用队列的函数：

* 使用 `dispatch_get_current_queue` 函数来调试，或测试当前队列的标识。从 block 对象里调用这个函数可以获取 block 被提交的队列。从 block 的外面调用这个函数，获取系统提供的默认并发队列。
* 使用 `dispatch_get_main_queue` 函数获取跟应用主线程关联的串行队列。对于 Cocoa 应用和使用 `dispatch_main` 函数或在主线程上配置 run loop 的应用，这个队列是自动创建的。
* 使用 `dispatch_get_global_queue` 函数来获取共享的并发队列。

### Memory Management for Dispatch Queues
Dispatch queues 和其他 dispatch 对象是基于引用计数的类型。当你新建一个串行队列的时候，它的初始引用计数是 1。你可以调用 `dispatch_retain` 和 `dispatch_release` 来增加减少引用计数。当一个队列的引用计数为 0 时，系统异步的释放队列。

retain 和 release dispatch 对象很重要，像队列，需要保证它们在被使用的时候常住内存。和 Cocoa 对象的内存管理一样，一般的原则是如果你计划使用队列，你应该在使用前 retain，不再需要的时候 release。这样的基本模式，保证了队列在需要的时候常住内存。

### Storing Custom Context Information with a Queue
所有的 dispatch 对象允许你关联 context 数据。通过 `dispatch_set_context` 和 `dispatch_get_context` 函数来设置或获取这样的 context 数据。系统不会使用你的自定义数据，所有由你来分配和释放这些数据。

对于队列，你可以使用 context data 来存储一个 Objective-C 对象的指针，或其它能标识队列或其用途的数据结构。在队列释放前，你可以使用队列的析构函数从队列中释放 (or disassociate) 你的 context data。

### Providing a Clean Up Function For a Queue
在你创建了串行队列后，你可以提供一个 finalizer 函数来进行一些自定义的清理工作，当队列被释放时。Dispatch queue 是基于引用计数的对象，你可以使用 dispatch_set_finalizer_f 函数来指定一个函数在队列的引用计数为 0 时执行。你使用这个函数来清理跟队列关联的 context data，这个函数只在 context 指针不为 NULL 时被调用。

下例展示了一个自定义的 finalizer 函数，一个函数创建一个队列，install the finalizer。队列使用 finalizer 函数释放队列的 context pointer 存储的数据。

```objc
void myFinalizerFunction(void *context) {
MyDataContext* theData = (MyDataContext*)context;

// Clean up the contents of the structure
myCleanUpDataContextFunction(theData);

// Now release the structure itself.
free(theData);
}

dispatch_queue_t createMyQueue() {
MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
myInitializeDataContextFunction(data);

// Create the queue and set the context data.
dispatch_queue_t serialQueue = dispatch_queue_create("com.example.CriticalTaskQueue", NULL);
if (serialQueue) {
dispatch_set_context(serialQueue, data);
dispatch_set_finalizer_f(serialQueue, &myFinalizerFunction);
}
return serialQueue;
}
```

## Adding Tasks to a Queue
执行一个任务，你必须添加任务到合适的 dispatch queue。你可以同步或异步的 dispatch 任务，你也可以 dispatch 单个或一组任务。一旦进入队列，队列就开始负责尽快的执行任务，综合考虑它的限制和队列中已有任务。这部分会描述一些 dispatch 任务的技术和各自的优点。

### Adding a Single Task to a Queue
两种方式添加一个任务到队列：同步或异步。可能的话，尽量使用 `dispatch_async` 和 `dispatch_async_f` 函数来异步的添加任务。当添加一个任务到队列的时候，你没有方法知道什么时候代码会执行。结果是，异步的添加 blocks 或函数 lets you schedule the excution of the code，在调用线程中继续做其他的事。如果你是从应用的主线程来调度任务的话会显得很重要。

尽管你应该尽可能的异步添加任务，但仍然有些时候你需要同步的添加任务来避免 race conditions 或其他一些同步错误。在这些时候，你可以使用 `dispatch_sync` 和 `dispatch_sync_f` 函数来添加任务到队列。这些函数阻塞当前线程，直到指定的任务完成执行。

**重要**: 你一定不要从跑任务的队列中调用 `dispatch_sync` 或 `dispatch_sync_f` 给这个队列添加任务。这个对于串行队列尤其重要，那样做必然导致死锁。但同样也要比较在并行队列中调用。

示例如下：

```objc
dispatch_queue_t myCustomQueue;
myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);

dispatch_async(myCustomQueue, ^{
printf("Do some work here.\n");
}

printf("The first block may or may not have run.\n");

dispatch_sync(myCustomQueue, ^{
printf("Do some more work here.\n");
}

printf("Both blocks have completed.\n");
```

### Performing a Completion Block When a Task Is Done
本质上说，添加到队列上的任务代码跟创建它的代码是相互独立的。然而，当任务完成的时候，应用也许仍然想被通知到，一边应用可以获取结果。在传统的异步编程中，你也许会通过回调的机制，但是针对于 dispatch queue 的话，你可以设置 completion block。

一个 completion block 只是在原始任务结束运行后添加到 dispatch queue 的一段段代码。调用者通常在启动任务的时候提供 completion block。任务代码需要做的是当它完成工作的时候，提交指定的 block 到指定的 queue 上。

下列展示使用 block 计算平均值。averaging 函数的最后两个参数是用来回报结果时指定队列和 block的。当平均值算好之后，任务给相应的队列提交 block。

```objc
void average_async(int *data, size_t len, dispatch_queue_t queue, void (^block)(int)) {
// Retain the queue provided by the user to make
// sure it does not disappear before the completion
// block can be called.

dispatch_retain(queue);

// Do the work on the default concurrent queue and then
// call the user-provided block with the results.
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
int avg = average(data, len);
dispatch_async(queue, ^{ block(avg);});

// Release the user-provided queue when done
dispatch_release(queue);
}
}
```

### Performing Loop Iterations Concurrently
一个并发队列可能提高性能的地方是你有一个循环执行固定数量的迭代。如下：

```objc
for (i = 0; i < count; i++) {
for (i = 0; i < count; i++) {
}
```

如果每次迭代做的事情都不一样，而且每次迭代的数序不重要的话，你可以使用 `dispatch_apply` 或 `dispatch_apply_f` 来替换这样的循环。这些函数将每次迭代提交给队列。当提交给并发队列时，因此就可能同时执行多个循环迭代。

调用 `dispatch_apply` 或 `dispatch_apply_f` 时你可以指定串行或并发队列。使用并发队列可以使多次迭代同事进行，这也是使用这些方法最普遍的情况。尽管可以使用串行队列，但是这样做相对于循环并没有提供任何好处。

>**重要**: 跟一般的 `for` 循环一样，`dispatch_apply` 或 `dispatch_apply_f` 函数知道所有的迭代完成后才返回。因此当从队列的上下看已经在执行的代码中调用这些函数时你需要注意。如果你传递给这些函数的队列是一个串行队列，而且跟正在执行的队列是同一个的话，必然会导致死锁。

>因为这些函数实际上阻塞了当前线程，当你从主线程调用这些函数时你应该注意，它们会阻止不断响应事件的事件处理循环。如果你的循环代码需要能被感知到的时间量的话，你也许想要从另一个线程上调用这些方法。

下列展示了怎么使用 `dispatch_apply` 替换之前的 `loop`。传递给 `dispatch_apply` 函数的 block 必须有一个参数，标识当前的迭代。当 block 执行的时候，第一个迭代的参数值为 0，最后一个为 count - 1.

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_apply(count, queue, ^(size_t i) {
printf("%u\n",i);
}
```

跟任何添加到队列的 block 和 function 一样，你应该保证你的 task code 做了合理的一定量的工作在每次迭代中。因为调度这些任务是有消耗的。如果调度的成本高了并发带来的性能提升，这是不好的。

### Performing Tasks on the Main Thread
Grand Central Dispatch 提供了一个特殊的队列，你可以使用这个队列在应用的主线程上跑任务。 This queue is provided automatically for all applications and is drained automatically by any application that sets up a run loop (managed by either a CFRunLoopRef type or NSRunLoop object) on its main thread. If you are not creating a Cocoa application and do not want to set up a run loop explicitly, you must call the dispatch_main function to drain the main dispatch queue explicitly. You can still add tasks to the queue, but if you do not call this function those tasks are never executed.

你可以通过调用 `dispatch_get_main_queue` 来获取主线程的 dispatch queue。添加到这个队列的任务串行的在主线程上执行。因此，你可以把这用作应用其他地方工作的同步点。

### Using Objective-C Objects in Your Tasks
GCD 对 Cocoa 的内存管理技术提供了内建的支持，你可以在提交给 dispatch queue 的 blokc 里面使用 Ojbective-C 的对象。每个 dispatch queue 维护了自己的 autoreleasing pool，它们会在某个时间点保证 autoreleased 的对象 release 掉。但队列不保证什么时候它们 release 这些对象。

如果你的应用内存紧张，创建你自己的 autoreleasing pool 是唯一保证你创建的对象按时的释放。如果你创建了大量的对象，你可能需要创建多个 autoreleasing pool 来保证更频繁的释放对象。

## Suspending and Resuming Queues
你可以在执行的 blocks 中暂停 dispatch queue 来暂时阻止 dispatch queue 运行。你使用 `dispatch_suspend` 函数暂停，`dispatch_resume` 恢复运行。调用 `dispatch_suspend` 函数增加队列的 suspension reference count，调用 `dispatch_resume` 减少这个 reference count。只要 suspension reference count 大于 0，队列就会保持暂停状态。因此这两个调用应该是平衡的。

> **注意**: suspend 和 resume 是异步调用，只在 blocks 之间生效。suspend 一个队列不会导致正在执行的 block 暂停。

## Using Dispatch Semaphores to Regulate the Use of Finite Resources
如果提交到队列的任务访问有限的资源，你可能想要使用一个 dispatch semaphore 来管理同时访问该资源的任务数量。dispatch semaphore 跟一般的 semaphore 除了一点外很像。当资源可用时，获取资源的 dispatch semaphore 时间比系统的少。因为 GCD 这种情况下没有调用到内核。唯一需要调用到内核的时候是资源不可用的时候，系统需要线程等待到资源可用。

使用 dispatch semaphore 的步骤如下：

1. 使用 `dispatch_semaphore_create` 函数创建 semaphore，使用一个正数标识可用资源数。
2. 在每个任务中调用 `dispatch_semaphore_wait` 等待资源可用。
3. 当方法返回的时候，获得资源，做相应的任务
4. 当使用完资源，释放它，并调用 `dispatch_semaphore_signal` 给 semaphore 发送信号。

举个例子来说这些步骤是怎么工作的，考虑使用系统上的 file descriptor。每个应用只能使用有限的 file descriptor。如果你有一个应用处理大量的文件的话。你不会想要一次打开过多的文件，以至一次就用尽了 file descriptor。你可以使用信号量来限制 file descriptor 的使用数量。代码如下：

```objc
// Create the semaphore, specifying the initial pool size
dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);

// Wait for a free file descriptor
dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
fd = open("/etc/services", O_RDONLY);

// Release the file descriptor when done
close(fd);
dispatch_semaphore_signal(fd_sema);
```

当你创建 semaphore 的时候，你指定可用资源的数量。这个值成为 semaphore 初始的计数值。每次你在 semaphore 上等待时，`dispatch_semaphore_wait` 函数将计数减 1。如果这个值为负数时，函数调用会告诉 kernel 阻塞当前线程。另外一端 `dispatch_semaphore_signal` 函数将计数增加 1，表示资源已被释放。如果有任务因为等待资源而阻塞的话，它们中的一个会获得资源开始运行。

## Waiting on Groups of Queued Tasks
Dispatch groups 是一种阻塞一个线程等待一个或多个任务完成执行的方式。你在那些需要等待指定任务结束后进行某些任务的地方。例如，在分发了几个任务来完成计算之后，你也许会使用一个 group 来等待这些计算任务完成，并处理它们的结果。另一种使用 dispatch group 的方式是 thread join 的替换。与其创建多个子线程，然后 join 每个子线程，你可以添加相应的任务到 dispatch group 里，并等待这个 group 完成执行。

下列展示设置一个 group，分发任务给它，等待结果的基本过程。你使用 `dispatch_group_async` 函数而不是 `dispatch_async` 。等待一个 group 的任务完成，你使用 `dispatch_group_wait` 函数。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
dispatch_group_t group = dispatch_group_create();

// Add a task to the group
dispatch_group_async(group, queue, ^{
// Some asynchronous work
});

// Do some other work while the tasks execute.

// When you cannot make any more forward progress,
// wait on the group to block the current thread.
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);

// Release the group when it is no longer needed.
dispatch_release(group);
```

## Dispatch Queues and Thread Safety
在 dispatch queue 的上下文讨论线程安全看起来很怪，但是线程安全仍是一个相对的话题，任何时候你在你的应用中实现并发，你都应该知道以下点：

* dispatch queue 它们自身是线程安全的。也就是说你可以在任何线程给 dispatch queue 提交任务，不需要任何同步机制。
* Do not call the dispatch_sync function from a task that is executing on the same queue that you pass to your function call. Doing so will deadlock the queue. If you need to dispatch to the current queue, do so asynchronously using the dispatch_async function.
* 避免在提交到 dispatch queue 中的任务上使用锁。尽管在任务内使用锁是安全的，但如果锁不可用的话，你会将整个串行队列阻塞住。如果你需要同步机制，使用串行队列，而不是锁。
* 尽管你可以跑任务的底层线程信息，但最好避免这么做。

---
# Dispatch Sources
跟底层系统打交道的时候，你必须对任务会一定量的时间有所准备。调用到 kernel 或系统的其他层涉及到上下文的切换，相对于发生在用户进程中调用而言是很昂贵的。因此，许多系统框架会提供异步的接口，允许你的代码提交一个调用调用的请求，然后继续做你的工作。GCD 是基于这种通用行为，允许通过使用 blocks 和 dispatch queues 来提交你的请求和返回结果到你的代码。

## About Dispatch Sources
*dispatch source* 是一个基本的数据结构，用来协调处理底层的系统事件。GCD 支持以下类型的 dispatch source:

* timer dispatch source 产生间隔的通知
* signal dispatch source 通知你 UNIX signal
* descriptor sources 通知各种基于 file 和 socket 操作的通知：
* 当有数据可供读
* 当能够写数据
* 当文件被删除、移动、重命名
* 当 file meta 信息被改动
* process dispatch source 通知进程相关的事件：
* 当进程退出
* 当进程发出一个 `fork` 或 `exec` 类型的调用
* 当 signal 被发送给进程
* Mack port dispatch source 通知 Mach 相关的事件
* 自定义 dispatch source 是一些你自己定义的

Dispatch source 代替系统调用通常用在处理系统相关的事件。当你配置一个 dispatch source 时，你指定你关心的事件、dispatch queue 和处理这些事件的代码。处理事件的代码可以是 block 对象或函数。当一个事件发生的时候，dispatch source 会提交 block 或函数到指定的队列执行。

跟手动提交任务到 dispatch queue 不同，dispatch source 给应用提供了持续的事件源。一个 dispatch source 会保持附着在 dispatch queue 上，直到你显式的取消它。只要 dispatch source 附着在 dispatch queue 上，相应的事件发生的时候就会提交 block 或函数到相应的队列上。像 timer 的一些事件是有规律间隔发生，但大部分时间是在特定条件下才发生的。因为这个原因，dispatch source 会 retain 关联的 dispatch queue，阻止 dispatch 在还有事件等待处理之前被释放掉了。

为了阻止事件在队列中堆积，dispatch source 实现了事件合并的机制。如果一个新事件在上一个事件的 handler 被取出队列并被执行之前到达，dispatch source 会把新事件中的数据和老事件中的数据合并。合并事件依赖于事件的类型，合并可能会替换或更新老事件的数据。例如，一个基于 signal 的 dispatch source 只提供最新 signal 的信息，同时也报告自上次调用 event handler 之后有多少事件被传递。

## Creating Dispatch Sources
创建 dispatch source 涉及到创建事件的源和 dispatch source 自身。事件的源是处理事件所需的任何 native 数据结构。例如，对于一个 descriptor-based dispatch source 你需要打开 the descriptor，对于 process-based source 你需要获取目标程序的进程 ID。当你有事件源时，你可以按照一些步骤创建 dispatch source：

1. 使用 `dispatch_source_create` 函数创建
2. 配置 dispatch source:
* 设置 dispatch source 的 event handler
* 对于 timer source，使用 `dispatch_source_set_timer` 设置事件信息
3. 选择性的设置 dispatch source 的 cancellation handler
4. 调用 `dispatch_resume` 来开始处理事件。

因为 dispatch source 在可以工作之前需要一些其他的配置，`dispatch_source_create` 函数会返回一个处于 suspended 状态的 dispatch source。当 dispatch source 处于 suspended 状态时，dispatch source 会收到事件，但不处理它们。这给了你时间来设置 event handler 和处理事件所需要的其他配置。

下面的部分展示怎么配置 dispatch source。可参见  Grand Central Dispatch (GCD) Reference 来查看创建和配置 dispatch source 的函数。

### Writing and Installing an Event Handler
为了处理 dispatch source 产生的事件， 你必须定义处理这些事件的 handler。一个时间处理 handler 是一个 blcok object 或函数，通过调用 `dispatch_source_set_event_handler` 或 `dispatch_source_set_event_handler_f` 函数来安装到 dispatch source 上。当事件发生的时候，dispatch source 会将事件处理的 handler 提交到指定的 dispatch queue 上处理。

event handler 的主体是负责处理到达的事件。如果一个新事件发生，之前的 event hander 已经进入队列并等待调用处理事件，dispatch source 会合并两个事件。一个 event handler 通常只看到最新的事件。根据 dispatch source 的不同类型，event handler 可能也能获得其他发生并被合并事件的信息。如果一个或多个事件在 event handler 开始执行之后发生，dispatch source 会将这些时间 hold 住，直到当前的 event handler 结束执行。在那之后，它会提交新的 event handler 来处理新的事件。

基于函数的 handler 需要一个包含 dispatch source 对象的 context 指针，没有返回值。基于 block 的 handler 不需要参数，也没有返回值。

```objc
// Block-based event handler
void (^dispatch_block_t)(void)

// Function-based event handler
void (*dispatch_function_t)(void *)
```

在 event handler 内，你通过 dispatch source 对象来获取给定事件的信息。尽管基于函数的 handler 被传了一个指向 dispatch source 的指针，block-based handler 必须自己去捕获那个指针。你可以通过引用指向这个 dispatch source 的变量来达成这点。如下，下面的代码捕获了 source 变量，这个变量定义在 block 的 scope 外。

```objc
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, myDescriptor, 0, myQueue);

dispatch_source_set_event_handler(source, ^{
// Get some data from the source variable, which is captured
// from the parent context.
size_t estimated = dispatch_source_get_data(source);

// Continue reading the descriptor...
});

dispatch_resume(source);
```

在 block 内部捕获变量通常是为了允许极大的方便性和动态性。当然，默认捕获的变量只是 read only 的。尽管某些情况下，blocks 是支持修改捕获的变量，但是在与 dipatch souce 关联的 event handler 中不应该这么做。dispatch source 是异步的执行它的 event handler，当 event handler 被调用的时候那些捕获的变量所定义的 scope 往往已经没了。想要了解 blocks 内部捕获和使用变量的更多信息，可以参见 `Blocks Programming Topics`。

下表列出了 event handler 代码获取关于事件信息的函数调用

Funcation  | Description
--------------  | ----------------
dispatch_source_get_handle | 这个函数返回 dispatch source 管理的底层系统数据类型。<br />对于一个 从文件读数据的 discriptor dispatch source，这个函数返回可供读取的字节数。<br /> 对于一个写数据到文件的 descriptor dispatch source，如果有可用的空间的话，这个函数返回一个正数。<br /> 对于一个监听系统文件系统行为的 descriptor dispatch source，这个函数返回一个常量表示发生的事件类型。查看 `dispatch_source_vnode_flags_t` 枚举类型有哪些类型。<br /> 对于一个 process dispatch source，这个函数返回一个常量来表示发生事件的类型。查看 `dispatch_source_proc_flags_t` 有哪些类型。 <br />对于一个 Mach port dispatch source，这个函数返回返回一个常量，表示发生事件的类型。查看 `dispatch_source_machport_flags_t` 有哪些类型。<br />对于自定义的 dispatch source，这个函数返回由传递给 `dispatch_source_merge_data` 函数已有数据和新数据的返回值。
dispatch_source_get_mask | 返回你创建 dispatch source 时你使用的 flags。<br />对于一个 process dispatch source，这个函数返回 dispatch source 所受到事件的 mask。<br /> 对于一个有 send right 的 Mach port dispatch source，这个函数期望事件的 mask。<br />对于一个自定义 OR 的 dispatch source 而言，这个返回返回用于 merge data 的 mask。

### Installing a Cancellation Handler
cancellation handler 被用来在 dispatch source 释放前清理它。对于大部分 dispatch source， cancellation handler 是可选的，只在你绑定到 dispatch source 的自定义行为同样需要更新的时候才需要。但是对于使用 descriptor 或 Mach port 的 dispatch source 而言，你必须提供一个 cancellation handler 来关闭 descriptor 或释放 Mach port。没有这么做会导致那些无意使用这些结构的代码或系统其他部分产生微妙的 bugs。

你可以在任何时刻 install cancellation handler，但一般在创建 dispatch source 时做。你调用 `dispatch_source_set_cancel_handler` 或 `dispatch_source_set_cancel_handler_f` 函数来 intall，具体哪个函数依赖于你使用 block 还是函数来作为 dispatch source 的 cancellation handler。下面的示例展示了一个关闭 descriptor 的 cancelllation handler。

```objc
dispatch_source_set_cancel_handler(mySource, ^{
close(fd); // Close a file descriptor opened earlier.
}
```

### Changing the Target Queue
尽管当你创建 dispatch source 时指定了执行 event handler 和 cancellation handler 的 dispatch queue，你还是可以在任何时候调用 `dispatch_set_target_queue` 来改变这个队列。你可能调用这个方法来改变 dispatch source 的事件被处理的优先级。

改变一个 dispatch source 的队列是一个一步操作，dispatch source 会尽自己最大的努力是改变尽快发生。如果一个 event handler 已经进入队列并等待执行，它会在之前的队列中执行。而在你改变队列的时候到来的时间，可能再两者之一的队列上执行。

### Associating Custom Data with a Dispatch Source
跟 Grand Central Dispatch 当中其他的数据累心相似，`dispatch_set_context` 函数被用来关联自定义的数据到 dispatch source 上。你可以使用 context 指针保存任何处理事件所需要的数据。当你保存自定义的数据的时候，你也需要提供 cancellation hanlder 来释放那些数据，当 dispatch source 不再被需要的时候。

如果是使用 blocks 来实现 event handler 的话，你可以通过 capture local variable 来在 block 中使用使用这些变量。尽管这减少了存储数据到 dispatch source 的 context 指针的需求，但是你始终应该谨慎的使用 block 的这个特性。因为 dispatch source 也是常驻应用的，当你 capture 包含指针的变量时你应该小心。如果指针指向的数据可能在任何时候被释放掉，你要么复制这块数据，要么 retain 来防止数据释放掉。所以不论是使用 block 的这个特性还是 dispatch source 的 context 指针，你都需要提供一个 cancellation handler 来释放数据。

### Memory Management for Dispatch Sources
跟其他的 dispatch 对象一样，dispatch source 是基于引用计数的对象。一个 dispatch source 初始的时候计数为 1，通过 `dispatch_retain` 和 `dispatch_release` 来 retain 和 release。当计数为 0时，系统自动释放。

以为它们的使用方式，dispatch source 的 ownership 相对于它自己而言要么被内部管理，要么被外部管理。外部管理的时候，一个对象或一段代码或得 dispatch source 的 ownership，当不再需要的时候负责释放它。内管管理的时候，dispatch source 拥有自己，负责在合适的时候释放自己。尽管外部管理很普遍，但你也许会在想要创建自管理的 dispatch source 并让它管理你的代码的行为的时候用到内部管理。例如，如果一个 dispatch source 被设计成处理一个全局事件，你可能会让它处理这个事件，并自动退出。

## Dispatch Source Examples
下面的部分将给你展示怎样创建和配置一些常用的 dispatch source

### Creating a Timer
Timer dispatch source 会产生固定事件间隔的时间。你可以使用它来创建一个需要有规律间隔的任务。例如，游戏和其他图形密集的应用也许会用 timer 来更新屏幕或动画。你也可以设置一个 timer 来检查一个经常更新的 server。

所有的 timer dispatch source 都是 interva timers，也就是说，一旦创建，它们会有规律在你指定的间隔发送事件。当你创建一个 timer dispatch source 的时候，其中一个你需要指定的值是 leeway 值，让系统知道你期望的 timer 事件的精度。 Leeway 值给了系统灵活性来管理能耗和唤醒 cores。例如，系统可能使用 leeway 值提前或延迟发送 timer 事件，与系统的其他的事件对齐。当你创建 timer 时你应该尽可能指定一个 leeway 值。

> **Note** 即使你指定 leeway 值为 0，你也不应该期望 timer 事件在你请求的 nanosecond 发生。系统只是尽可能的协调但不保证满足。

当一个电脑进入 sleep 状态时，所有的 timer 进入被暂停。电脑唤醒时，这些 dispatch timer 也被自动的唤醒。根据 timer 的配置，暂停的这个属性可能会影响 timer 下次什么什么 fire。如果你的 timer 是使用 `dispatch_time` 函数或 `DISPATCH_TIME_NOW` 常量，你的 timer 会使用默认的 system clock 来绝对什么时候 fire，然而，默认的 system clock 在电脑 sleep 的时候并不会前进。对比下，当你通过 `dispatch_walltime` 函数的时候，timer 是基于 wall clock time 来计时的。后一个常常适用于时间间隔较大的 timer，因为可以阻止事件之间有太大的漂移。

下列中展示了一个间隔 30s leeaway 1s 的timer。因为间隔较大，dispatch source 使用 `dispatch_walltime` 来创建的。timer 初次会立即 fire，之后每 30s 到达一次。

```objc
dispatch_source_t CreateDispatchTimer(uint64_t interval, uint64_t leeway,
dispatch_queue_t queue, dispatch_block_t block) {
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

if (timer) {
dispatch_source_set_timer(timer, dispatch_walltime(NULL, 0), interval, leeway);
dispatch_source_set_event_handler(timer, block);
dispatch_resume(timer);
}
return timer;
}

void MyCreateTimer() {
dispatch_source_t aTimer = CreateDispatchTimer(30ull * NSEC_PER_SEC, 1ull * NSEC_PER_SEC, dispatch_get_main_queue(), ^{ MyPeriodicTask(); });

// Store it somewhere for later use.
if (aTimer) {
MyStoreTimer(aTimer);
}
}
```

尽管创建一个 timer dispatch source 是获取基于时间事件的主要方式。但也有其他的方式。如果你想在一段时间之后执行一个 block，可以调用 `dispatch_after` 或 `dispatch_after_f` 函数。这个函数跟 `dispatch_async` 行为很像，除了它允许你指定一个时间值，在指定的时间提交相应的 block 到指定的队列上。时间可以根据你的需要设置成相对或是绝对的。

### Reading Data from a Descriptor
从一个文件或 socket 读数据，你必须打开文件或 socket，并创建一个 `DISPATCH_SOURCE_TYPE_READ` 类型的 dispatch source。你提供的 event handler 应该能够读取并处理 file descriptor 的内容。对于文件而言，以为着读取文件的内容，给应用创建相应的结构。对于网络 socket 而言，涉及到处理新收到的网络数据。

无论在什么时候读取数据，你都应该配置你的 descriptor 来使用非阻塞的操作。尽管你可以使用 `dispatch_source_get_data` 函数来获取有多少数据可供读取，但是这个函数返回的值可能在你调用这个函数和实际读操作之间发生改变。如果潜在的文件被 truncated 或发生网络错误，从一个 descriptor 读数据会阻塞当前线程，使得事件处理的代码在中间暂停，最后阻止 dispatch queue 继续分发其他的任务。对于一个 serial queue，这样可能会造成死锁。即使对一个并发队列而言，这也会减少可以被启动的新任务数。

下例展示了怎样配置一个从文件读取数据的 dispatch source。这个例子中，event handler 从指定的文件中读取所有内容到一个 buffer 中，然后调用自定义方法处理数据。当没有数据可读取的时候，为了保证不阻塞 dispatch queue，这个例子使用了 `fcntl` 函数配置 file descriptor 来进行非阻塞的的操作。dispatch source 上的 cancelllation handler 保证了在读取数据结束后 file descriptor 的关闭。

```objc
dispatch_source_t ProcessContentsOfFile(const char* filename) {
// Prepare the file for reading.
int fd = open(filename, O_RDONLY);
if (fd == -1) return NULL;
fcntl(fd, F_SETFL, O_NONBLOCK);  // Avoid blocking the read operation

dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t readSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, fd, 0, queue);

if (!readSource) {
close(fd);
return NULL;
}

// Install the event handler
dispatch_source_set_event_handler(readSource, ^{
size_t estimated = dispatch_source_get_data(readSource) + 1;
// Read the data into a text buffer.
char* buffer = (char*)malloc(estimated);
if (buffer) {
ssize_t actual = read(fd, buffer, (estimated));
Boolean done = MyProcessFileData(buffer, actual);  // Process the data.
// Release the buffer when done.
free(buffer);

// If there is no more data, cancel the source.
if (done) {
dispatch_source_cancel(readSource);
}
}
});

// Install the cancellation handler
dispatch_source_set_cancel_handler(readSource, ^{close(fd);});

// Start reading the file.
dispatch_resume(readSource);
return readSource;
}    
```

上例中，MyProcessFileData 函数决定了什么时候读取了足够多的数据和 dispatch source 可以被取消。默认情况下，一个从文件读取的 dispatch source 在有数据可以读的时候就调度一个 event handler。如果 socket 断开连接或到达了文件尾部，dispatch source 会自动停止调度 event handler。如果你知道你不再需要一个 dispatch source，你可以直接取消它。

### Writing Data to a Descriptor
写数据到 socket 或文件跟读很想。再配置了一个用于写的 descriptor 之后，你创建一个 `DISPATCH_SOURCE_TYPE_WRITE` 类型的 dispatch source。一旦 dispatch source 创建，系统就调用 event handler 以给它一个机会来写数据到文件或 socket。当你结束写数据之后，使用 `dispatch_source_cancel` 来取消 dispatch source。

同样，无论什么时候写数据，你都应该配置你的 file descriptor 以便使用非阻塞操作。尽管你可以使用 `dispatch_source_get_data` 来获取有多少空间可供写，但这个值只是指导性的。在你获取这个值和实际写操作之间可能已经发生改变。如果写的时候发生错误，且使用阻塞式的写操作的话，会导致 event handler 执行到一半，并阻碍 dispatch queue 分发任务。

下列展示了使用 dipatch source 写数据的基本方式。在创建新文件后，传递结果 file descriptor 到 event handler。写入的数据由 `MyGetData` 函数提供，你可以使用你自己的代码替换掉。在写入数据到文件后，event handler 取消 dispatch source，阻止被再次调用。

```objc
dispatch_source_t WriteDataToFile(const char* filename) {
int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, (S_IRUSR | S_IWUSR | S_ISUID | S_ISGID));
if (fd == -1) return NULL;
fcntl(fd, F_SETFL); // Block during the write.

dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t writeSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_WRITE, fd, 0, queue);
if (!writeSource) {
close(fd);
return NULL;
}

dispatch_source_set_event_handler(writeSource, ^{
size_t bufferSize = MyGetDataSize();
void* buffer = malloc(bufferSize);

size_t actual = MyGetData(buffer, bufferSize);
write(fd, buffer, actual);

free(buffer);

// Cancel and release the dispatch source when done.
dispatch_source_cancel(writeSource);
});

dispatch_source_set_cancel_handler(writeSource, ^{close(fd);});
dispatch_resume(writeSource);
return (writeSource);
}
```

### Monitoring a File-System Object
如果你想监听一个文件系统对象的变化，你可以设置一个 `DISPATCH_SOURCE_TYPE_VNODE` 类型的 dispatch source。你用这种类型的 dispatch source 来接受关于文件被删除、写入、重命名的通知。你也可以使用它观察文件的 meta info 的改变。

>  **NOTE**: 你指定给 dispatch source 的 file descriptor 在 dispatch source 处理事件的时候必须保持 open

下例监听文件名字的修改并执行一些自定义的操作当改变发生的时候。因为一个 file descriptor 为指定的 dispatch source 打开了，dispatch source 需要包含一个 cancellation handler 来关闭 descriptor。Because the file descriptor created by the example is associated with the underlying file-system object, this same dispatch source can be used to detect any number of filename changes. (不懂这句，by @0oneo)

```objc
dispatch_source_t MonitorNameChangesToFile(const char* filename) {
int fd = open(filename, O_EVTONLY);
if (fd == -1) return NULL;

dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE, fd, DISPATCH_VNODE_RENAME, queue);
if (source) {
// Copy the filename for later use.
int length = strlen(filename);
char* newString = (char*)malloc(length + 1);
newString = strcpy(newString, filename);
dispatch_set_context(source, newString);

// Install the event handler to process the name change
dispatch_source_set_event_handler(source, ^{
const char*  oldFilename = (char*)dispatch_get_context(source);
MyUpdateFileName(oldFilename, fd);
});

// Install a cancellation handler to free the descriptor
// and the stored string.
dispatch_source_set_cancel_handler(source, ^{
char* fileStr = (char*)dispatch_get_context(source);
free(fileStr);
close(fd);
});

// Start processing events.
dispatch_resume(source);
} else  {
close(fd);
}
return source;
}
```

### Monitoring Signals
UNIX signals 允许从应用外操作应用。一个应用可以收到许多类型的 signals，从不可恢复的错误到关于重要信息的通知。通常应用使用 `sigaction` 函数来设置一个 signal handler 函数，signal handler 同步的处理到达的 signal。如果你只想被通知一个 signal 被收到了但不想处理它，你可以使用 signal dispatch source 来异步的处理 signal。

signal dispatch source 并不是 `sigaction` 函数设置的同步处理 signal 的替换。同步 signal handlers 可以截获一个 signal，避免它终止你的应用。Signal dispatch source 允许你监听到来的信号。另外，你不用使用 signal dispatch source 来接受所有类型的 signal。准确的说，你不能使用来监听 `SIGILL`、`SIGBUS`、`SIGSEGV` signals。

因为 signal dispatch source 是在一个队列上异步的执行，他们没有同步 signal handler 的一些限制。例如，在 signal dispatch source 的 event handler 中你调用任何函数没有限制。这里的 tradeoff 是灵活性的提高，也意味着信号到达的时间和信号处理时的时刻之间是有 latency。

下例展示了怎么配置一个 dispatch source 处理 `SIGHUP` signal.

```objc
void InstallSignalHandler() {
// Make sure the signal does not terminate the application.
signal(SIGHUP, SIG_IGN);

dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGHUP, 0, queue);

if (source) {
dispatch_source_set_event_handler(source, ^{
MyProcessSIGHUP();
});

// Start processing signals
dispatch_resume(source);
}
}
```

如果你在开发自定义的框架，使用 signal dispatch source 的一个好处是监听 signal 的代码跟链接框架的应用代码相互独立。Signal dispatch source 不会干涉其他的 dispatch source 或任何同步信号处理 handler。

### Monitoring a Process
一个 process dispatch source 允许你监听一个指定进程的行为，并做出相应的响应。一个父进程也许会使用这种类型的 dispatch source 来监听它创建的子进程。同样子进程也可以监听父进程，当父进程退出的时候自己也退出。

下列展示了设置一个 dispatch source 监听父进程结束的步骤。当父进程死亡的时候，一个 dispatch source 设置一些内部状态信息让子进程知道它应该退出了 (你的应用需要实现 `MySetAppExitFlag` 函数来设置一个结束的合适标志)。因为 dispatch source 是独立自主的跑的，因为它 owns 自己，并程序退出的过程中 cancel 和 release 自己。

```objc
void MonitorParentProcess() {
pid_t parentPID = getppid();

dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_PROC, parentPID, DISPATCH_PROC_EXIT, queue);
if (source) {
dispatch_source_set_event_handler(source, ^{
MySetAppExitFlag();
dispatch_source_cancel(source);
dispatch_release(source);
});
dispatch_resume(source);
}
}
```

## Canceling a Dispatch Source
dispatch source 保持活跃状态，知道你调用 `dispatch_source_cancel` 函数取消它。Cancelling 一个dispatch source 会停止事件的传递，并且不可以被取消。因此，通常你取消 dispatch source 后，会立即释放它：

```objc
void RemoveDispatchSource(dispatch_source_t mySource) {
dispatch_source_cancel(mySource);
dispatch_release(mySource);
}
```

取消一个 dispatch source 是一个异步操作。尽管在你调用 `dispatch_source_cancel` 后不会有新事件被处理，但是正在被处理的事件会继续被处理。在它处理完最后的事件后，dispatch source 会调用 cancellation handler 如果有的话。

cancellation handler 是你替 dispatch source 释放内存清理资源的机会。如果你的 dispatch source 使用了一个 descriptor 或 Mach port，你必须提供一个 cancellation
handler 来关闭 descriptor 或销毁 mach port。

## Suspending and Resuming Dispatch Sources
你可以调用 `dispatch_suspend` 或 `dispatch_resume` 来暂停或恢复 dispatch source 分发事件。这些函数会增加和减少你的 dispatch 对象的 suspend count。因此 `dispatch_suspend` 和 `dispatch_resume` 的调用必须是平衡的。

当你 suspend 一个 dispatch source 时，任何当 dispatch source 处于 suspended 状态时发生的事件会被累积起来，直到 dispatch source 被恢复。当 dispatch source 被恢复后，事件会被合并成一个单一的事件被传递，而不是没有的事件都被传递。合并事件阻止了它们在队列上堆积，并使你的应用反应不过来。

---
# Migrating Away from Threads
有许多方式使现有的多线程代码使用 Grand Central Dispatch 和 operation
对象。尽管并不是所有的情况都可以从线程转移过来，但在那些你转移的地方，性能和代码的简单性会大大的提升。准确的说，使用 dispatch queues 和 operation queues 相对于线程有以下优点：

* 在应用的内存空间中减少了应用存储线程栈的损耗
* 减少了创建和配置线程的代码
* 减少了管理和调度线程的代码
* 简化了你写的代码

这章提供了一些关于怎样使用 dispatch queue 和 operation queues 替换现有线程代码做同样事的 tips 和引导。

## Replacing Threads with Dispatch Queues
要理解怎样使用 dispatch queue 替换线程，首先你需要考虑你的应用目前是怎样使用线程：

* **单一任务线程**. 创建一个线程来完成一个任务，任务完成的时候释放线程
* **Worker threads**. 创建一个或多个 worker 线程，每个 worker 执行特定的任务。间隔的分发任务给 worker 
*  **Thread pools**. 创建一个通用线程池，并给每个形成设置 run loop。当你有一个任务要做，从线程池中取一个线程，分发一个任务给它。如果没有空闲线程，将任务排入队列，等待可用线程。

尽管这些技术看起来有很大的不同，他们只是单一原则的变种。每种情况都是，一个线程被用来执行应用需要做的任务。唯一的不同是被用来管理线程和将任务排队。通过 dispatch queue 和 operation queue 你可以消除你所有线程和线程通讯的代码，专注于你需要做的任务。

如果你使用上述一种线程模型，你应该很清楚你的应用要执行的任务。你可以试着封装这些任务成一个 operation 或 block 对象，并提交到合适的队列，而不是将这些任务提交给线程。对于那些什么争议的任务——也就是说，那些没有使用锁的任务——你可以使用以下技术替换：

* 对于当个线程任务，封装任务成一个 block 或 operation，提交给并发队列。
* 对于 worker 线程，你需要决定是使用串行队列还是并发队列。如果你使用 worker thread 来同步执行一些任务，使用串行队列。如果你使用 worker threads 来执行任意没有依赖的任务，使用并发队列。
* 对于线程池，封装任务成一个 block 或 operation，提交给并发队列。

当然，简单的替换可能并总有效。如果你执行的任务共享资源，理想的方案是首先移除或最小化资源。如果有方式重构掉相互间的依赖的话，最好移除。然后，如果这么做并不可能或更低效的话，仍然是有方法使用队列的。使用队列的优点是对于执行的代码它们提供了可预测性。这种可预见性意味着仍然有方式不用锁或比较重的同步机制来同步执行代码。与其使用锁，你可以使用队列完成相同的任务：

* 如果你有任务必须按照指定的顺序执行，把这些任务提交到串行队列。如果你喜欢使用 operation queue，使用 operations 之间的依赖来保证这些对象按照指定的顺序执行
* 如果你目前在使用锁来保护共享资源，创建一个串行队列任何修改资源的任务。于是串行队列就作为同步机制替换掉了现有的锁。
* 如果你线程使用 join 来等待后台线程完成，考虑使用 dispatch group 替换。你可以可以使用 `NSBlockOperation` 对象或 operation 对象依赖来达成 group-completion 的行为。
* 如果你使用 producer-consumer 算法来管理一个有限的资源。见后文
* 如果你使用线程来从 descriptor 读写，或监听 file 操作，使用前文描述的 Dispatch source。

记住队列并不是替换线程的万灵药。队列提供的同步变成模型很适合延迟不重要的场景。即使队列提供了配置任务优先级的机制，高优先级并不保证任务在指定的时刻执行。因此，线程仍然在那些你需要低延迟的场景更合适，如音视频的播放。

## Eliminating Lock-Based Code
对于多线程代码，锁是传统的同步访问共享资源的机制。然而使用锁是有成本的。即使是在非竞争的情况下，获取一个锁总是有性能损耗的。再竞争情况下，就有一个或多个线程被阻塞住不定的时间，等待所的释放。

使用碎裂替换基于锁的代码消除了与锁相关的损耗，同样简化了你的代码。与其使用锁来保护共享的资源，你可以创建一个串行队列，让任务串行的访问资源。队列没有一样的性能损耗，例如，队列不需要陷入到内核中为了获得锁。

当将任务排队的时候，你需要做的决定是使用串行队列还是并行队列。异步的提交任务允许当前的线程继续执行的同时任务也可以执行。同步的提交任务会阻塞当前线程知道任务完成。两种情况都有相应的场景，尽管异步的提交任务总是更好的。

### Implementing an Asynchronous Lock
一个异步锁是一种不阻塞任何修改共用资源代码用来保护共享资源的方式。你可以使用 asynchronous lock 当你需要修改一个数据结构，而修改这个数据结构会对你的代码正在做的其他任务有副作用的时候。使用传统的线程，你的实现方式经常是需要使用锁来保护共享资源。然而，使用 dispatch queue，调用代码异步的做这些修改，不用等待这些修改完成。

下列展示了一个异步锁的实现。这个例子中，共享资源定义自己的串行队列。调用代码提交一个包含修改这个资源的 block 对象给这个队列。因为队列串行的执行这些 block，这些改变保证以它们被提交的顺序执行。然后，因为这些任务是异步的执行，调用线程不会阻塞。

```objc
dispatch_async(obj->serial_queue, ^{
// Critical section
});
```

### Executing Critical Sections Synchronously
如果你目前的代码必须等待给定的任务完成，你可以使用 `dispatch_sync` 函数同步的提交任务。这个函数添加任务到 dispatch queue，然后阻塞当前线程直到任务完成。dispatch queue 自身可以是串行或并行，取决于你的需要。因为这个函数阻塞当前线程。你应该在需要的时候使用。下列的代码中使用 `dispatch_sync` 来 wrap 了一个 critical section：

```objc
dispatch_sync(my_queue, ^{
// Critical section
}
```

如果你已经使用一个串行队列来保护一个共享资源，同步的分发任务并不比异步的分发任务更能保护共享资源。同步分发任务的唯一原因是阻止当前代码继续执行直到 critical section 结束。例如，如果你想要从共享资源中立即 get 一些值的话，你将需要使用同步的机制。如果你不需要等待 critical section 结束，或如果可以简单提交一系列的任务给串行队列的话，异步的提交往往是更好的。

## Improving on Loop Code
如果你的循环的每次迭代都是相互独立的话，你也许应该考虑使用 `dispatch_apply` 或 `dispatch_apply_f` 重新实现你的循环。这两个函数将每个迭代提交给队列处理。当和并行队列一起使用的时候，这个特性让你能够同时进行多个迭代。

`dispatch_apply` 和 `dispatch_apply_f` 函数是同步函数调用，会阻塞当前线程，直到所有的循环迭代完成。当提交给并发队列的时候，循环迭代的执行顺序是不保证的。执行这些迭代的线程可能会阻塞，导致给定的迭代在之前或之后完成。因此，这个 block 对象或函数必须是可 reentrant。

下例展示了一个 `for` 循环的 dispatch-based 替代。传递给 `dispatch_apply` 或 `dispatch_apply_f` 的参数必须带有一个 integer 的参数表示当前的迭代。

```objc
queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(count, queue, ^(size_t i) {
printf("%u\n", i);
});
```

尽管前面的例子很简答，但它展示了使用 dispatch queue 替换循环的基本技术。尽管这个技术可以提供基于循环代码的性能，使用这项技术时，你必须有一定的洞察力。尽管 dispatch queue 有很小的负载，但是调度每次迭代仍有一定的损耗。因此，你必须保证你的迭代代码做足够多的工作来抵消掉这些损耗。具体多少工作你得用性能工具来做评测。

一个简单提高每次迭代工作量的方式是 striding。striding 意味着重写 block 代码，多做些原始循环的迭代。然后减少 `dispatch_apply` 函数中指定的 count 值。下例中显示上例中循环怎样按照 striding 实现。

```objc
int stride = 137;
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)

dispatch_apply(count / stride, queue, ^(size_t idx){ 
size_t j = idx * stride;
size_t j_stop = j + stride;
do {
printf("%u\n", (unsigned int)j++);
} while (j < j_stop);
});

size_t i;
for (i = count - (count % stride); i < count; i++) {
printf("%u\n", (unsigned int)i);
}

使用 strides 一定会有性能提升。尤其是当原来的循环迭代次数很大的时候。分发更少的 block 到队列以为更多的时间花在执行 block 上。跟任何性能度量一样，你可能需要去调整这个 striding 值来达到最好的性能。

## Replacing Thread Joins
线程 join 允许你创建一个或多个子线程，然后让当前的线程等待，直到这些线程结束运行。为了实现一个线程 join，父进程创建一个 `joinable thread`。当父线程没有子线程的结果不再有任何进展时，it joins with the child。这个过程阻塞了父线程，直到子线程完成任务并退出，这个时候父线程收到子线程的结果，继续执行自己的任务。如果父线程需要 join 多个子线程，它一次只能一个。

dispatch group 提供了跟 thread join 相似的语义，但有其他的优点。跟线程 join 相似，dispatch groups 是一个线程阻塞，直到一个或多个子任务完成执行。不一样的是，一个 dispatch group 同时等待所有的子任务完成执行。因为 dispatch group 使用dispatch queue 来完成任务，它们很高效。

要使用 dispatch group 来完成 joinable 线程完成的任务， 你需要按以下步骤做：

1. 调用 `dispatch_group_create` 创建一个 dispatch group
2. 调用 `dispatch_group_async` 或 `dispatch_group_async_f` 函数添加任务。每个提交的任务代表着你在 joinable 线程上执行的工作
3. 当当前的线程不能再进一步的时候，调用 `dispatch_group_wait` 函数等待。这个函数阻塞当前线程，直到 group 中的所有任务完成

如果你使用 operation 对象来完成你的任务， 你使用依赖完成 thread join。你将父线程的代码移到一个 operation 对象中，让它依赖其他完成子线程任务的 operation 对象。依赖使得完成子线程的 operations 完成之后才会执行完成父线程任务的 operation。

## Changing Producer-Consumer Implementations
一个 producer-consumer 模型允许你管理有限动态生成的资源。当 producer 创建资源 (or work)，一个或多个 consumers 等待这些资源被准备好，然后消耗这些资源。实现一个 producer-consumer 模式的典型机制是 conditions 或 semaphores。

使用 conditions，producer 线程通常做以下事情：

1. 使用 `pthread_mutex_lock` 锁住与 condition 相关的 mutex
2. 生产被消耗的资源或任务
3. 使用 `pthread_cond_signal` 给 condition 变量发送信号，有资源可被 consume
4. 解锁 mutex

相应的 consumer 线程要做的如下：

1. 使用 `pthread_mutex_lock` 锁住与 condition 相关的 mutex
2. 设置一个 while loop 做以下事情：
1. 检查是否有工作要做
2. 如果没有工作或资源，调用 `pthread_cond_wait` 阻塞当前线程，直到收到相应的信号
3. 获取 producer 生产的任务或资源
4. 解锁 mutex
5. 进行工作

使用 dispatch queue 的话，你通过以下一个调用来简化 producer 和 consumer 的行为：

```objc
dispatch_async(queue, ^{
// Process a work item.
});
```

当你的 producer 有工作要做的时候，它所需要做的只是添加相应的工作到一个队列，让队列来处理。上面的代码唯一需要改的部分是队列的类型。如果你的 producer 产生的任务需要特定的顺序，使用串行队列。如果你的 producer 产生的任务可以并发的执行，把它们添加到并发队列中。

## Replacing Semaphore Code
如果你使用 `semaphores` 来限制访问共享资源，你应该考虑使用 `dispatch semaphores` 替换。传统的 semaphores 总是会调用掉内核代码来测试 `semaphores`。相比之下，`dispatch semaphores` 在用户空间很快的测试状态，只在测试失败的时候调用内核代码阻塞调用线程。`dispatch semaphores` 的行为相对于传统的要快多了。在所有的其他面上，`dispatch semaphores` 跟传统的行为一致。

## Replacing Run-Loop Code
如果你使用 run loop 来管理一个或多个线程的任务的话，你也许会发现使用 dispatch queue 实现和维护都简单很多。设置一个自定义 run loop 涉及到设置潜在的线程和 run loop 自身。run loop 代码又由设置一个或多个 run loop source 和处理这些 source 到达事件的回调组成。因此，你可以简单创建一个串行队列，并分发任务给它。因此，你可以使用以下代码替换 run loop 创建的代码。

```objc
dispatch_queue_t myNewRunLoop = dispatch_queue_create("com.apple.MyQueue", NULL);
```

因为队列会自动执行那些提交给它的任务，你不需要其他的代码来管理这个队列。你也不需要创建或配置线程，你也不需要创建或 attach 任何 runloop sources。另外，你只需要简单的添加任务给队列就可以了。使用 run loop 做同样的事，你需要修改 run loop sources 或创建一个来处理新数据。

run loops 的一个通用配置是用来处理异步到来的网络数据。与其使用 run loop 来完成这样的行为，不如使用 dispatch source。dispatch source 相对于传统的 run loop sources 提供了更多的选项。除了处理 timer 和网络端口事件，你可以使用 dispatch source 来处理文件读写，监听系统文件，监听进程，监听 signals。你甚至可以自定义 dipatch source，异步的从你代码的其他部分 trigger。

## Compatibility with POSIX Threads
因为 Grand Central Dispatch 管理了你提供的任务和这些执行任务的线程，你应该尽量避免使用 POSIX 线程 API 从你的任务代码。如果因为某些原因你这么做了，你应该小心你调用了哪些 API。这部分将提供哪些 API 在你的任务代码中调用是安全的，哪些不是。这份列表不全，但是应该可以给你一些关于哪些是安全哪些不是的指示。

一般，你的应用一定不要删除或改变不是它创建的对象或数据结构。因此，队列执行的 blocks 对象不应该调用以下 API:

> pthread_detach
> pthread_cancel
> pthread_join
> pthread_kill
> pthread_exit

尽管当任务被执行的时候可以改变线程的状态，但你必须使线程回到开始执行任务的状态。因此只要你使线程回到开始的状态，调用以下 API 就是安全的：

> pthread_setcancelstate
> pthread_setcanceltype
> pthread_setcanceltype
> pthread_sigmask
> pthread_setspecific

在调用之间执行给定 block 的潜在线程可以改变。结果是你的应用在不同的 blocks 调用间不应该依赖以下函数返回的结果：

> pthread_self
> pthread_getschedparam
> pthread_get_stacksize_np
> pthread_get_stackaddr_np
> pthread_mach_thread_np
> pthread_from_mach_thread_np
> pthread_getspecific