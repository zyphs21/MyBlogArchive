---
title: 在Swift扩展里添加"存储属性"
date: '2018-07-17T13:46:33.000Z'
---

# 在Swift扩展里添加"存储属性"

## 由来

最近 [ImageGotcha](https://itunes.apple.com/cn/app/imagegotcha/id1384107130?mt=8) 收到了第一封用户反馈的邮件。  这个用户希望可以有 `Dark Mode`。`ImageGotcha` 只是一个工具类 `App` ，好像也没有什么必要加上这个黑夜模式，不过我还是去想了想如何给应用加上黑夜模式，或者说加上一个换肤的功能。

基本的思路就是 `post` 一个自定义的 `NSNotification`，然后在需要修改颜色的地方监听这个通知然后进行修改。 按照惯例，我还是去 `Github` 上搜搜，看看别人是怎么做的。然后发现一部分人的做法是给现有的 `UIKit` 控件添加扩展属性，然后可以在定义这些控件的时候指定不同模式下的颜色，这的确是一种好方法。那么是如何在 `Swift` 的 `Extension 扩展` 里添加所谓的`"存储属性"`呢？

我们都知道，在 `Swift` 的 `Extension` 里是不能添加`存储属性`的，这里可以类比 `Objective-C`的 `Category 分类`，分类是不能添加实例变量和属性的。

## 疑问

这里就有个问题了，为什么不能添加呢？

> 因为不管是 `Swift` 的 `Extension` 还是 `Objective-C` 的 `Category` 都不能改变原有的类或者结构体的内存结构，在实例化这些类的时候，内存结构是确定的，而添加属性或者实例变量需要内存空间，会改变原有的内存结构。

## 利用关联对象

在 `Objective-C` 中我们常常用运行时 `Associated Object 关联对象` 来给 `Category` 添加属性，而在 `Swift` 里，我们同样可以利用关联对象在 `Extension` 中添加计算属性，以达到所谓的`存储属性`的效果。

```swift
struct AssociatedKeys {
    static var testNameKey: String = "testNameKey"
}

extension UIView {
    public var testName: String? {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.testNameKey) as? String
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.testNameKey, newValue, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
}
```

`&AssociatedKeys.testNameKey`: `&` 操作符是取出地址作为 `UnsafeRawPointer` 参数传入。 `.OBJC_ASSOCIATION_RETAIN_NONATOMIC`: 是一个 `objc_AssociationPolicy` 枚举，它有以下几种选择\(从字面意思可以猜测是与Objective-C中的属性修饰符相关\)：

```text
public enum objc_AssociationPolicy : UInt {
    case OBJC_ASSOCIATION_ASSIGN
    case OBJC_ASSOCIATION_RETAIN_NONATOMIC
    case OBJC_ASSOCIATION_COPY_NONATOMIC
    case OBJC_ASSOCIATION_RETAIN
    case OBJC_ASSOCIATION_COPY
}
```

我们可以测试一下：

```text
var testString = "test"

let view = UIView()

view.testName = testString
print(view.testName) // 输出 Optional("test")

testString.append("change")
print(view.testName) // 输出 Optional("test")

view.testName = "testChange"
print(view.testName) // 输出 Optional("testChange")
```

