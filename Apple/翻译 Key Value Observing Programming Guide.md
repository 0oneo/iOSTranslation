## 简介
KVO 是一种实现观察者模式的机制，当被观察者的某些属性发生改变的时候，通知相应的观察者。

---
## Registering for Key-Value Observing
为了能收到一个属性的 KVO notifcation，有三个条件必须满足：
* class 被观察的 property 必须是 KVO compliant 的
* 注册观察过程，调用 `addObserver:forKeyPath:options:context:`
* 观察者必须实现 `observeValueForKeyPath:ofObject:change:context:` 方法

### Receiving Notification of a Change
如果 property 是一个对象，值会直接提供。如果 property 是一个原子类型或 C struct，值会被 warpped 在一个 `NSValue` 实例中。

---
## KVO Compliance
KVO compliant property 必须满足以下条件：

* property 必须要是 KVC compliant
* class 发出 property 的 KVO 变更通知
* Dependent keys are registered appropriately （这里不知道咋翻译）

有两种技术保证 change notification are emitted。自动支持已经由 `NSObject` 提供，且默认适用于所有 KVC compliant 的 property。mannual change notification 能更好的控制什么时候发送通知。通过 override `automaticallyNotifiesObserversForKey:` 来控制是否自动发送通知。

### Automatic Change Notification
`NSObject` 提供了 automatic key-value change notification 的基本实现。Automatic change notification 通过调用 KVC compliant accessors or key-value coding methods 来通知 observer change notifcation。示例代码：

```objc
// Call the accessor method.
[account setName:@"Savings"];

// Use setValue:forKey:.
[account setValue:@"Savings" forKey:@"name"];

// Use a key path, where 'account' is a kvc-compliant property of 'document'.
[document setValue:@"Savings" forKeyPath:@"account.name"];

 // Use mutableArrayValueForKey: to retrieve a relationship proxy object.
Transaction *newTransaction = <#Create a new transaction for the account#>;
NSMutableArray *transactions = [account mutableArrayValueForKey:@"transactions"];
```

### Manual Change Notification
manual change notification 能够控制 how and when 发送通知给 observer。一个 class 实现 manual change notifcation 必须实现 `automaticallyNotifiesObserversForKey:`，示例代码如下：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)theKey {
  BOOL automatic = NO;
  if ([theKey isEqualToString:@"openingBalance"]) {
    automatic = NO;
  } else {
    automatic = [super automaticallyNotifiesObserversForKey:theKey];
  }
  return automatic;
}
```

要实现 manual observer notification， 在 changing the value 前调用 `willChangeValueForKey:`，发生改变后调用 `didChangeValueForKey: `。示例代码如下：

```objc
- (void)setOpeningBalance:(double)theBalance {
  [self willChangeValueForKey:@"openingBalance"];
  _openingBalance = theBalance;
  [self didChangeValueForKey:@"openingBalance"];
}
```

你可以通过检查 value 是否发生改变来减少发送不必要的通知，示例代码如下：

```objc
- (void)setOpeningBalance:(double)theBalance {
  if (theBalance != _openingBalance) {
    [self willChangeValueForKey:@"openingBalance"];
    _openingBalance = theBalance;
    [self didChangeValueForKey:@"openingBalance"];
  }
}
```

对于 an ordered to-many relationship，除了指出哪个 key changed，同样需要指明 type of change 和 indexes of  the objects involved。代码示例如下：

```objc
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
  [self willChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"transactions"];
  
  // Remove the transaction objects at the specified indexes.
  
  [self didChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"transactions"];
}
```

---
## Registering Dependent Keys
有些情况一个 property 的值依赖于 one or more attributes in another object，如果被依赖的某个 attribute 值变了，依赖于它们的 property 的值也应该是变了。怎么保证 KVO notification are posted for those property？

### To-one Relationships
To trigger notifcations for a to-one relationship 你要么override `keyPathsForValuesAffectingValueForKey:`，要么实现 registering dependent keys 所规定的 pattern。如下，`fullName` 依赖于 `firstName`，`lastName`。

```objc
- (NSString *)fullName {
  return [NSString stringWithFormat:@"%@ %@",firstName, lastName];
}
```

如果有 observer observe `fullName` 时，`firstName` `lastName`之一有啥改变的时候，observer 也应该收到 `fullName` 变更的通知。

一种方案是 override `keyPathsForValuesAffectingValueForKey:` 指定 `fullName` 依赖于 `lastName` 和 `firstName`，示例代码如下:

```objc
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
  NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
  
  if ([key isEqualToString:@"fullName"]) {
    NSArray *affectingKeys = @[@"lastName", @"firstName"];
    keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
  }
  return keyPaths;
}
```

你也可以通过遵循 naming convention 的 class method 来完成同样的功能。naming convention 是 `+ (NSSet *)keyPathsForValuesAffecting<Key>`。示例代码如下：

```objc
+ (NSSet *)keyPathsForValuesAffectingFullName {
  return [NSSet setWithObjects:@"lastName", @"firstName", nil];
}
```

### To-many Relationships
pls check the document 

--- 
reference: 

* [Apple's KVO Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html) 