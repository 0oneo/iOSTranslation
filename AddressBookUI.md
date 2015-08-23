本文来自 [Matt](http://nshipster.com/authors/mattt-thompson/) 的 [Address​Book​UI](http://nshipster.com/addressbookui/)

[AddressBookUI](https://developer.apple.com/library/ios/documentation/AddressBookUI/Reference/AddressBookUI_Framework/index.html) 是在用户的通讯录中显示、选择、编辑和创建联系人的 iOS 框架。跟 [Message UI](https://developer.apple.com/library/ios/documentation/MessageUI/Reference/MessageUI_Framework_Reference/index.html) 框架一样，Address Book UI 包含一些可被 present 的 controllers，这些 controllers 提供了统一的接口。

要使用这个框架，添加 `AddressBook.framework` 和 `AddressBookUI.framework` 到你的项目，在 build phase 的 `Link Binary With Libraries` 下。

除去那些 controllers 和 protocals，有一个 Address Book UI 函数非常有用：

`ABCreateStringWithAddressDictionary()` 从 components 返回一个本地化、格式化好的字符串。

函数的第一个参数包含了 address 的 components， 这些 components 的 key 是一些定义好的常量字符串：

* kABPersonAddressStreetKey
* kABPersonAddressCityKey
* kABPersonAddressStateKey
* kABPersonAddressZIPKey
* kABPersonAddressCountryKey
* kABPersonAddressCountryCodeKey

> `kABPersonAddressCountryCodeKey` 是个特别的属性，它决定了使用哪个本地化文件。如果你不确实你的 country code，或者没有提供特定的 data set， `NSLocale` 也许可以帮助到你

```objc
[mutableAddressComponents setValue:[[[NSLocale alloc] initWithLocaleIdentifier:@"en_US"] objectForKey:NSLocaleCountryCode] forKey:(__bridge NSString *)kABPersonAddressCountryCodeKey];
```

第二个参数一个 boolean 标志，`addCountryName`。 当 `YES`，country code 对应的国家名会添加到地址里。这个应该在 country code 已知的情况下使用。

```swift
let addressComponents = [
    kABPersonAddressStreetKey: "70 NW Couch Street",
    kABPersonAddressCityKey: "Portland",
    kABPersonAddressStateKey: "OR",
    kABPersonAddressZIPKey: "97209",
    kABPersonAddressCountryCodeKey: "US"
]

ABCreateStringWithAddressDictionary(addressComponents, true)
```

其他框架都没有提供这个功能。它不是 `NSLocale`，甚至也不是 `Map Kit` 或 `Core Location` 的功能。考虑到苹果这么关心本地化，这样的功能居然是在一个不相关框架的某个不透明的角落。

你可以看到地址在不同的地区 (region) 变化是很大的。例如美国的地址格式：

```text
Street Address
City State ZIP
Country
```

然而日本的地址格式如下：

```text
Postal Code
Prefecture Municipality
Street Address
Country
```