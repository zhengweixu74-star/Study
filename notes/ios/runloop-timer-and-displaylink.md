# RunLoop、Timer 与 CADisplayLink

[返回笔记目录](../README.md)

## 本文目录

- [RunLoop 与运行模式](#runloop-与运行模式)
  - [输入源：Source0 与 Source1](#输入源source0-与-source1)
  - [Source0 示例：后台压缩完成后刷新预览](#source0-示例后台压缩完成后刷新预览)
  - [Source1 示例：通过 Mach 端口接收完成消息](#source1-示例通过-mach-端口接收完成消息)
- [Timer 的调度与重复触发](#timer-的调度与重复触发)
  - [倒计时与单调时钟](#倒计时与单调时钟)
- [CADisplayLink 的回调时机](#cadisplaylink-的回调时机)
  - [按经过时间计算动画进度](#按经过时间计算动画进度)
  - [运行模式与回调工作量](#运行模式与回调工作量)
  - [用途与取舍](#用途与取舍)
- [停止回调与引用关系](#停止回调与引用关系)
  - [RunLoop 的持有与 invalidate](#runloop-的持有与-invalidate)
  - [强引用环](#强引用环)
  - [弱引用代理](#弱引用代理)

## RunLoop 与运行模式

RunLoop 是在线程上反复等待事件、再分发事件的循环。每个 RunLoop 属于一条线程，RunLoop 分发的回调都在这条线程上执行。UIKit 应用的主线程 RunLoop 由框架创建并运行；其他线程默认不运行 RunLoop，需要接收 Timer 等事件时由应用自己启动。

没有事件时，RunLoop 让线程阻塞；事件到来时线程被唤醒，再按调度获得 CPU，线程状态见[并发、并行与线程状态](threads-and-scheduling.md#并发并行与线程状态)。当前模式中没有任何输入源和 Timer、到达运行期限或收到停止请求时，RunLoop 的这次运行结束。回调在线程上依次执行，某个回调执行很久，后面的事件都要等这个回调结束。

运行模式（mode）是一组输入源和 Timer 的集合，RunLoop 每次运行只处理一个模式中的事件。`default` 是一个具体的模式；`common` 是一个标记：注册到 `common` 的输入源或 Timer，会被加入所有带 common 标记的模式中。

主线程在列表滚动期间切换到 tracking 模式。只注册在 default 模式的 Timer 和 CADisplayLink，滚动期间停止回调；注册到 common 后，因为 tracking 模式也带 common 标记，滚动期间也会回调。common 只覆盖带 common 标记的模式；注册到 common 也不改变执行线程，主线程繁忙时回调照样延迟。

### 输入源：Source0 与 Source1

输入源（source）是加入 RunLoop 的事件来源对象，每个输入源记录事件怎样就绪，以及就绪后调用哪个回调。Source0 和 Source1 是 CFRunLoopSource 的两种版本，数字 0 和 1 只是版本号，与优先级和执行顺序无关。

Source1 依靠 Mach 端口接收事件。Mach 端口是内核管理的消息通信端点，与 TCP/UDP 的端口号无关。发送方持有端口的发送权限，接收方持有接收权限，内核把发到端口的消息交给接收方。每条消息由头部和消息体组成，头部记录接收端口、消息编号等信息。

| 比较项 | Source0 | Source1（Apple 平台的 Mach 端口源） |
|-|-|-|
| 创建时的版本 | version = 0 | version = 1 |
| 事件就绪方式 | 调用方或框架手动标记 source | 对应 Mach 端口收到消息 |
| 等待中的线程如何被唤醒 | 标记本身不唤醒线程；需要及时处理时，再唤醒目标 RunLoop | 内核把消息交给端口，并唤醒在这个端口上等待的线程 |
| 回调由谁执行 | 正在运行且当前模式包含该 source 的 RunLoop，在所属线程执行 | RunLoop 收到对应端口消息后，在所属线程执行 source 的回调 |
| CFRunLoopSourceSignal 的效果 | 标记指定的 Source0 为待处理 | 对 Source1 无效 |

一个 RunLoop 可以注册任意多个 Source0 和 Source1，数量只受内存、端口等资源限制。一个模式可以包含多个 source；同一个 source 可以加入多个模式，在同一个模式中重复添加同一个 source 只保留一份。Timer 是与输入源并列的另一类 RunLoop 对象。

回调在创建输入源时就设置好了。Source0 使用 CFRunLoopSourceContext，其中 version 为 0，perform 是处理回调。Source1 使用 CFRunLoopSourceContext1，其中 getPort 返回端口，perform 处理收到的消息。用 CFMachPort 创建 Source1 时，框架内部先接收消息，再调用创建 CFMachPort 时注册的回调。

发送方只负责发出就绪标记或消息，无需传递回调函数。RunLoop 找到对应的 source，再调用 source 上注册的回调。回调在哪个线程执行，取决于处理这个 source 的 RunLoop 属于哪个线程。

### Source0 示例：后台压缩完成后刷新预览

场景：后台线程压缩图片，完成后通知主线程刷新预览。主线程事先创建 refreshSource，把刷新逻辑设为 perform 回调，并把 refreshSource 加入主线程 RunLoop 的 common 模式。后台线程完成压缩后执行：

```swift
// 后台任务已完成，refreshSource 已注册到主线程 RunLoop
CFRunLoopSourceSignal(refreshSource)
CFRunLoopWakeUp(CFRunLoopGetMain())
```

1. 后台线程先把压缩结果保存好（并做好同步），再调用 `CFRunLoopSourceSignal(refreshSource)`，把 refreshSource 标记为待处理。
2. 后台线程调用 `CFRunLoopWakeUp(CFRunLoopGetMain())`。主线程正在阻塞时会被唤醒；主线程正在执行其他代码时，当前代码照常执行到结束，RunLoop 之后再处理事件。
3. 主线程 RunLoop 执行到处理 Source0 的阶段，并且当前模式包含 refreshSource 时，调用 perform 回调。
4. perform 回调读取压缩结果，在主线程刷新预览。

`CFRunLoopSourceSignal` 只做标记：perform 回调要等主线程 RunLoop 处理到 refreshSource 时才执行，标记也不向回调传递数据。标记只有“已标记”和“未标记”两种状态，所以处理之前连续 Signal 多次，回调可能只执行一次。每项任务都要处理时，把任务数据另外保存在一个加锁的队列中，由回调逐项取出。

Swift 可以直接调用 `CFRunLoopSourceSignal`、`CFRunLoopWakeUp` 这些 Core Foundation 的 C 函数，原理见[Swift 与 C 接口互操作](../language/swift-basics.md#swift-与-c-接口互操作)。自己创建的 Source0 才需要手动 Signal；使用框架封装好的功能时，框架负责标记和唤醒。`CFRunLoopWakeUp` 唤醒的是处理这个 source 的 RunLoop，这里是主线程的 RunLoop。

### Source1 示例：通过 Mach 端口接收完成消息

仍以“后台压缩完成后刷新预览”为例。主线程创建 CFMachPort 并注册接收回调，再用 CFMachPortCreateRunLoopSource 创建关联这个端口的 Source1，加入主线程 RunLoop。事件按以下步骤传递：

1. 主线程 RunLoop 在包含这个 Source1 的模式中等待消息。
2. 后台线程完成压缩，按双方约定向端口发送 Mach 消息，例如把 msgh_id 设置为 1001，表示压缩完成。
3. 消息到达端口后，内核唤醒正在等待这个端口消息的主线程。主线程什么时候真正开始运行，还要看线程调度。
4. 主线程 RunLoop 根据收到消息的端口找到 Source1，调用对应的回调。回调根据消息编号和数据判断发生了什么，然后刷新预览。

这个过程中，消息传递和唤醒由内核完成，回调由主线程的 RunLoop 执行，所以发送方无需调用 `CFRunLoopSourceSignal` 和 `CFRunLoopWakeUp`。主线程繁忙或当前模式不包含这个 source 时，回调会延迟。

消息编号属于 Mach 消息本身：每条消息的头部 mach_msg_header_t 都有 msgh_id 字段；1001 是本例中发送方和接收方约定的值，系统不规定它的含义。例如，同一个端口、同一个 Source1 可以接收 1001（压缩完成）、1002（保存完成）、1003（任务取消）三种消息。处理分两步：RunLoop 先按接收端口找到 source；回调再按消息编号选择处理分支。

## Timer 的调度与重复触发

Timer 保存触发时间和回调，RunLoop 负责在触发时间到达后执行回调。Timer 不占用单独的线程，RunLoop 也无需轮询时间：RunLoop 向内核设置最近一次触发时间的定时通知，然后让线程阻塞，时间到达时内核唤醒线程。

1. `scheduledTimer` 把 Timer 加入当前线程 RunLoop 的 default 模式，记录首次触发时间、重复间隔和回调。
2. RunLoop 按最近的触发时间向内核设置定时通知。Apple 开源的 Core Foundation 早期实现用 `mach_absolute_time` 的计数表示触发时间，并调用 `mk_timer_arm` 设置通知；当前 iOS 的内部实现可能不同。
3. RunLoop 没有事件要处理时让线程阻塞。时间到达后，内核发送消息唤醒线程，线程再按调度获得 CPU。
4. 线程开始执行后，RunLoop 找出已经到期的 Timer，在这个线程上执行回调。主线程繁忙、当前模式不包含这个 Timer，都会让回调延迟。
5. 重复 Timer 按最初的时间表安排下一次。原定在 1、2、3 秒触发，第一次延迟到 1.3 秒、且回调很快结束，第二次仍在 2 秒触发。一次延迟跨过了好几个触发点时，错过的触发点被跳过：Timer 只执行一次回调，再按时间表安排下一个尚未到达的触发点。

设置 tolerance 允许系统把多个 Timer 的唤醒合并在一起以节省电量，因此回调可能在 tolerance 范围内稍晚执行。

`scheduledTimer` 只把 Timer 加入 default 模式。列表滚动期间也要触发时，用 `Timer(timeInterval:repeats:block:)` 创建一个尚未加入 RunLoop 的 Timer，再调用 `RunLoop.main.add(timer, forMode: .common)`，模式规则见[RunLoop 与运行模式](#runloop-与运行模式)。加入更多模式只增加回调的机会，回调仍可能延迟。

### 倒计时与单调时钟

倒计时每次回调时，用“截止时间 − 当前时间”重新算剩余时间。回调每次都可能延迟，把剩余值减 1 会让误差越积越多。

Date 是日历时间，适合有具体日期的截止时间；用户修改系统时间时，Date 会跟着变。只计算经过了多长时间时，使用单调时钟。单调时钟是不受系统时间修改影响、只会向前走的时间，例如 `CACurrentMediaTime()` 返回的时间。Cell 的可见性与计时器配合见 [Cell 中的倒计时](uikit-layout-and-mvvm.md#cell-中的倒计时与可见性)。

## CADisplayLink 的回调时机

`CADisplayLink` 是 Core Animation 提供的、随屏幕刷新触发回调的对象。每次屏幕刷新前，系统调用创建时指定的 `target` 的 `selector` 方法，并把 `CADisplayLink` 对象作为参数传入。`target` 是接收回调的对象，`selector` 是回调方法的 Objective-C 方法名。

创建 `CADisplayLink` 后，调用 `add(to:forMode:)` 把 `CADisplayLink` 加入某个 `RunLoop`，系统才开始回调，回调在这个 `RunLoop` 所属的线程执行。回调中要修改 UIKit 视图时，就加入主线程 `RunLoop`，并在主线程创建和停止。

```text
显示刷新时序
    → 系统安排 CADisplayLink 回调
    → 注册的 RunLoop 所在线程执行目标方法
    → 应用计算动画进度并修改视图
    → 系统为后续画面准备内容
```

`CADisplayLink` 只提供回调时机，位置和进度由应用自己计算；例如做自动滚动时，应用要自己算出位置并设置 `contentOffset`。普通 `Timer` 的触发时间与屏幕刷新不同步；`CADisplayLink` 与屏幕刷新同步，但主线程繁忙、模式不匹配、系统降低帧率时，回调仍会延迟。RunLoop 的循环次数和屏幕刷新次数各自独立。

### 按经过时间计算动画进度

动画进度按实际经过的时间计算。回调可能被跳过或延迟，按“回调次数 × 固定增量”计算时，动画总时长会随回调次数变化。开始时间和当前时间都用 `CACurrentMediaTime()` 这个单调时钟获取。

```swift
// duration 已确定为大于 0 的动画时长。
let elapsed = CACurrentMediaTime() - startTime
let progress = min(elapsed / duration, 1)
let offset = startOffset + (endOffset - startOffset) * progress
```

例如，动画时长 0.5 秒，经过 0.2 秒时进度为 40%。中间少了几次回调时，下一次回调仍按实际经过的时间计算，动画总时长不变。这种算法让动画速度与回调次数无关，但掉帧时画面仍会跳一下。

`targetTimestamp` 是下一帧预计显示的时间，可以用来计算下一帧应该显示的状态。回调中剩余的可用时间是 `targetTimestamp` 减去当前时间，比整个刷新间隔短。

### 运行模式与回调工作量

`CADisplayLink` 只加入主线程 `.default` 模式时，列表滚动期间暂停回调；加入 `.common` 后，滚动期间也会回调，规则见[RunLoop 与运行模式](#runloop-与运行模式)。`.common` 只决定在哪些模式下回调；用户开始拖动后应该由用户控制滚动时，应用要在手势或滚动回调中主动停止自动动画。

每次回调只做轻量的进度计算和必要的界面更新。60 Hz 屏幕的刷新间隔约 16.7 毫秒，120 Hz 约 8.3 毫秒，这段时间还要留给系统的布局和渲染。大图解码、复杂布局放在回调中执行时，耗时不变，仍占用主线程。

### 用途与取舍

`CADisplayLink` 适合需要应用每一帧自己计算画面的任务，例如自定义物理动画、波浪曲线、音频波形显示、序列帧动画，以及把高频输入合并后按屏幕刷新节奏更新界面。显示音频波形时，音频采集和解码由音频框架的回调驱动，`CADisplayLink` 回调只按帧更新波形画面。播放 GIF 时，每次回调检查当前时间是否到了下一帧，到了才换图；FLAnimatedImage 就是这样做的。

是否使用 `CADisplayLink`，要看画面是否持续变化、是否需要应用自己计算，以及现有框架是否已有刷新循环。普通的位移、透明度和弹簧动画交给系统的 UIView 动画即可；`MTKView` 自带绘制循环，无需再加一个 `CADisplayLink`。

回调频率反映回调是否及时；每次回调产生的画面是否显示到了屏幕上，要用渲染工具另外观测，方法见 [回调 FPS 与卡顿监测](rendering-and-performance.md#回调-fps-与卡顿监测)。

## 停止回调与引用关系

Timer 和 CADisplayLink 都由 RunLoop 持有，停止回调和释放对象的规则相同。

### RunLoop 的持有与 invalidate

RunLoop 强引用加入其中的 Timer 和 CADisplayLink。只把保存它们的属性设为 nil，RunLoop 仍持有对象，回调会继续。停止时先调用 `invalidate()`，再清空属性：

```swift
timer?.invalidate()
timer = nil

displayLink?.invalidate()
displayLink = nil
```

`invalidate()` 让对象失效，并把对象从 RunLoop 中移除；CADisplayLink 失效时还会释放对 `target` 的强引用。失效的对象不能重新启动，需要时重新创建。`invalidate()` 应在添加对象的那个线程调用；界面用的 Timer 和 CADisplayLink 都在主线程创建和停止。

CADisplayLink 的 `isPaused = true` 只暂停回调：CADisplayLink 仍留在 RunLoop 中，仍强引用 `target`；设回 `false` 就恢复回调。

### 强引用环

强引用环是多个对象的强引用首尾相连，环中对象的引用计数都降不到零。Timer 强引用回调闭包，target/selector 形式的 Timer 强引用 target；CADisplayLink 强引用 target。控制器保存 CADisplayLink，同时把自己作为 `target` 时，引用关系是：

```text
控制器 → CADisplayLink → 控制器
RunLoop → CADisplayLink
```

Timer 的闭包强捕获持有 Timer 的控制器时，也形成同样的环。控制器在环中时引用计数降不到零，`deinit` 不会执行，所以停止回调的代码要放在任务完成、任务取消或页面不再需要回调的时机主动调用。`deinit` 中的 `invalidate()` 只在环被打破后才有机会执行，只作为兜底。

打破引用环的方法：Timer 的闭包用 [weak 捕获](../language/swift-basics.md#weak-与-unowned-的使用条件)；CADisplayLink 和 target/selector 形式的 Timer 强引用 target，要改用弱引用代理作为 target。打破环后仍要调用 `invalidate()`：控制器释放后，RunLoop 还持有 Timer 或 CADisplayLink，回调会继续空转。

### 弱引用代理

弱引用代理是一个中间对象：代理对象代替控制器作为 `target` 接收回调，再通过 `weak` 属性把回调转给控制器。`weak` 引用不阻止控制器释放，控制器释放后 `weak` 属性自动变为 `nil`。引用关系变为：

```text
控制器 → CADisplayLink → 代理 --weak→ 控制器
RunLoop → CADisplayLink
```

下面用一个页面的透明度动画演示。`@MainActor` 表示这些类的代码在主线程执行；本例把 `CADisplayLink` 加入主线程 RunLoop，所有 UIKit 访问都在主线程进行。并发概念见 [MainActor、主队列与异步调用](../language/swift-basics.md#mainactor主队列与异步调用)。

```swift
import UIKit
import QuartzCore

@MainActor
private final class DisplayLinkProxy: NSObject {
    weak var owner: AnimatedViewController?

    init(owner: AnimatedViewController) {
        self.owner = owner
        super.init()
    }

    @objc func tick(_ link: CADisplayLink) {
        owner?.updateAnimation(link)
    }
}

@MainActor
final class AnimatedViewController: UIViewController {
    private var displayLink: CADisplayLink?
    private var startTime: CFTimeInterval = 0
    private let duration: CFTimeInterval = 0.5

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        startAnimation()
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        stopAnimation()
    }

    private func startAnimation() {
        stopAnimation()
        startTime = CACurrentMediaTime()
        view.alpha = 0
        let proxy = DisplayLinkProxy(owner: self)
        let link = CADisplayLink(
            target: proxy,
            selector: #selector(DisplayLinkProxy.tick(_:))
        )
        displayLink = link
        link.add(to: .main, forMode: .common)
    }

    fileprivate func updateAnimation(_ link: CADisplayLink) {
        let progress = min((CACurrentMediaTime() - startTime) / duration, 1)
        view.alpha = CGFloat(progress)
        if progress >= 1 {
            stopAnimation()
        }
    }

    func stopAnimation() {
        displayLink?.invalidate()
        displayLink = nil
    }
}
```

系统调用代理的 `tick(_:)`，代理再调用控制器的 `updateAnimation(_:)`。控制器释放后，`owner` 变为 `nil`，代理不再转发。`CADisplayLink` 强引用代理，所以控制器无需另外保存代理。

弱引用代理只解除 `CADisplayLink` 对控制器的强引用，任务结束时仍要调用 `invalidate()` 停止回调。上例在动画完成和 `viewDidDisappear` 时停止；还有取消、用户接管等情况时，也要在对应位置调用 `stopAnimation()`。不同展示方式是否触发消失回调，见 [push、present 与出现消失回调](uikit-layout-and-mvvm.md#pushpresent-与出现消失回调)。
