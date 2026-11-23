# 优化版的 SwiftUI onAppear 和 onDisappear 封装

> 本文假设读者已经熟练使用 SwiftUI，并对 @State, Preference 等机制有深入理解。

## 为什么

在我现在的项目中，SwiftUI 的 onAppear 和 onDisappear 的行为在两个方面不符合实际业务需求：

1. 并不保证时序，即后一个页面的 onAppear 会先于 前一个页面的 onDisappear 调用
2. 在新的 sheet/fullScreenCover 被present出的场景，老页面的 onDisappear 不会调用，但 老页面 其实已经对用户不可见的了

本文将讨论如何通过合理的架构设计，封装一套适配以上需求的新 API： 暂且叫做 **onAppearAction** 和 **onDisappearAction**。名字随意，本文只讨论核心机制。

## 架构实现

要解决以上问题，最核心的机制是要 **基于 SwiftUI 实现监听各种页面级的跳转**，包括

1. 在 NavigationStack 中 push/pop 页面
2. 在 TabView 中 切 tab
3. present 出新的 sheet/fullScreenCover


#### 1. 监听在 NavigationStack 中 push/pop 页面

在 [一种在 NavigationStack 进行依赖注入的架构设计](./07-An-architecture-design-of-dependency-injection-on-NavigationStack.md) 一文已经涉及相关的讨论，简单总结如下：

**为每一个页面绑定与之关联的对象，并通过 Preference 机制向上传递， 在 reduce 方法中收集所有关联对象，最终可以在 NavigationStack 中监听 preference 的变化，以此作为页面 push/pop 的时机。**

#### 2. 监听在 TabView 中 切 tab

**通过与 NavigationStack 相同的思路，可以为每一个 tab 绑定一个关联对象，并通过 Preference 向上传递被选中的tab的关联对象，在 TabView 中即可监听得到被选中的 tab。**


#### 3. 监听 present 出新的 sheet/fullScreenCover

关于的 presentation(sheet/fullScreenCover) 的详细原理这里暂不展开(我没时间写)，具体实现可参考 [PresentationStack](https://github.com/TStrawberry/PresentationStack) 


通过以上三处核心机制的拆解，实际上我们已经可以实现追踪页面的跳转的目的，如果觉得不容易理解，可以将上面提到的 **关联对象** 理解为 ViewController 😜.


OK，我们距离最终目标还差一块拼图：如何收集需要被调用的 onAppearAction 和 onDisappearAction


#### 如何收集需要被调用的 onAppearAction 和 onDisappearAction

在我的设计中，会通过一个 modifier 接收一个回调闭包，同时利用 Preference 收集一个页面中所有的回调闭包。在 Preference 处理和传递的过程中调整最终的调用的先后顺序，并将最终收集到的结果存储到每一个页面的关联对象上，等待被执行。


#### 触发调用

以上我们具备了 监听各种页面级的跳转 的能力，也收集了所有需要被执行的 onAppearAction 和 onDisappearAction，因此剩下需要做的就是把他们整合到一起：
当一次跳转发生时：
- 调用前一个页面的 onDisappearAction
- 调用后一个页面的 onAppearAction


> 本文仅阐述了最核心的设计，诸多细节控制未涉猎，供读者参考，也欢迎与我讨论。

---

# English Version

# An Enhanced Encapsulation of onAppear and onDisappear in SwiftUI

> This article assumes that the reader is already proficient in SwiftUI and has a thorough understanding of mechanisms such as `@State` and `Preference`.

## Motivation

In the project under discussion, the behavior of SwiftUI’s `onAppear` and `onDisappear` fails to meet practical requirements in two respects:

1. The order of invocation is not guaranteed. The `onAppear` of a subsequent page may be called before the `onDisappear` of the preceding page.
2. When a new `sheet` or `fullScreenCover` is presented, `onDisappear` is not called on the underlying page, even though that page is no longer visible to the user.

This article examines how a coherent architectural design can encapsulate a new API that satisfies the requirements above. The methods are provisionally named **onAppearAction** and **onDisappearAction**. The names themselves are incidental; the discussion is confined to the underlying mechanism.

## Architectural Implementation

The essential means of addressing these issues is to **observe page-level transitions within SwiftUI**, including the following:

1. Pushing and popping pages within a `NavigationStack`
2. Switching tabs within a `TabView`
3. Presenting a new `sheet` or `fullScreenCover`

#### 1. Observing Push and Pop within a NavigationStack

The subject has already been treated in [An Architectural Design for Dependency Injection on NavigationStack](./07-An-architecture-design-of-dependency-injection-on-NavigationStack.md). It may be summarized as follows:

**Associate an object with each page and propagate it upward through the Preference mechanism. Collect every associated object in the `reduce` method, and observe changes to the preference from within the `NavigationStack`. Those changes serve as the signal that a page has been pushed or popped.**

#### 2. Observing Tab Selection within a TabView

**Following the same approach as with `NavigationStack`, an associated object is bound to each tab, and the Preference mechanism propagates upward the associated object of the selected tab. The `TabView` can then observe which tab is currently selected.**

#### 3. Observing the Presentation of a Sheet or fullScreenCover

A detailed account of presentation (`sheet` / `fullScreenCover`) lies outside the scope of this article. A concrete implementation may be found in [PresentationStack](https://github.com/TStrawberry/PresentationStack).

Once these three mechanisms have been separated out, page transitions can be tracked. Where the notion is difficult to follow, the **associated object** described above may be regarded as the counterpart of a view controller 😜.

One further element is required before the objective is complete: the collection of the `onAppearAction` and `onDisappearAction` callbacks that are to be invoked.

#### Collecting the onAppearAction and onDisappearAction Callbacks

In this design, a view modifier accepts a callback closure, and `Preference` is used to gather every callback closure belonging to a given page. While the preference is processed and propagated, the eventual order of invocation is determined. The collected result is then stored on the associated object of each page, where it awaits execution.

#### Invoking the Callbacks

The preceding sections establish both the ability to observe page-level transitions and a complete collection of the `onAppearAction` and `onDisappearAction` callbacks that must be executed. What remains is to combine the two.

When a transition takes place:

- Invoke the `onDisappearAction` of the preceding page.
- Invoke the `onAppearAction` of the subsequent page.

> This article presents only the essential design. A number of finer points of control lie outside its scope. It is offered for the reader’s reference, and further discussion is welcome.


