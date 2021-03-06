---
layout:     article
title:      "iOS图片拉伸变形小技巧"
subtitle:   "\"图片拉伸，再平常不过，如果你遇到拉伸后变形，那么此文，或许可以给你提供一些思路。\""
date:       2017-12-18 23:30:00
key:        1005
tags:
    - iOS小技巧
    - iOS
---

> 每天进行 UI 开发，不记录点什么，怎么对得起设计师。
<!--more-->
关于 iOS 上的图片拉伸，想必作为一名 iOS 开发者，应该都不陌生吧。尤其是对于使用 Storyboard 来说，简直是太友好了，不用写一行代码，就可以对图片的拉伸进行控制，简直是友好的不要不要的啦。建议对图片拉伸不熟悉的可以看下简书上这篇文章《[iOS中拉伸图片的几种方式](http://www.jianshu.com/p/c9cbbdaa9b02)》，写的比较详细。笔者在这里就不详细介绍的拉伸细节了。不过笔者今天要分享的是一个图片拉伸的变形问题，也是笔者在实际开发中遇到的一个小问题。

## 问题
这里是其原图，我想要在保证左边完整的情况下，左右拉伸中间部分。  
![](https://user-images.githubusercontent.com/9990834/34115482-c12a2800-e450-11e7-9edc-a94c2ef6fc27.png)

- 原图高为 **53**， 宽为 **127**

我期望得到的图片是这样的：
![](https://user-images.githubusercontent.com/9990834/34115447-a50b44ba-e450-11e7-8fd7-6cd9fe26a035.png)

- 上图中 imageView 高度为 **53**， 宽度为屏幕宽度。（得到和期望一致）

在上面我只是左右拉伸中间部分，就达到预期效果。具体代码如下：

```swift
/// 图片的高度为 53， 所以为了保证左边的圆完整，这里也保留左边 53.
imageView1.image = image?.resizableImage(withCapInsets: UIEdgeInsets(top: 0, left: 53, bottom: 0, right: 53), resizingMode: .stretch)
```
但是当 `imageView` 的高度 > 53 的时候, 依然是拉伸中间部分，却得不到预期效果。两种情况下对比截图如下所示：其中 `imageView1` 与 `imageView3` 的宽度相同, `imageView1` 高度为53, `imageView3` 高度为80。
![](https://user-images.githubusercontent.com/9990834/34115739-68096e24-e451-11e7-9e67-ca0e85bd1b53.png)

那么为什么会出现这种现象呢？

## 分析
在进行分析之前，我画了几个草图。这样一来，问题也比较直观些了。
![](https://user-images.githubusercontent.com/9990834/34116335-46a93d48-e453-11e7-8940-62e6b6e54ce4.png)

如上图，既然是左右拉伸中间部分，也就是拉伸上图中的红色方块，如箭头所示，向左右两端拉伸，左端的黄块和右端的蓝块的宽度在拉伸的过程中始终保持不变。拉伸的结果如下图所示：
![](https://user-images.githubusercontent.com/9990834/34116633-276b781e-e454-11e7-8893-7852094d7652.png)
但是如果此时视图的高度大于图片的高度，不管图片的高度拉不拉伸，（不拉伸的话系统默认缩放）都会造成黄块和蓝块的高度大于宽度的。结果如下图所示：
![](https://user-images.githubusercontent.com/9990834/34116824-a77a5e58-e454-11e7-82de-f64a563405cb.png)
所以就造成了上面的现象。简单总结之，在想要保持图片的某一部分宽高比例不变的情况下，拉伸图片。（这里只能怪需求太独特）

那么究竟有什么办法可以解决呢？

## 解决
就上面例子来看，既然要保证左边黄块的宽高比例，也要保证右边蓝块的宽高比例，那么仅仅使用拉伸肯定是不行的。笔者在这里可以提供两种思路：

1. 将图片拉伸后的宽高比例控制与视图的宽高比例保持一致。然后通过缩放来实现。不过显然如果你使用系统提供的 `resizableImage` 方法的话肯定是不行的。因为它只有两个参数呀，一个是你需要保护的区域，另一个是铺开方式。无法做到拉伸到指定 size 的。笔者在这里还翻了一下系统提供的其他接口，貌似都不可以拉伸到指定 size。所以这种方式果断放弃。

2. 先将图片进行宽高等比例 resize 操作，然后再去拉伸。结合上面的例子来看，我们的目的是左右拉伸，那么就需要先将图片的高度 resize 到视图的高度相同，然后再去拉伸。实际的效果如下图所示：

![](https://user-images.githubusercontent.com/9990834/34117792-b830740a-e457-11e7-8c19-819713cb97e3.png)

完美解决，具体的代码如下所示：

```swift
let imageSize = image!.size
let ratio = imageView3.frame.height / imageSize.height
imageView3.image = image?.resize(to: CGSize(width: ratio * imageSize.width, height: imageView3.frame.height)).resizableImage(withCapInsets: UIEdgeInsets(top: 0, left: imageView3.frame.height, bottom: 0, right: 53), resizingMode: .stretch)
```
resize 部分代码：

```swift

extension UIImage {
    
    func resize(to size: CGSize) -> UIImage {
        if size.height == 0 || size.width == 0 {
            return self
        }
        UIGraphicsBeginImageContextWithOptions(size, false, UIScreen.main.scale)
        draw(in: CGRect(x: 0, y: 0, width: size.width, height: size.height))
        guard let image = UIGraphicsGetImageFromCurrentImageContext() else {
            UIGraphicsEndImageContext()
            return self
        }
        UIGraphicsEndImageContext()
        return image
    }
}
```

这里需要注意下：当你 resize 了图片的 size 之后，再去拉伸的时候一定要注意保护区域的改变。就上面这个例子来看，因为其高度增大了，所以在 resize 之后左边保护区域的宽度也相应更新了，只需满足预期保护区域的宽高比即可。

## 总结
图片拉伸本身没什么。对于大部分的需求，都只是拉伸中间区域就可以完美解决问题，不过如果你真的遇到我这种问题，还需要结合实际情况分析下，然后再对图片做适当调整。

## 参考：
[iOS中拉伸图片的几种方式](http://www.jianshu.com/p/c9cbbdaa9b02)

