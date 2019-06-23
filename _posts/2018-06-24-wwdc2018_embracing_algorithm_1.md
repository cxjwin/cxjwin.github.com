---
layout: post
title: "WWDC 2018: Embracing Algorithm (1)"
description: ""
category: WWDC
tags: [Algorithm]
---

[Session 链接](https://developer.apple.com/videos/play/wwdc2018/223/)

## 前言

作为一个 iOS App 开发人员, 经常会听到这样的吐槽, 做一个 App 主要是 UI 布局和动画, 平时基本上都用不到算法, 为啥面试的时候总喜欢考算法?

自己也有过这样的疑惑, 项目中确实很少使用到算法, 一般就是常用的几种设计模式用熟, MVC 和 MVVM 选一个, 然后就开始各种第三方库, 难一点的可能会遇到一些多线程的问题或者组件化开发?

放出 PPT 里面的一张图感受下, 确实很好的总结了 iOS App 开发的精髓.

![App架构](https://user-gold-cdn.xitu.io/2018/6/24/16431eaa1c093e30?w=2394&h=1216&f=png&s=253992)

但是看过这个 Session 之后, 算是得到一些启示, 一个程序员的自我修养最终还是绕不过算法, 代码的优雅和高效始终是我们所追求的, 这无关乎业务和面试.

## 介绍演讲者

觉得有必要介绍下演讲者

WWDC 2015 时候第一次看到 Dave Abrahams 的演讲, 当时讲的是这个 Session ["Protocol-Oriented Programming in Swift"](https://developer.apple.com/videos/play/wwdc2015/408/).

在维基百科上搜了下, [Dave Abrahams](https://en.wikipedia.org/wiki/David_Abrahams_(computer_programmer)) 之前就已经是 C++ STL 的贡献者之一了, 13年加入 Apple 在 Swift 的核心库小组里面担任 TL. 所以这篇演讲也引用了一些 C++ STL 中的哲学思想.

Dave Abrahams 的演讲方式也挺有意思的, 采用一种自编自导自演的方式, 创造了一个苛刻的老学究的角色, 模拟对话, 然后引出演讲的主题. 将本该严谨死板的算法讲出了一些趣味和发人深省的地方.

## 演讲源码

除了 Session 视频, PPT 当然也是要下载的, 得益于 Swift 的开源, PPT 中的代码实现都可以在 GitHub 上找到.

GitHub地址: https://github.com/apple/swift, 当然 master 分支上面是没有的, 切到 swift-4.2-branch-06-11-2018 分支然后在 swift/stdlib/public/core/RangeReplaceableCollection.swift 文件里面可以找到 removeAll 方法, 里面就是 PPT 中讲到的实现.

但是比较奇怪的是在 swift-4.2-branch 分支上面这个实现已经变了, 估计 Apple 的开发人员一直在优化这块的实现, 毕竟4.2目前还不是稳定版本.

## 一个 Bug 引起的思考

Session 以一个图形 App 作为例子, 看一下这个 App:

![App](https://user-gold-cdn.xitu.io/2018/6/24/16431ecfa905ab83?w=1760&h=1250&f=png&s=300402)

然后引出一个 bug, 这个 bug 也是我们新手开发 App 的时候比较容易犯错的一个问题, 对于数组边遍历边删除的问题.

看一下问题:

![p-1](https://user-gold-cdn.xitu.io/2018/6/24/16431eed5a9be67f?w=1734&h=324&f=png&s=47312)

如图, 图中有10个图形, 其中我们选中第8个将其删除, 但是删除的时候 crash 了, why?

看一下问题代码:

```swift
extension Canvas {
    mutating func deleteSelection() {
        for i in 0..<shapes.count {
            if shapes[i].isSelected {
                shapes.remove(at: i)
            }
        }
    }
}
```

![p-2](https://user-gold-cdn.xitu.io/2018/6/24/16431efd3ba44841?w=1774&h=564&f=png&s=126336)

遍历的范围 `0..<shapes.count` 一开始就已经确定了(10个元素), 当遍历到第8个图形的时候, 发现其被选中则进行 remove 操作, 后面两个元素往前补位, 这个时候数组里面只有9个元素了, 所以再按照最开始的范围遍历到第十个元素时组数越界产生 crash.

因为平时使用 Objective-C 比较多, 我们结合 Objective-C 来看看, 我们熟悉的数组遍历方式有:

1.普通 for 循环遍历

```Objective-C
    for (NSInteger i = 0; i < shapes.count; i++) {
        // do something
    }
```

2.for-in 遍历 (这种方式在边遍历边删除的时候会抛异常).

```Objective-C
    for (Shape *shape in shapes) {
        // do something
    }
```

3.block 枚举遍历

```Objective-C
    [shapes enumerateObjectsUsingBlock:^(Shape * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        // do something
    }];
```

以上除了第二种方式会抛异常以外, 1和3这两种都能"混"过去, 为什么是"混", 我们来分析下, 假设这个 bug 中的第9个元素也是被选中的, 那么当遍历到第8个图形的时候, 发现其被选中则进行 remove 操作, 后面两个元素往前补位, 但是此时下标并没有处理, 下一次遍历会直接从第9个元素开始(也就是原先的第10个元素), 从而把原生的第9个元素直接跳过去了, 出现了漏删除的行为.

此类问题我出过一个面试题, 面试题不是很难, 有近一半的面试者出现过边遍历边删除的问题(为啥出这个题, 因为我也是踩过坑的~).

好了回到正题上, 那么原因找到了, 具体怎么个解法呢? Session 中还绕了好几个弯, 我们先来看第一个弯:

```Swift
extension Canvas {
    mutating func deleteSelection() {
        var i = 0
        while i < shapes.count {
            if shapes[i].isSelected {
                shapes.remove(at: i)
            }
            i += 1
        }
    }
}
```

这个改法和普通的 for 循环类似, 只是改成了 while 循环, 问题也比较明显, 假设如果第9个元素也同样被选中, 就会存在漏删的问题, 原因上面已经分析过了.

既然是因为下标没有处理, 那么处理下下标不就可以了? 第二个弯:

```Swift
extension Canvas {
    mutating func deleteSelection() {
        var i = 0
        while i < shapes.count {
            if shapes[i].isSelected {
                shapes.remove(at: i)
            }
            else {
                i += 1
            }
        }
    }
}
```

这个解法是可行的, 还有别的解法么? 逆向思维下, 既然删除一个元素之后, 后面的元素是往前进行补位的, 这样影响的是正序遍历时候的下标. 如果我们采用逆序遍历是不是就不存在这个问题了? 第三个弯:

```Swift
extension Canvas {
    mutating func deleteSelection() {
        for i in (0..<shapes.count).reversed() {
            if shapes[i].isSelected {
                shapes.remove(at: i)
            }
        }
    }
}
```

其实我们一般修改 bug 的话至此就已经完事了, 甚至连逆向思考一下可能都不会去想, 其实这只是刚刚开始.

鼠标移到 remove 方法, 按住 option 键然后点击查看下文档, remove 方法居然是个 O(n) 复杂度的操作. 再加上外层的 while 循环, 整个方法的复杂度有O(n²), 看到这里我也吃了一惊.

后面, 作者给我们科普了下算法的复杂度还有 Mac 上字典中对于算法的定义. 应该也是作为一个引子吧.

这个时候已经不是在解 bug 了, 上升了一个层次 - 代码优化, 先放代码:

```Swift
extension Canvas {
    mutating func deleteSelection() {
        shapes.removeAll(where: { $0.isSelected })
    }
}
```

代码精简了很多, 语义也十分清晰, 这里多了个 removeAll 方法, 这个方法应该是 Swift 4.2 新的方法, 之前的版本并没有找到这个方法. 当然整个过程是值得我们学习的, 对于我们后续封装自己的扩展方法也是很有启发的.

如果你装了 Xcode 10 可以点开 removeAll 的文档看一下, 复杂度为 O(n), 这里是不是勾起了你的好奇心, 从 O(n²) -> O(n) 这个是怎么办到的? 如果是你自己优化了这个解法, 是不是这一整天都是神清气爽的.

```Swift
extension RangeReplaceableCollection where Self: MutableCollection {
  /// Removes all the elements that satisfy the given predicate.
  ///
  /// Use this method to remove every element in a collection that meets
  /// particular criteria. This example removes all the odd values from an
  /// array of numbers:
  ///
  ///     var numbers = [5, 6, 7, 8, 9, 10, 11]
  ///     numbers.removeAll(where: { $0 % 2 == 1 })
  ///     // numbers == [6, 8, 10]
  ///
  /// - Parameter predicate: A closure that takes an element of the
  ///   sequence as its argument and returns a Boolean value indicating
  ///   whether the element should be removed from the collection.
  ///
  /// - Complexity: O(*n*), where *n* is the length of the collection.
  @inlinable
  public mutating func removeAll(
    where predicate: (Element) throws -> Bool
  ) rethrows {
    if var i = try firstIndex(where: predicate) {
      var j = index(after: i)
      while j != endIndex {
        if try !predicate(self[j]) {
          swapAt(i, j)
          formIndex(after: &i)
        }
        formIndex(after: &j)
      }
      removeSubrange(i...)
    }
  }
}
```

上半部分就先讲到这, 下半部分还会用到这个算法, 到时候详细阐述下.

最后放上一句 PPT 中的至理箴言 **"No Raw Loops"**. 怎么做到这一点? 那就是对 Swift 标准库要做到如数家珍.
