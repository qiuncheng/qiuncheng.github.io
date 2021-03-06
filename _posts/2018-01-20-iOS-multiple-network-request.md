---
layout:     article
title:      "iOS 如何判断多个同时进行的网络请求全部结束"
subtitle:   " \"对于iOS中的多个同时进行的网络请求，大家都很熟悉了，但如果让你在所有的请求结束之后再去刷新UI呢？\""
date:       2018-01-20 21:00:00
key:        1009
tags:
    - iOS
    - Swift
    - 网络请求
---

# 前言
这应该是一种比较常见的情况吧，多个网络请求之后刷新 `UI`，。由于所有的网络请求都是异步的，怎么判断才好呢？
这是笔者在实际开发过程中遇到的情况，简单总结下：
<!--more-->
### 一个一个地执行网络请求
```swift
requestOne {
	///第一个请求结束，执行第二个请求
    requestTwo {
        /// 第二个请求结束，执行第三个请求
    	requestThree {
    		/// to do something.
    	}
    }
}
```
如以上代码所示，在每一个请求结束之后再去执行新的请求。可想而知，本来网络请求就是费时操作，这样一来，更加费时了。而且有系统的多线程可以利用，为啥不用。所以这种方式不建议使用。

### 使用变量标记

```swift
var isCompletion: (Bool, Bool, Bool) = (false, false, false)
func makeRequest() {
    requestOne { [weak self] in
        self?.isCompletion.0 = true
        if self?.isCompletion.0 == true && self?.isCompletion.1 == true && self?.isCompletion.2 == true {
            self?.isCompletion = (false, false, false)
            /// to do something.
        }
    }
        
    requestTwo { [weak self] in
        self?.isCompletion.1 = true
        if self?.isCompletion.0 == true && self?.isCompletion.1 == true && self?.isCompletion.2 == true {
            self?.isCompletion = (false, false, false)
            /// to do something.
        }
    }
        
    requestThree { [weak self] in
        self?.isCompletion.2 = true
        if self?.isCompletion.0 == true && self?.isCompletion.1 == true && self?.isCompletion.2 == true {
            self?.isCompletion = (false, false, false)
            /// to do something.
        }
    }
}
```
以上代码利用了全局元组变量`isCompletion`来标记，当且仅当`isCompletion`里面的三个值全部为`true`的时候, 才表示三个请求全部加载完毕, 这种方式对于请求数目比较少的话还好处理, 一旦数目很多, `isCompletion` 元组中包含的`Bool`值也会增多, 在代码阅读和判断书写上就不会那么便利. 所以对于这种方式，请求数目比较少（小于等于3）推荐使用，请求数目比较多（大于3）不推荐使用。

### 使用完成次数标记
```swift
var remainRequestCount = 0
func makeRequest() {
    remainRequestCount += 1
    requestOne { [weak self] in
        self?.remainRequestCount -= 1
        if self?.remainRequestCount == 0 {
            /// to do something
        }
    }
    
    remainRequestCount += 1
    requestTwo { [weak self] in
        self?.remainRequestCount -= 1
        if self?.remainRequestCount == 0 {
            /// to do something
        }
    }
    
    remainRequestCount += 1
    requestThree { [weak self] in
        self?.remainRequestCount -= 1
        if self?.remainRequestCount == 0 {
            /// to do something
        }
    }
}
```
以上代码使用了变量`remainRequestCount`来统计当前剩余请求的个数. 在每次请求前将至 +1, 请求完成后 -1, 等于0就表示所有的请求已经进行完. 这种方式可以有效解决上面的元组带来的繁琐问题. 同时相对于元组的方式, 请求数目较多时也可以达到简化代码的效果. 所以这种方式在实际开发过程中，也是极为推荐使用的。

### 使用信号量
什么是信号量，对于很多iOS开发者可能这都是一个陌生的词，因为真的不常用。简单点理解，信号量就是一个资源计时器。与 ARC 内存的引用计数相似，可以认为有一个整形变量，当信号量进入时，该整形变量+1，离开时该整形变量-1。当该整形变量为0时就执行接下来的操作。

iOS 中使用信号量与这三个函数有关：
- `DispatchSemaphore(value:_)`: 创建一个信号量；
- `signal()`: 发送一个信号量；
- `wait(wallTimeout:_)`: 等待一个信号量。

```swift
let dispatchGroup = DispatchGroup()
let semaphore = DispatchSemaphore(value: 0)
dispatchGroup.notify(queue: DispatchQueue(label: "com.vsccw.test1")) {
    let value = semaphore.signal()
    sleep(2)
    /// request 1
}

dispatchGroup.notify(queue: DispatchQueue(label: "com.vsccw.test2")) {
    let value = semaphore.signal()
    sleep(2)
    /// request 2
}
_ = semaphore.wait(wallTimeout: .distantFuture)
/// 上面的操作执行完毕就会执行
/// to do something
```

### RxSwift
可以通过使用 `RxSwift` 的 `zip` 操作符来实现所有的网络请求结束后再进行相关的操作。
关于 `zip`：通过一个函数将多个 `Observables` 的元素组合起来，然后将每一个组合的结果发出来。

```swift
Observable.zip(request1, request2) { ($0, $1) }
    .subscribe(onNext: { 
        /// to do something.
    })
    .disposed(by: disposeBag)
```

### Promises
Promises 本是前端的东西，可是它真的好用到`iOS`上也有。  
 > Promise 对象是 JavaScript 的异步操作解决方案，为异步操作提供统一接口。它起到代理作用（proxy），充当异步操作与回调函数之间的中介，使得异步操作具备同步操作的接口。Promise 可以让异步操作写起来，就像在写同步操作的流程，而不必一层层地嵌套回调函数。以上来源自 [[Promise 对象]](https://javascript.ruanyifeng.com/advanced/promise.html)

在这里可以使用`all`操作符：
`all` 会在所有的`Promise`实例执行完毕后，再去执行`then`操作。经验证，`all`里面的所有操作都是异步的。简直完美解决这个问题。
```swift
func requestOne() { }
func requestTwo() { }
func requestThree() { }
all([requestOne(), requestTwo(), requestThree()]).then { result in
  /// to do something.
}
```
以上代码参考：[google/promises](https://github.com/google/promises/blob/master/g3doc/index.md#nested-promises)


# 总结
笔者在上面提供的方法里面，可以说使用普遍度是依次降低的，当然笔者在实际的开发项目中使用的第二种，使用了变量的形式**使用变量标记**的形式，相对以上几种来说，这种方式，更快捷也更易懂；第四种对`RxSwift`比较熟悉的同学还是没什么难度的, 不过不熟悉的同学也可以去了解一下； 第三种使用信号量的方式建议掌握, 即使自己在实际开发中不会用到, 但是这算是`GCD信号量`的基本使用方式之一了；最后一种`Promise`方式是最近*（2018-06-07）*看到Google开源的这个库之后才了解的，也很推荐使用。

# 参考

- [多个网络请求并发执行、顺序执行](https://www.jianshu.com/p/56313152e87e)
- [一些 RxSwift 思考题 - 回答](https://blog.dianqk.org/2017/04/03/rxswift-answer/)
- [基于 Rx 的网络层实践](https://xiaozhuanlan.com/topic/2380179546)
- [Google开源的iOS Promises框架](https://github.com/google/promises)


