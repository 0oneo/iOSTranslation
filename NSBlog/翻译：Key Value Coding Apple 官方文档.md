## 它是啥？
一种间接访问对象属性的机制，通过字符串来标识属性，而不是直接调用 accessor method 或者 直接访问 instance variable。本质上而言，key-value coding 定义了一些模式和应用的 access method 所应有的 method signature。

key-value coding 定义的必须方法申明在 `NSKeyValueCoding` Objective-C informal protocol 中，`NSObject` 有提供默认实现。

---
## 一些术语
Key-value coding 可以被使用于三种不同的对象值类型：attributes，to-one relationship，to-many relationships.

`attribute` 是一个属性（property），属性的值是一个简单的值，如原子类型，字符串，bool 类型。如 `NSNumber`、`NSColor` 之类的 value object 也认为是 attributes。

`to-one relationship`  property 是一个拥有自己 properties 的对象，它的 properties 可以随意修改，但是它自身不变（地址不变，immutable object的话就必须要变了）。

`to-many relationship` proerty 是由相关的 objects 组成的 colletion。一个 `NSArray` or `NSSet` 的实例常被用于 hold 这样一个 collection。

---
## 机制

### Keys and Key Paths
一个 key 标识了对象的一个 property，就是一个简单的字符串。

key path 是 dot separated keys， 被用于标识一串需要遍历的 keys

### 通过 KVC 获取 attributes
调用 `valueForKey:`，如果没有相应的 accessor 或者 instance variable 的话，receiver 将收到一个 `valueForUndefinedKey:` 消息，这个方法的默认实现就是抛一个 `NSUndefinedKeyException` 的异常。

### 通过 KVC 设置 attribute
调用 `setValue:forKey:` 设置 key 的 value。默认实现会自动 unwrap 代表原子类型、结构体的 `NSValue`  实例，并使用它们赋值。如果 key 不存在，receiver 会收到 `setValue:forUndefinedKey:` 消息，该方法实现同 `valueForUndefinedKey:`。

---
## KVC Accessor methods
为了让`valueForKey:`，`setValue:forKey:`，`mutableArrayValueForKey:`，`mutableSetValueForKey:` 找到相应的 accessor methods， accessor method 应该遵循 KVC 规定的 pattern。

### 通用的 Accessor Patterns
getter 的通用格式是 `- <key>`。`- <key>` 通常返回一个对象原子类型，数据结构。对于 bool 类型，getter 可以使用另外一个类型 -` is<Key>`。

```objc
- (BOOL)isHidden {
  // Implementation specific code.
  return ...;
}

- (BOOL)hidden {
  // Implementation specific code.
  return ...;
}
```

setter 的通用格式是 `- set<Key>:` 。**注意** 如果 attribute 是非对象类型的话，当试着将该 attribute 赋值为 nil 时，KVC 会发送 `setNilValueForKey:` 消息给该对象，这时你应该做相应的替换。

```objc
- (void)setNilValueForKey:(NSString *)theKey {
  if ([theKey isEqualToString:@"hidden"]) {
    [self setValue:@YES forKey:@"hidden"];
  } else {
    [super setNilValueForKey:theKey];
  }
}
```

### Collection Accessor Patterns for To-Many Properties
尽管可以使用 `-<key>` 和 `-set<Key>:` 的 accessor form，通常你用它们返回创建 collection 对象。但是如果需要通过 KVC 来操作 collection 中的内容的话就需要实现 `collection accessor methods`。

有两种 `collection accessors`:
1. indexed accessors for ordered  to-many relationships (typically represented by `NSArray`)
2. unordered accessors (represented by `NSSet`).

#### Indexed Accessor Pattern

##### Getter Indexed Accessors
In order to support read-only access to an ordered to-many relationship, implement the following methods:

* `-countOf<Key>`. Required. This is the analogous to the NSArray primitive method count.
* `-objectIn<Key>AtIndex:` or `-<key>AtIndexes:`. One of these methods must be implemented. They
correspond to the `NSArray` methods `objectAtIndex:` and `objectsAtIndexes:`.
* `-get<Key>:range:`. Implementing this method is optional, but offers additional performance gains. This method corresponds to the `NSArray` method `getObjects:range:`.

`-countOf<Key>` 的实现简单的返回 `to-many relationship` 中 object 的数量。可参见如下代码。

```objc
- (NSUInteger)countOfEmployees {
  return [self.employees count];
}
```

` -objectIn<Key>AtIndex:` 返回 `to-many relationship` 中指定 index 的 object。` -<key>AtIndexes:` 返回 `NSIndexSet` 实例指定的 indexes 处的对象数组。实例代码如下：

```objc
- (id)objectInEmployeesAtIndex:(NSUInteger)index {
  return [employees objectAtIndex:index];
}
- (NSArray *)employeesAtIndexes:(NSIndexSet *)indexes {
  return [self.employees objectsAtIndexes:indexes];
}
```

如果 benchmarking 表示性能提升是必须的，可以实现 `-get<Key>:range:`。实例如下：

```ojbc
- (void)getEmployees:(Employee * __unsafe_unretained *)buffer range:(NSRange)inRange {
  // Return the objects in the specified range in the provided buffer.
  // For example, if the employees were stored in an underlying NSArray
  [self.employees getObjects:buffer range:inRange];
}
```

##### Mutable Indexed Accessors
实现 `mutable indexed accessors` 允许你通过使用 `mutableArrayValueForKey:` 返回的 `array proxy` 来跟 indexed collection 进行交互。

对于 mutable ordered to-many relationship，你必须实现以下方法：

* `-insertObject:in<Key>AtIndex:` or ` -insert<Key>:atIndexes:`，必须实现其一
* `-removeObjectFrom<Key>AtIndex:` or `-remove<Key>AtIndexes:`，必须实现其一
* `-replaceObjectIn<Key>AtIndex:withObject:` or `-replace<Key>AtIndexes:with<Key>:`，可选

`-insertObject:in<Key>AtIndex:` 在指定的 index 插入对象。` -insert<Key>:atIndexes: ` 在 `NSIndexSet` 指定的 indexes 插入 array 中的对象。示例代码如下：

```objc
- (void)insertObject:(Employee *)employee inEmployeesAtIndex:(NSUInteger)index {
  [self.employees insertObject:employee atIndex:index];
  return;
}

- (void)insertEmployees:(NSArray *)employeeArray atIndexes:(NSIndexSet *)indexes {
  [self.employees insertObjects:employeeArray atIndexes:indexes];
  return; 
}
```

`-removeObjectFrom<Key>AtIndex:` 删除指定 index 处的对象。`-remove<Key>AtIndexes:` 删除 `NSIndexSet` 指定的对象。实例代码如下：

```objc
- (void)removeObjectFromEmployeesAtIndex:(NSUInteger)index {
  [self.employees removeObjectAtIndex:index];
}
- (void)removeEmployeesAtIndexes:(NSIndexSet *)indexes {
  [self.employees removeObjectsAtIndexes:indexes];
}
```

**注意** 如果 benchmarking 标识需要提升性能，你可以同时或择其一实现 `-replaceObjectIn<Key>AtIndex:withObject:` `-replace<Key>AtIndexes:with<Key>:` 。示例代码如下：

```objc
- (void)replaceObjectInEmployeesAtIndex:(NSUInteger)index withObject:(id)anObject {
  [self.employees replaceObjectAtIndex:index withObject:anObject];
}
- (void)replaceEmployeesAtIndexes:(NSIndexSet *)indexes withEmployees:(NSArray *)employeeArray {
  [self.employees replaceObjectsAtIndexes:indexes withObjects:employeeArray];
}
```

#### Unordered Accessor  Pattern
##### Getter Unordered Accesssors
为了支持 read-only 式访问 unordered to-many relationship，你需要实现以下方法：

* `-countOf<Key>`  Required. This method corresponds to the NSSet method count.
* `-enumeratorOf<Key>`. Required. Corresponds to the NSSet method objectEnumerator.
* `-memberOf<Key>:`. Required. This method is the equivalent of the NSSet method member:

简单的示例代码如下：

```objc
- (NSUInteger)countOfTransactions {
  return [self.transactions count];
}
- (NSEnumerator *)enumeratorOfTransactions {
  return [self.transactions objectEnumerator];
}
- (Transaction *)memberOfTransactions:(Transaction *)anObject {
  return [self.transactions member:anObject];
}
```

##### Mutable Unordered Accessors
mutable unordered accessors 为了 KVC complaint 需要实现以下方法：

* `-add<Key>Object:` or `-add<Key>:`. At least one of these methods must be implemented. These are analogous to the `NSMutableSet` method `addObject:`
* `-remove<Key>Object:` or `-remove<Key>:`. At least one of these methods must be implemented. These are analogous to the `NSMutableSet` method `removeObject:`.
* `-intersect<Key>:`. Optional. Implement if benchmarking indicates that performance is an issue. It performs the equivalent action of the `NSSet` method `intersectSet:`.

示例代码如下

```objc
- (void)addTransactionsObject:(Transaction *)anObject {
  [self.transactions addObject:anObject];
}
- (void)addTransactions:(NSSet *)manyObjects {
  [self.transactions unionSet:manyObjects];
}

- (void)removeTransactionsObject:(Transaction *)anObject {
  [self.transactions removeObject:anObject];
}

- (void)removeTransactions:(NSSet *)manyObjects {
  [self.transactions minusSet:manyObjects];
} 

- (void)intersectTransactions:(NSSet *)otherObjects {
  return [self.transactions intersectSet:otherObjects];
}
```

---
## Key-Value Validation
KVC 提供了一致的 API 来 validate  属性的值。the validation infrastructure 给一个 class 提供了接受一个值，class 提供另一个值，或拒绝新的值并报错。

### Validation Method Naming Convention
跟 accessor methods 一样，validation method 同样是依赖 convention 的。一个 `validation method` 格式是 `validate<Key>:error:`，示例如下：

```objc
-(BOOL)validateName:(id *)ioValue error:(NSError * __autoreleasing *)outError {
  // Implementation specific code.
  return ...;
}
```

### Implementing a Validation Method
validation method 接受两个参数，一个是 value object to validate，一个 NSError 来返回错误。 对于一个 `validation method` 有三种返回结果：
1. 如果 value object 合法，返回 `YES`
2. value object 不合法，且不能创建一个合法的 new object，返回 `NO`，并且用 `NSError` 实例来标识错误。
3. 一个 new object 被创建，返回 `YES	`.

示例代码如下：

```objc
-(BOOL)validateName:(id *)ioValue error:(NSError * __autoreleasing *)outError {
  // The name must not be nil, and must be at least two characters long.
  if ((*ioValue == nil) || ([(NSString *)*ioValue length] < 2)) {
    if (outError != NULL) {
      NSString *errorString = NSLocalizedString( @"A Person's name must be at least two characters long",
        @"validation: Person, too short name error");
      NSDictionary *userInfoDict = @{ NSLocalizedDescriptionKey : errorString};
       *outError = [[NSError alloc] initWithDomain:PERSON_ERROR_DOMAIN 
         code:PERSON_INVALID_NAME_CODE   
         userInfo:userInfoDict];
     }
     return NO;
  }
  return YES;
}
```

### Invoking Validation Methods
你可以直接调用或者调用 `validateValue:forKey:error:`。`validateValue:forKey:error:` 的默认实现会搜索相应的 key 的 validator method。

### Automatic Validation
KVC 不自动进行 validation。`core data` 在 save value 的时候会自动进行。

### Validation of Scalar Values
原子类型会被 wraped 在 `NSValue` 和 `NSNumber` 中。可以参见以下示例：

```objc
- (BOOL)validateAge:(id *)ioValue error:(NSError * __autoreleasing *)outError {
  if (*ioValue == nil) {
    // Trap this in setNilValueForKey.
    // An alternative might be to create new NSNumber with value 0 here.
    return YES;
  }
  if ([*ioValue floatValue] <= 0.0) {
    if (outError != NULL) {
      NSString *errorString = NSLocalizedStringFromTable(@"Age must be greater than zero", @"Person",@"validation: zero age error");
      NSDictionary *userInfoDict = @{ NSLocalizedDescriptionKey : errorString};
      NSError *error = [[NSError alloc] initWithDomain:PERSON_ERROR_DOMAIN code:PERSON_INVALID_AGE_CODE userInfo:userInfoDict];
      *outError = error;
    }
    return NO;
  } else {
    return YES;
  }
}
```

---
## Ensuring KVC Compliance
为了使一个 property 是认为是 KVC compliant 的话，它必须实现 `valueForKey:` 和 `setValue:forKey:` 要求的 methods。

### Attribute and To-One Relationship Compliance

* Implement a method named `-<key>`, `-is<Key>`, or have an instance variable `<key>` or `_<key>`. Although key names frequently begin with a lowercase letter, KVC also supports key names that begin with an uppercase letter, such as URL.
* If the property is mutable, then it should also implement `-set<Key>:`.
* Your implementation of the `-set<Key>:` method should not perform validation.
* Your class should implement `-validate<Key>:error:` if validation is appropriate for the key.

### Indexed To-Many Relationship Compliance
For indexed to-many relationships, KVC compliance requires that your class:

* Implement a method named `-<key>` that returns an array.
* Or have an array instance variable named `<key>` or `_<key>`.
* Or implement the method `-countOf<Key>` and one or both of `-objectIn<Key>AtIndex:` or `-<key>AtIndexes:`.
* Optionally, you can also implement `-get<Key>:range:` to improve performance.

For a mutable indexed ordered to-many relationship, KVC compliance requires that your class also:

* Implement one or both of the methods `-insertObject:in<Key>AtIndex:` or `-insert<Key>:atIndexes:`.
* Implement one or both of the methods `-removeObjectFrom<Key>AtIndex:` or `-remove<Key>AtIndexes:`.
* Optionally, you can also implement `-replaceObjectIn<Key>AtIndex:withObject:` or `-replace<Key>AtIndexes:with<Key>:` to improve performance.

### Unordered To-Many Relationship Compliance
For unordered to-many relationships, KVC compliance requires that your class:

* Implement a method named `-<key>` that returns a set
* Or have a set instance variable named `<key>` or `_<key>`.
* Or implement the methods `-countOf<Key>`, `-enumeratorOf<Key>`, and `-memberOf<Key>:`.

For a mutable unordered to-many relationship, KVC compliance requires that your class also:

* Implement one or both of the methods `-add<Key>Object:` or `-add<Key>:`.
* Implement one or both of the methods `-add<Key>Object:` or `-add<Key>:`.
* Optionally, you can also implement `-intersect<Key>:` and `-set<Key>:` to improve performance.

---
## Scalar and Structure Support
check `NSValue` and `NSNumber`

---
reference: 
Apple's [document](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html)
