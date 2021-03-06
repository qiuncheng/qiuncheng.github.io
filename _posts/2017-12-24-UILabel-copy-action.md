---
layout:     article
title:      "UILabel 添加复制 iOS小技巧"
subtitle:   "\"呵，UILabel也能添加复制功能？还真的可以。\""
date:       2017-12-24 21:54:00
key:        1007
tags:
    - iOS小技巧
    - iOS
---

> `UITextView`, `UITextField` 都自带长按复制功能，可是凭啥 `UILabel` 没有啊？

## 问题
`UILabel`, `UITextView`, `UITextField` 这三个控件是 iOS 开发中比较常用的文本控件，不过三者也有些许区别：
<!--more-->
  
- `UILabel`: 显示的文本无法编辑，不过可以换行显示
- `UITextField`: 可以编辑文本，也可以与键盘有点交互，不过只能在一行显示
- `UITextView`: 可以编辑文本，可以换行显示文本，还可以与键盘交互

对于以上三者来说，可以编辑的话就可以影响复制功能，所以 `UITextField` 与 `UITextView` 默认就可以响应复制的。不过 `UILabel` 默认则是无法响应复制的。 今天笔者暂且不谈 `UITextView`, `UITextField` , 仅仅通过自定义的方式实现给 UILabel 添加复制功能。

## 分析
对于大多数 iOS 开发的同学来说，`UIMenuController` 都不会陌生，它会在指定位置显示几个 `action`, 这里的 `action` 我们可以自定义, 也可以使用系统默认。`UIMenuController` 的具体细节部分这里不再多介绍，对于不是很了解的同学，推荐这篇文章: [iOS --苹果自带的UIMenuController功能扩展](https://www.jianshu.com/p/ddd59867909a)。里面讲的还比较细。 具体的显示如下面所示：
![](https://user-images.githubusercontent.com/9990834/34327285-58567acc-e8fb-11e7-8f19-e8ecb17ffc31.jpg)

我们来拆分一下要实现这个复制功能的具体步骤： 

1. 长按，弹出 `UIMenuController`， 所以要有长按手势；
2. 在弹出的 `UIMenuController` 要有 "拷贝" 或者 "复制" 或者 "Copy" 这个选项；
3. 点击了 "拷贝" 或者 "复制" 或者 "Copy" 这个选项后要完成复制操作；
4. `UIMenuController` 消失，然后 Over.

## 解决
针对上面提出的四点，我们按照步骤一一解决。

- 使用长按手势，并弹出 `UIMenuController`

```swift
@objc private func showMenuGestureAction(_ gesture: UILongPressGestureRecognizer) {
    _ = becomeFirstResponder()
    if !menuController.isMenuVisible {
        UIMenuController.shared.setMenuVisible(true, animated: true)
    }
}

/// 这里必须重写 canBecomeFirstResponder，否则不会弹出
override var canBecomeFirstResponder: Bool {
    return true
}

/// 执行 copyAction 方法的 action
override func canPerformAction(_ action: Selector, withSender sender: Any?) -> Bool {
    return action == #selector(copyAction(_:))
}
```
- 添加 `复制` 选项，这里仅仅作为复制举例，当然也可以有其他选项

```swift
let copyItem = UIMenuItem(title: "复制", action: #selector(copyAction(_:)))
UIMenuController.shared.menuItems = [copyItem]
```
-  执行复制操作

```swift
@objc func copyAction(_ item: UIMenuItem) {
	 /// 将内容复制到粘贴板，其他应用也可以获取到该内容
    UIPasteboard.general.string = text
    UIMenuController.shared.menuItems = nil
}
```
- `UIMenuController` 消失: 自动消失
- 补充： `isUserInteractionEnabled = true` 别忘了将该 `UILabel` 的交互事件属性设置为 `true`， 否则长按是没有效果的。

## 总结
实现 `UILabel` 的复制功能只是一个小例子，你还可以通过这种方式给 `UILabel` 添加更多可编辑的选项，以执行更多的操作。当然如果使用 `UITextView` 也完全自带上复制功能，而且还是系统管理的，但是考虑到 `UITextView` 的性能的话，就不想多说什么了。

本文涉及到的完整的代码如下：

```swift
import UIKit

class CopyLabel: UILabel {

    override init(frame: CGRect) {
        super.init(frame: frame)
        commonInit()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    private func commonInit() {
        isUserInteractionEnabled = true
        addGestureRecognizer(UILongPressGestureRecognizer(target: self, action: #selector(showMenuGesture(_:))))
    }
    
    @objc private func showMenuGesture(_ gesture: UILongPressGestureRecognizer) {
        _ = becomeFirstResponder()
        let menuController = UIMenuController.shared
        if !menuController.isMenuVisible {
            let copyItem = UIMenuItem(title: "复制", action: #selector(copyAction(_:)))
            menuController.menuItems = [copyItem]
            menuController.setTargetRect(bounds, in: self)
            menuController.setMenuVisible(true, animated: true)
        }
    }
    
    override var canBecomeFirstResponder: Bool {
        return true
    }
    
    override func canPerformAction(_ action: Selector, withSender sender: Any?) -> Bool {
        return action == #selector(copyAction(_:))
    }
    
    @objc func copyAction(_ item: UIMenuItem) {
        UIPasteboard.general.string = text
        UIMenuController.shared.menuItems = nil
    }
}
```

## 不足

由于本文只是很简单实现 `UILabel` 的复制功能，其实完整的复制还应该包含**文本选择**，因为用户可能不想复制整个文本，而是想复制部分文本，这个是本文没有涉及到的，不过将会在另一篇文章中涉及。

如果你有更好的实现方案，也请及时告诉我；如果你发现了本文中存在问题，也请及时告诉我。本文源于开发中的笔记，仅仅作为笔者日常记录。☺️


## 参考
- [iOS --苹果自带的UIMenuController功能扩展](https://www.jianshu.com/p/ddd59867909a)
- [简单实现UIlabel可复制功能](https://www.jianshu.com/p/b0eb1e88a928)

