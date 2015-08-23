本文来自 [Chris Eidhof](https://twitter.com/chriseidhof) 的 [Lighter View Controllers](http://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)

View controllers 通常是 iOS 项目中最大的文件，通常也包含很多多余的代码。View controller 也几乎是代码中重用性最差的部分。我们将看看一些缩减 view controllers ，提高他们重用性，移到代码到更合适地方的技术。

GitHub 上有的 [example project](https://github.com/objcio/issue-1-lighter-view-controllers)

### 将 Data Soure 和其他 Protocols 分开
一个缩减 view controller 很有用的技术是将 `UITableViewDataSource` 的代码移到一个它单独自己的类中。如果你多次这样做，你会看到模式，并为这创建可重用的类。

例如，在我们的项目中，有一个类 `PhotosViewController` 有以下方法:

```objc
# pragma mark Pragma 

- (Photo*)photoAtIndexPath:(NSIndexPath*)indexPath {
    return photos[(NSUInteger)indexPath.row];
}

- (NSInteger)tableView:(UITableView*)tableView 
 numberOfRowsInSection:(NSInteger)section {
    return photos.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView 
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    PhotoCell* cell = [tableView dequeueReusableCellWithIdentifier:PhotoCellIdentifier 
                                                      forIndexPath:indexPath];
    Photo* photo = [self photoAtIndexPath:indexPath];
    cell.label.text = photo.name;
    return cell;
}
```

许多代码是跟数组相关的，少量的代码是 view controller 管理的照片相关的。所以让我们把数组相关的代码抽出来单独作为一个类。我们使用一个 block 来配置 cell，但是使用 delegate 是同样的，根据你自己的情况和品位。

```objc
@implementation ArrayDataSource

- (id)itemAtIndexPath:(NSIndexPath*)indexPath {
    return items[(NSUInteger)indexPath.row];
}

- (NSInteger)tableView:(UITableView*)tableView 
 numberOfRowsInSection:(NSInteger)section {
    return items.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView 
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    id cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifier
                                              forIndexPath:indexPath];
    id item = [self itemAtIndexPath:indexPath];
    configureCellBlock(cell,item);
    return cell;
}

@end
```

view controller 中的三个方法可以移除了，并通过创建上述类的一个实例，设置为 tableview 的 data source 来替换掉。

```objc
void (^configureCell)(PhotoCell*, Photo*) = ^(PhotoCell* cell, Photo* photo) {
   cell.label.text = photo.name;
};
photosArrayDataSource = [[ArrayDataSource alloc] initWithItems:photos
                                                cellIdentifier:PhotoCellIdentifier
                                            configureCellBlock:configureCell];
self.tableView.dataSource = photosArrayDataSource;
```

现在你不需要担心映射一个 indexpath 到数组的某个位置，每次你通过 tableview 展示一个数组的时候就可以复用上述代码。你也可以在另外的方法——`tableView:commitEditingStyle:forRowAtIndexPath:`，并在多个 table view controller 中共享。

比较好的是我们可以单独测试上述代码了，以后也不用担心重写上述代码了。如果你使用数组以外的其他东西，同样的原则还是使用的。

在我们今年工作的一个应用上，我们重度的使用 `Core Data`。我们创建一个类似的类，不同通过数组来支持的，而是一个 fetched results controller。它实现了更新动画，dong section header，删除的所有逻辑。你可以通过 a fetch request 和一个配置 cell 的 block 来创建一个这样的对象。

同样，这种方式也使用于其他的 prorocals。一个很显然的候选者是 `UICollectionViewDataSource`。如果在未来的某个时候你想用 `UICollectionView` 替换 `UITableView`， 你几乎不需要改动 view controller 中的任何代码。你甚至可以使用你的 data source 同时实现两个协议。

### Move Domain Logic into the Model
以下是一个 view controller 的样例代码：

```objc
- (void)loadPriorities {
  NSDate* now = [NSDate date];
  NSString* formatString = @"startDate <= %@ AND endDate >= %@";
  NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
  NSSet* priorities = [self.user.priorities filteredSetUsingPredicate:predicate];
  self.priorities = [priorities allObjects];
}
```

然后，如果将上述代码移到 `User` 类的一个 category 中的话，代码会更清晰，`View Controller.m`：

```objc
- (void)loadPriorities {
  self.priorities = [self.user currentPriorities];
}
```

`User+Extensions.m`:

```objc
- (NSArray*)currentPriorities {
  NSDate* now = [NSDate date];
  NSString* formatString = @"startDate <= %@ AND endDate >= %@";
  NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
  return [[self.priorities filteredSetUsingPredicate:predicate] allObjects];
}
```

一些代码不能轻易的移到一个 `model` 对象中，但是仍然是跟 `model` 相关的，对于这样的代码，我们可以创建一个 `Store`:

### Creating the Store Class
在我们的样例应用中，我们有些代码是从文件中加载数据并解析它们，这些代码是在 view controller 中:

```objc
- (void)readArchive {
    NSBundle* bundle = [NSBundle bundleForClass:[self class]];
    NSURL *archiveURL = [bundle URLForResource:@"photodata"
                                 withExtension:@"bin"];
    NSAssert(archiveURL != nil, @"Unable to find archive in bundle.");
    NSData *data = [NSData dataWithContentsOfURL:archiveURL
                                         options:0
                                           error:NULL];
    NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
    _users = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"users"];
    _photos = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"photos"];
    [unarchiver finishDecoding];
}
```

view controller 不应该知道这些。我们可以创建一个干这事的 **Store** 对象。通过分离出上述代码我们可以复用上述代码，并单独测试。**Store** 对象负责数据的加载、缓存、设置 database stack。**Store** 经常被称为 **service layer** 或一个 **repository**.

### Move Web Service Logic to the Model Layer
这个话题跟上述的主题很像：不要在 view controller 中做 web service logic。而是，单独用一个类来做这样的事。你的 view controller 可以通过调用这些方法。比较好的地方是你可以在这个类中做 cache 或者 error handling。

### Move View Code into the View Layer
构建复杂的 view hierarchy 不应该在 view controller 中进行。要么使用 interface builder，要么封装这些 view 到一个单独 `UIView` 子类中。例如：你实现你自己的 data picker control，在一个单独的 `DatePickerView` 来实现比较合理。这样可以增加复用性和使代码简洁。

如果你喜欢 insterface builder，你可以使用他们。一些人以为只能使用 interface builder 来处理 view controller ，其实不是，你也可以创建自定义的 view，然后 load 进来就可以了。

### Communication
View controllers 做的其他事中的一个是与其他的 view controller、mode、view 通讯。尽管这是 controller 应该做的，但是我们也可以通过尽量少的代码来实现同样的目的。

这个会在未来的 issue 中讨论。

### Conclusion
我们已经看了这么技术来创建小的 view controller。我们不是必须在任何可能的地方使用这些技术。但我们有一个很简单的诉求：to write maintainable code. 知道这些模式，我们有更大的可能发现臃肿的 view controller，并使它们更简单清晰。