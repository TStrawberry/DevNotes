# 合并 SwiftUI 的视图

# 为什么？

项目中经常有给多个view之间加一个分割线的需求，但是最后一个 view 后面不需要加。通常的做法是通过 ForEach 遍历数据和索引，判断如果是最后一个元素，就不加。简单直接，但是不够优美。
因此我希望提供一个抽象度更高，且能够被复用的方案。

# 实现

```swift
func Joined<Content: View, Separator: View>(
    @ViewBuilder _ content: () -> Content,
    @ViewBuilder separator: @escaping () -> Separator
) -> some View {
    var shouldInsert: Bool = false
    return ForEach(subviews: content()) { subview in
        if shouldInsert {
            separator()
        }
        subview
        let _ = shouldInsert = true
    }
}
```

`Joined` 这个命名来自于 [String] 的 `func joined(separator: String = "") -> String` 方法。
这个实现将 `仅仅在两个元素之间添加` 这一核心需求抽象，适用在任意的 SwiftUI 代码中。

---

# Join Views in SwiftUI

# Why?

In projects, we often need to add a separator between multiple views, but not after the last one. The usual approach is to iterate over data and indices with ForEach, then skip the separator if it is the last element. This is simple and direct, but not particularly elegant.
Therefore, I wanted a more abstract, reusable solution.

# Implementation

```swift
func Joined<Content: View, Separator: View>(
    @ViewBuilder _ content: () -> Content,
    @ViewBuilder separator: @escaping () -> Separator
) -> some View {
    var shouldInsert: Bool = false
    return ForEach(subviews: content()) { subview in
        if shouldInsert {
            separator()
        }
        subview
        let _ = shouldInsert = true
    }
}
```

The name `Joined` comes from [String]'s `func joined(separator: String = "") -> String` method.
This implementation abstracts the core requirement of "inserting something only between two elements," and can be used anywhere in SwiftUI.
