本文来自 [Mike Ash](https://www.mikeash.com/) 的 [Objective-C Messaging](https://www.mikeash.com/pyblog/friday-qa-2009-03-20-objective-c-messaging.html) ，主要是关于Objective C的消息发送是怎么工作的，消息发送到底做了什么？


###一些定义
在我们解释机制之前，我们需要定义一些terms，很多人不大清楚method与message的区别，但是这对于深刻理解消息发送机制是怎么工作又至关重要。
	
* **Method**: 跟类(class)关联的一段代码，并有一个名称，例如：`- (int)meaning { return 42; }`
* **Message**: 发送给一个对象的一个名字和一串参数。例如：发送 “meaning” 且没有参数给对象 `0x1234567`.
* **Selector**: 一种特殊的方式来表示 mesage 或 method 的名字，类型是 `SEL`，Selector本质上是字符串，但没有对开发者开放出来，可以直接通过指针对边地址来比较他们，(from @0oneo， 有点儿像ruby中的Symbol). Example: @selector(meaning).
* **Message send**:  接受 message，查找并执行相应的 method 的过程。


### Methods
接下来我们从机器的角度看一个Method到底是什么。从定义的角度看，它是一段与类关联的代码，但是它最终在你应用的二进制中是怎么呈现的了？

Methods 最终以 C 函数的形式呈现，并额外增加了几个参数。你也许知道 `self` 作为隐式参数被传入，当然在 C 函数中最终以显示的参数出现。很少人知道的另一个隐式参数 `_cmd` 也是如此。写一个以下的Method:

``` objc
	- (int)foo:(NSString *)str { ...
```

会被转换成以下函数：

```objc
   int SomeClass_method_foo_(SomeClass *self, SEL _cmd, NSString *str) { ...
```

那么写以下代码时发生了什么了？

```objc
  int result = [obj foo: @"hello"];
```

编译器会生成以下等效的代码：

```
int result = ((int (*)(id, SEL, NSString *))objc_msgSend)(obj, @selector(foo:), @"hello");  
```

这段代码做的事情是将 Objective C runtime 的函数 `objc_msgSend` 转换成一个接受 id, SEL, 和原 method 被调用时的参数列表，返回 int 类型的函数。

**NOTE**：这里有个矛盾的点，尽管编译器在这里只需要处理这个被发送的 message， 编译器仍然需要知道 method 的原型。编译器很简单的处理的这个矛盾，它根据已经解析到目前的 method prototype 来进行猜测。如果没有找到或有一个mismatch的话，运行的时候就会有不好的事情发生。这也是为什么 Objective C 在处理 methods 拥有同样的名字，但有不同的参数和返回类型时，表现很差经的原因。

### Messaging
代码中的消息发送会被转换成  `objc_msgSend` 的函数调用， 但是它是干啥的了？简单说的话很简单，因为只是一个函数调用，显然它是查找相应的 method 实现并调用。调用很简单：跳到相应的地址即可。但是它是怎么查找相应的实现了？

Objctive C 的头文件 `runtime.h` 包含了 objc_class 结构的成员:

```objc
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
``` 

```objc
struct objc_method_list {
        struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;
    
        int method_count                                         OBJC2_UNAVAILABLE;
    #ifdef __LP64__
        int space                                                OBJC2_UNAVAILABLE;
    #endif
        /* variable length structure */
        struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}
```

```objc
struct objc_method {
        SEL method_name                                          OBJC2_UNAVAILABLE;
        char *method_types                                       OBJC2_UNAVAILABLE;
        IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                                OBJC2_UNAVAILABLE;
```

即使我们不应该去碰这些结构（当然 Apple 有提供函数来操作这些结构），我们仍然可以看出 runtime 认为一个 method 是什么。它是包含一个名字、一个包含参数和返回类型的字符串（查阅@encode）、和一个 IMP， 而 IMP 只是一个函数指针，类型如下：

```objc
    typedef id 			(*IMP)(id, SEL, ...);
```

现在我们看下这些是怎么工作的。所有的 `objc_msgSend` 都是查找对象的 class，获取 class 的 method list，并搜索 method list 知道找到相应的 selector，如果没找，搜索 superclass，以此类推。如果找到，跳到 IMP 执行。

另一个细节需要注意，上面查找并调用执行的过程很慢。`objc_msgSend` 在 x86 上只花了几个 CPU 周期，显然它没有通过上面的过程。这是由于 `objc_class` 的另一个成员：

```objc
struct objc_cache {
        unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
        unsigned int occupied                                    OBJC2_UNAVAILABLE;
        Method buckets[1]                                        OBJC2_UNAVAILABLE;
}
``` 

这里定义了一个保存 Method 的 hash table，使用 selector 作为 key。`objc_msgSend` 首先 hash selector
然后查找 class 的 method cache，如果找到，直接跳到执行就是了，没找到走上面说的过程。
