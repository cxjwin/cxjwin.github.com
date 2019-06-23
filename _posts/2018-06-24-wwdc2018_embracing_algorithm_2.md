---
layout: post
title: "WWDC 2018: Embracing Algorithms(2)"
description: ""
category: WWDC
tags: [Algorithm]
---

此文承接[上一篇](https://juejin.im/post/5b2f976bf265da597b0f920f), [Session 链接](https://developer.apple.com/videos/play/wwdc2018/223/)

## 算法分析

首先我们按 PPT 拆解下代码:

```Swift
extension MutableCollection {
    /// Moves all elements satisfying `isSuffixElement` into a suffix of the collection,
    /// returning the start position of the resulting suffix.
    ///
    /// - Complexity: O(n) where n is the number of elements.
    mutating func halfStablePartition(isSuffixElement: (Element) -> Bool) -> Index {
        guard var i = firstIndex(where: isSuffixElement) else { return endIndex }
        var j = index(after: i)
        while j != endIndex {
            if !isSuffixElement(self[j]) { swapAt(i, j); formIndex(after: &i) }
            formIndex(after: &j)
        }
        return i
    }
}

extension MutableCollection where Self : RangeReplaceableCollection {
    /// Removes all elements satisfying `shouldRemove`.
    ///   ...
    /// - Complexity: O(n) where n is the number of elements.
    mutating func removeAll(where shouldRemove: (Element)->Bool) {
        let suffixStart = halfStablePartition(isSuffixElement: shouldRemove)
        removeSubrange(suffixStart...)
    }
}
```

我们找一个例子走一遍过程(选出所有负数并删除):

```Swift
    [1, 2, -1, -2, 3, 4, -3, -4, -5, 5] // 初始

    [1, 2, -1, -2, 3, 4, -3, -4, -5, 5] // i == 2, j == 3

    [1, 2, 3, -2, -1, 4, -3, -4, -5, 5] // i == 2, j == 4

    [1, 2, 3, -2, -1, 4, -3, -4, -5, 5] // i == 3, j == 5

    [1, 2, 3, 4, -1, -2, -3, -4, -5, 5] // i == 4, j == 6

    [1, 2, 3, 4, -1, -2, -3, -4, -5, 5] // i == 4, j == 7

    [1, 2, 3, 4, -1, -2, -3, -4, -5, 5] // i == 4, j == 8

    [1, 2, 3, 4, -1, -2, -3, -4, -5, 5] // i == 4, j == 9

    [1, 2, 3, 4, 5, -2, -3, -4, -5, -1] // i == 5, j == endIndex

    [1, 2, 3, 4, 5] // 删除右边部分
```

上述算法中 i 和 j 都是顺序遍历, 通常情况下 j 会比 i 前进的快些(j 每次都会自增), 总的复杂度为 O(n).

`halfStablePartition` 方法的主要作用是按 `isSuffixElement` 条件将数组分为左右两个部分, 左边是不满足条件的部分, 右边是满足条件的部分, 并返回右边部分的起始下标.

![图1](https://user-gold-cdn.xitu.io/2018/7/5/164688e4ecce2500?w=1656&h=360&f=png&s=31138)

然后通过 `removeSubrange` 方法将右边部分全部删除, 这样就实现了 `removeAll`.

这个算法的巧妙之处在于, 左边部分不影响在原有数组中的相对顺序, 右边部分虽然顺序有变但是因为随后会被删除, 所以不受影响.

到这里大家可能会觉得做些解法都有点绕, 直接用额外的数组存一下, 或者使用 `filter` 方法是不是更直接些? 但是这两种方法会用到额外的存储空间.

## 举一反三

正当作者准备背起书包回家的时候, "老学究"问他"难道项目中没有类似问题了么?" 其实对比我们自己往往也是这样的, 解决完一个 bug 就大功告成了, 至于还有其他地方需要优化, 有空再说吧.

然后作者放下书包开始继续查看代码. 代码写习惯了, 相似的错误可能会被带到项目的各个角落, 下面感受下类似错误的地方:

![图2](https://user-gold-cdn.xitu.io/2018/7/5/1646891b671e49bd?w=2242&h=1230&f=png&s=522524)

我先看第一个, 将选中的图形都移到前面, 并保持相对顺序不变:

```Swift
extension Canvas {
    mutating func bringToFront() {
        var i = 0, j = 0
        while i < shapes.count {
            if shapes[i].isSelected {
                let selected = shapes.remove(at: i)
                i += 1
                shapes.insert(selected, at: j)
                j += 1
            }
        }
    }
}
```

查一下文档 `remove` 和 `insert` 都是 O(n) 复杂度的操作, 合起来还是 O(n), 再加上 while 循环, 又是一个 O(n²) 复杂度的算法.

那么看一下优化后的代码:

```Swift
extension Canvas {
    /// Moves the selected shapes to the front, maintaining their relative order.
    mutating func bringToFront() {
        shapes.stablePartition(isSuffixElement: { !$0.isSelected }) 
    }
}
```

其中 `stablePartition` 的实现可以在这个[链接](https://github.com/apple/swift/blob/master/test/Prototypes/Algorithms.swift)中找到, 我们留到最后进行分析.

这个算法的含义是按条件 `isSuffixElement` 进行分类, 满足条件的放在后面, 不满足条件的放在前面, 算法复杂度为O(n log n).

![图三](https://user-gold-cdn.xitu.io/2018/7/5/1646895dbebcb522?w=1612&h=214&f=png&s=41746)
![图四](https://user-gold-cdn.xitu.io/2018/7/5/1646895e7521b6c0?w=1626&h=226&f=png&s=42147)

既然是 `bringToFront` 那么就是没有选中的放后面, 所以条件就是 `!$0.isSelected`.

同理我们可以实现一个 `sendToBack` 方法, 即选中的放后面, 所以条件就是 `$0.isSelected`:

```Swift
extension Canvas {
    /// Moves the selected shapes to the back, maintaining their relative order.
    mutating func sendToBack() {
        shapes.stablePartition(isSuffixElement: { $0.isSelected })
    }
}
```

我们来看一下另一个方法 `bringForward`, 这个方法的作用是将选中的所有元素统一插入到选中的第一个元素的前一个位置并保持相对顺序不变.

调用方法之前:

![图五](https://user-gold-cdn.xitu.io/2018/7/5/16468987195fae94?w=1592&h=212&f=png&s=41227)

调用方法之后:

![图六](https://user-gold-cdn.xitu.io/2018/7/5/1646898bedddd464?w=1586&h=210&f=png&s=40587)

我们还是先看一下修改前的代码:

```Swift
extension Canvas {
    mutating func bringForward() {
        for i in shapes.indices where shapes[i].isSelected {
            if i == 0 { return }
            var insertionPoint = i - 1
            for j in i..<shapes.count where shapes[j].isSelected {
                let x = shapes.remove(at: j)
                shapes.insert(x, at: insertionPoint)
                insertionPoint += 1
            }
            return
        }
    }
}
```

这里虽然是两层 for 循环, 但是这两个循环是前后衔接的关系, 所以还是 O(n) 的复杂度, 总的复杂度还是 O(n²).

到这里你或许会问, 那这个算法和 `stablePartition` 方法有什么联系呢? 这里作者给了我们一个提示, 如果我们把选中的第一个元素的前一个位置作为分割点把数组分为左右两个子数组, 然后对右边的子数组做 `stablePartition` 是不是就可以了? 那么这个算法的复杂度就可以优化到 O(n log n) 了. 

分割示意图:

![图7](https://user-gold-cdn.xitu.io/2018/7/5/164689a4f61937b1?w=1576&h=286&f=png&s=43643)

修改后的代码:

```Swift
extension Canvas {
    mutating func bringForward() {
        if let i = shapes.firstIndex(where: { $0.isSelected }) {
            if i == 0 { return }
            let predecessor = i - 1
            shapes[predecessor...].stablePartition(isSuffixElement: { !$0.isSelected })
        }
    }
}
```

这里我对解题思路有一个反思. 作者是怎么一步一步联想到这些解题步骤的呢? 难道是仅仅是他自己设计了这个演讲的原因么?

- **"No Raw Loops"**, 不是优先想到直接用循环去解决这些问题, 而是思考下标准库是否已经提供了类似的解决方案?
- 对标准库要熟稔于心, 这样才能够第一时间想到 `ArraySlice` 和 `stablePartition` 这些数据结构和算法
- 有一颗 **Clean Code** 的心, 不断的完善自己的代码

算法优化暂告一段落, 作者做了一下延伸, 我们怎么去测试我们的代码? 难道是要在 **Canvas App** 上手动创建一堆图形, 然后手动选择图形, 点击对应的操作按钮肉眼看一下效果么? 其实这个正是我们开发 App 的时候最常用且最原始的 debug 方式, 得益于 Xcode 模拟器超快的启动速度, 所以很多开发人员直接修改代码, run 起来看一下效果, 不行就改一下再 run, 或者加一些 log, 或者断点调试下. 作为 App 开发人员很少会去思考对自己的算法做单元测试. 

对自己代码做单元测试的这个习惯我是后面重构遗留代码的时候才养成的, 再后来开始做 SDK 的相关开发, 更加意识到单元测试的重要性.

既然上述写法并不利于单测, 那么怎么去修改呢?

1. 采用 UI 测试替代单元测试, 通过 Mock 数据配合宿主 App 把上述手动流程自动化, 这样可以减少手工操作
2. 改写上述算法, 将其变的可单元测试. 那么怎么改写? 主要原则就是减少耦合(这里就是不依赖于 `Canvas` 这个类) 

从代码的通用性和复用性来讲第二种方式比较好, 这里作者就是朝这个方向去改写代码的.

首先我们想到的是, 既然不依赖于 `Canvas` 这个类, 而且这个算法的整个功能其实是对 `Array` 的操作, 那么是不是可以抽取到 `Array` 的 `extension` 里面去呢? 我们看一下修改后的代码:

```Swift
extension Array where Element == Shape {
    mutating func bringForward() {
        if let i = firstIndex(where: { $0.isSelected }) {
            if i == 0 { return }
            let predecessor = i - 1
            self[predecessor...].stablePartition(isSuffixElement: { !$0.isSelected })
        }
    }
}
```

但是你会发现, 虽然做了抽取, 但是这个 `extension` 依然依赖于 `Shape` 类, 解耦的还不彻底, 所以进行第二次修改:

```Swift
extension Array {
    mutating func bringForward(elementsSatisfying predicate: (Element) -> Bool) {
        if let i = firstIndex(where: predicate) {
            if i == 0 { return }
            let predecessor = i - 1
            self[predecessor...].stablePartition(isSuffixElement: { !predicate($0) })
        }
    }
}
```

这里修改的地方涉及到两个:

1. `where Element == Shape` 中去除对于 `Shape` 类的依赖
2. `$0.isSelected` 中将判断条件由外面传参进来(因为 `isSelected` 是 `Shape` 类特有的), 使算法更通用

既然说到了更为通用, 那么这个算法仅仅只适用于 `Array` 么? 是不是 `MutableCollection` 都适用呢? 想想挺有道理, 于是修改代码变成 `extension MutableCollection` 试试, 但是编辑器直接报错了.

![图八](https://user-gold-cdn.xitu.io/2018/7/5/16468a132453f879?w=2218&h=624&f=png&s=187340)

因为 `MutableCollection` 的 `index` 并非是 `Int` 类型的, 不能直接和 0 比较, 或者进行减 1 操作. 第一直觉是改成这样 `extension MutableCollection where Index == Int`, 作者提醒 "Don't do this.". 这样又算法进行特殊化了, 变的不够通用了.

![图九](https://user-gold-cdn.xitu.io/2018/7/5/16468a21e21d47be?w=2234&h=820&f=png&s=221912)

其实如果是我的话, 修改到 `extension Array` 已经觉得可以了, 已经足够通用且可单元测试, 毕竟这个算法在 App 中也是给 `Array` 使用的. 

## Building Towers Of Abstraction

"老学究"几个直击灵魂的提问, 使人有更进一步的想法. 如果我们不纠结于 **"和 0 比较, 进行减 1 操作"** 等细节问题, 将问题进一步抽象化, 思考下这两行代码的作用是什么呢? 选中的第一个元素的前一个位置 -- `indexBeforeFirst`. 那么抽象后的代码:

```Swift
extension MutableCollection {
    mutating func bringForward(elementsSatisfying predicate: (Element) -> Bool) {
        if let predecessor = indexBeforeFirst(where: predicate) {
            self[predecessor...].stablePartition(isSuffixElement: { !predicate($0) })
        }
    }
}
```

然后再来具体看下 `indexBeforeFirst` 的实现:

```Swift
extension Collection {
    func indexBeforeFirst(where predicate: (Element) -> Bool) -> Index? {
        return indices.first {
            let successor = index(after: $0)
            return successor != endIndex && predicate(self[successor])
        }
    }
}
```

适当的抽象能够简化问题, 也能够将问题拆解然后进行聚焦. 

![图十](https://user-gold-cdn.xitu.io/2018/7/5/16468a3f52ad4ae2?w=1942&h=358&f=png&s=86341)

最后要加上必要的**文档**, 完美. 你会问自己给自己写的接口也需要文档么? 那么回去看一下半年前写过的超过100行的没有注释的一段代码, 还记得是干啥的么? 清晰的文档, 于人于己都是方便, 特别在大厂你的代码后续肯定由别人一起维护, 为了减少 **WTF** 的数量, 建议还是写上 **^.^** .

![图十一](https://user-gold-cdn.xitu.io/2018/7/5/16468a5ec32d9171?w=1986&h=730&f=png&s=164940)

## 善始善终

整个优化工作并没有完成, 作者放出了最后一段待优化的代码, 这段代码的作用的是将选中的元素聚焦于选择的位置:

```Swift
extension Canvas {
    mutating func gatherSelected(at target: Int) {
        var buffer: [Shape] = []
        var insertionPoint = target
        var i = 0
        while i < insertionPoint {
            if shapes[i].isSelected {
                let x = shapes.remove(at: i)
                buffer.append(x)
                insertionPoint -= 1
            }
            else {
                i += 1
            }
        }
        while i < shapes.count {
            if shapes[i].isSelected {
                let x = shapes.remove(at: i)
                buffer.append(x)
            }
            else {
                i += 1
            }
        }
        shapes.insert(contentsOf: buffer, at: insertionPoint)
    }
}
```

![图十二](https://user-gold-cdn.xitu.io/2018/7/5/16468ace5a69070e?w=1604&h=282&f=png&s=48052)

![图十三](https://user-gold-cdn.xitu.io/2018/7/5/16468ad2e0d5b96d?w=1574&h=276&f=png&s=46909)

受前面 `bringForward` 方法的启发, 我们在选择的位置处将数组分为左右两个部分, 左边部分将选中元素后置, 右边部分将选中元素前置, 这样总的算法复杂度还是 O(n log n):

```Swift
extension MutableCollection {
    /// Gathers elements satisfying `predicate` at `target`, preserving their relative order. ///
    /// - Complexity: O(n log n) where n is the number of elements.
    mutating func gather(at target: Index, allSatisfying predicate: (Element)->Bool) {
        self[..<target].stablePartition(isSuffixElement: predicate)
        self[target...].stablePartition(isSuffixElement: { !predicate($0) })
    }
}

extension Canvas {
    mutating func gatherSelected(at target: Int) {
        shapes.gather(at: target) { $0.isSelected }
    }
}
```

![图十四](https://user-gold-cdn.xitu.io/2018/7/5/16468aeb448879c7?w=1612&h=282&f=png&s=48143)

![图十五](https://user-gold-cdn.xitu.io/2018/7/5/16468aec280a2b58?w=1616&h=286&f=png&s=48114)

## 算法分析2

最后我们来分析下 `stablePartition` 算法:

```Swift
extension MutableCollection {
    /// Moves all elements satisfying `isSuffixElement` into a suffix of the
    /// collection, preserving their relative order, and returns the start of the
    /// resulting suffix.
    ///
    /// - Complexity: O(n) where n is the number of elements.
    /// - Precondition: `n == self.count`
    mutating func stablePartition(count n: Int, isSuffixElement: (Element) -> Bool) -> Index {
        if n == 0 { return startIndex }
        if n == 1 { return isSuffixElement(self[startIndex]) ? startIndex : endIndex }
        let h = n / 2, i = index(startIndex, offsetBy: h)
        let j = try self[..<i].stablePartition(count: h, isSuffixElement: isSuffixElement)
        let k = try self[i...].stablePartition(count: n - h, isSuffixElement: isSuffixElement)
        return self[j..<k].rotate(shiftingToStart: i)
    }
}
```

这里用到了递归+旋转的方式. 

用例子来看一下:

```Swift
    7, 6, -7, -6, 5, 4, -5, -4, -3, 3, 2, -2, -1, 1

                           i
    7, 6, -7, -6, 5, 4, -5 | -4, -3, 3, 2, -2, -1, 1

               j            i         k
    7, 6, 5, 4 | -7, -6, -5 | 3, 2, 1 | -4, -3, -2, -1

               |      rotate          |
    7, 6, 5, 4 | 3, 2, 1 | -7, -6, -5 | -4, -3, -2, -1

    7, 6, 5, 4, 3, 2, 1, -7, -6, -5, -4, -3, -2, -1
```
