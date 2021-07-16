# 弹幕数据的设计

笔者最近参与了弹幕功能的开发，将采用的技术方案记录于此。本文并不会涉及弹幕的所有内容，将只聚焦于对数据的处理，并且内容基于以下几点：

- 以最基础的横向滚动的弹幕为例，只讨论一条轨道的情况
- 弹幕不会发生重叠
- 弹幕的出现和消失满足：**先出现的弹幕一定先消失** 的规律

## 存储弹幕的信息
当一条弹幕在某一时刻被发出时，会生成一条弹幕数据如下：
```swift
struct DanmakuItem {
  let appearTiming: NSTimeInterval
  let disappearTiming: NSTimeInterval
  ...
}
```
涉及本文的有 `appearTiming` 和`disappearTiming` 两个字段，即出现时间和消失时间。
```swift
disappearTiming = appearTiming + 弹幕在屏幕上展示的时间
```
所有弹幕数据会依次存放进的`danmakuItems`数组里面，基于本文一开始的约定，他们是按照`appearTiming`排好序的，即两个相邻的弹幕满足：
```swift
danmakuItems[i].appearTiming    < danmakuItems[i + 1].appearTiming
danmakuItems[i].disappearTiming < danmakuItems[i + 1].disappearTiming
```

## 让弹幕动起来

笔者选择的方案是利用 CADisplayLink，计算每一帧上**出现在屏幕上的弹幕（appearItems）**和**它们对应的位置**。
### 计算出现屏幕上的弹幕
CADisplayLink 每次回调会关联到一个不断增长的时间 `seekTiming`上:

```swift
seekTiming += 1.0 / 60.0 // 以60的帧率为例
```

在`danmakuItems`中查询包含这个时间的数据就好，即`appearItems`包含了满足以下条件的数据：

```swift
danmakuItems[i].appearTiming <= seekTiming < danmakuItems[i].disappearTiming
```
### 计算弹幕的位置
现在已经计算好了`appearItems`，那它们的位置就可以根据弹幕配置的 速率，播放器宽度，以及滚动的时间计算得到，然后更新对应的UI展示就好。以本文中约定的弹幕为例：

```swift
let speed = ... 
let playerWidth = ...
let distance = speed * (seekTiming - appearTiming) 
// 此时弹幕就应该展示在距离播放器左侧 (playerWidth - distance) 的位置上
// 这里仅以横向滚动的弹幕为例，此处可以根据弹幕的运动方式替换成任意的计算逻辑
```

## 支持Seek操作
当用户拖动进度条时，所有弹幕应该更新到那一时刻的状态(类似B站的实现)。
其实无论是视频顺序播放还是用户拖动了进度条，弹幕的更新逻辑都是：
- 根据一个时间查询弹幕数据(`appearItems`)
- 更新这些弹幕对应的UI

因此这其实和上一节讨论的是一个问题，命名为 `seekTiming` 也是基于此。

### 处理历史弹幕

对于一个回放类型的视频，在视频播放开始可能就已经有若干弹幕了，他们被发出的时间 `appearTiming` 可能是视频进度的任意时刻。笔者的做法是在一开始就在子线程把 `danmakuItems` 计算好，以此来支持用户向后拖动进度条的情况，即在视频播放之初就计算好所有历史弹幕的显示逻辑。当用户发出新的弹幕的时候，再把数据加入到  `danmakuItems` 中。

## 效率
### 视频顺序播放时
当视频顺序播放时，`appearItems` 在 `danmakuItems`中的位置变化其实是一个**滑动窗口**。

任意前后两帧的`appearItems`大部分时候是一样的, 出现差异的情况只有：

- 最前面的弹幕滚出了屏幕

- 新的弹幕开始出现


因此在查询`appearItems`的时候，可以利用前一次的查询结果，不必每一次都在整个`danmakuItems`数组中查找。

### 用户Seek操作时
由于时间跨度变大，无法利用之前的结果了。但由于数据是按时间排序的，因此可以利用**二分搜索**优化查询效率。

## 横竖屏切换
到目前为止，本文都没有讨论UI如何展示，只涉及了对数据的处理。对于横竖屏切换的情况，需要为两种情况分别创建`danmakuItems`，计时器根据业务需要 选择性的驱动不同的数据进行显示。

## 大数组优化

上文提到历史弹幕的情况，如果数据非常多`danmakuItems`非常大，向其中插入数据无疑是低效的，时间复杂度为 n。

可以采用以下方案优化：

- 用小数组映射到大数组，将对大数组的操作转化为对小数组的操作
- 将`danmakuItems`替换为数据库，内存中只预加载一部分数据，采用**滑动窗口**，随着播放进度的不断更新内存数据

