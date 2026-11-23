# 响应式编程与事件通知

[返回笔记目录](../README.md)

## 本文目录

- [NotificationCenter 与通知回调](#notificationcenter-与通知回调)
  - [注册、匹配与分发](#注册匹配与分发)
  - [回调在哪个线程执行](#回调在哪个线程执行)
  - [移除观察关系与闭包捕获](#移除观察关系与闭包捕获)
- [响应式关系与 OpenCombine](#响应式关系与-opencombine)
- [事件值、失败与流结束](#事件值失败与流结束)
- [无载荷事件：PassthroughSubject<Void, Never>](#无载荷事件passthroughsubjectvoid-never)
- [@Published：状态订阅与赋值时序](#published状态订阅与赋值时序)
  - [状态与事件：为什么都用 sink 仍不相同](#状态与事件为什么都用-sink-仍不相同)
  - [CurrentValueSubject：可主动发送的当前状态](#currentvaluesubject可主动发送的当前状态)
- [receive(on:) 与主队列异步调度](#receiveon-与主队列异步调度)
- [CombineX 的订阅关系与事件传递](#combinex-的订阅关系与事件传递)
- [CombineX 与 NotificationCenter 的选择](#combinex-与-notificationcenter-的选择)
- [removeDuplicates() 与连续重复值](#removeduplicates-与连续重复值)
- [RxSwift：输入组合与按钮状态绑定](#rxswift输入组合与按钮状态绑定)
- [RxSwift：Observable 与订阅](#rxswiftobservable-与订阅)
- [RxSwift：debounce 与 distinctUntilChanged](#rxswiftdebounce-与-distinctuntilchanged)
  - [debounce 的发送时机](#debounce-的发送时机)
  - [distinctUntilChanged 的比较范围](#distinctuntilchanged-的比较范围)
- [RxSwift：observe(on:) 与回调线程](#rxswiftobserveon-与回调线程)
- [RxSwift：flatMapLatest 与 flatMap](#rxswiftflatmaplatest-与-flatmap)
  - [返回类型与闭包执行时机](#返回类型与闭包执行时机)
  - [每次订阅的内部状态](#每次订阅的内部状态)
  - [取消订阅、取消请求与编号检查](#取消订阅取消请求与编号检查)

## NotificationCenter 与通知回调

`NotificationCenter` 是 Foundation 提供的进程内通知分发对象。发送方把事件交给通知中心，通知中心调用符合条件的观察者回调。回调是一段代码：观察者在注册时提供回调，通知中心收到匹配的通知时执行回调。

一次通知由 `Notification` 表示，主要包含三个字段：

| 字段 | 含义 |
| --- | --- |
| `name` | 通知名称，用于区分事件 |
| `object` | 与事件关联的对象，通常由发送方传入自身；也可以为 `nil` |
| `userInfo` | 可选的附加数据字典，例如新的标题 |

### 注册、匹配与分发

观察者调用 `addObserver(forName:object:queue:using:)`，提交名称、对象过滤条件、执行队列和回调闭包。通知中心保存这次观察关系，并返回一个标识对象，通常称为 token；调用方之后用 token 移除这次观察关系。

发送方调用同一个通知中心的 `post`。通知中心查找与这次通知匹配的观察关系，并调用这些观察关系的回调。匹配规则如下：

- 注册时指定 `name`，只接收该名称的通知。
- 注册时指定 `object`，只接收与该对象匹配的通知。
- 两项都指定时，两项都要匹配。
- 某项为 `nil` 时，通知中心不按该项过滤。

这套机制的工作模型是“保存观察关系 → 接收通知 → 按条件查找 → 执行回调”。工作模型描述的是外部行为；通知中心内部用什么数据结构保存观察关系，由系统实现决定。每个 `NotificationCenter` 实例只管理自己的观察关系：发给实例 A 的通知，只分发给在实例 A 上注册的观察者。

观察关系只接收注册之后发送的通知，通知中心不补发注册之前的历史通知。因此，观察者必须在通知发送之前完成注册。

### 回调在哪个线程执行

使用上述闭包接口且 `queue` 为 `nil` 时，通知中心在调用 `post` 的线程上同步调用回调。同步指回调执行完后，这次 `post` 才返回。回调线程由发送线程决定，与注册观察者的线程无关：观察者在主线程注册，后台线程发送通知时，回调就在后台线程执行。

`queue` 的类型是 `OperationQueue?`，用于指定回调的执行队列。传入 `.main` 时，回调在主队列执行。`queue` 参数只规定回调在哪个队列执行，没有规定 `post` 在回调执行之前还是之后返回，代码不应依赖其中任何一种顺序。任务队列和线程的区别见[iOS 线程、GCD 与同步](threads-and-scheduling.md#operation-与-operationqueue)。更新 UIKit 界面的代码需要在主线程执行。

下面的示例中只有一个匹配的观察者，回调里也没有再次发送通知：

```swift
import Foundation

let center = NotificationCenter.default
let sender = NSObject()
let name = Notification.Name("titleDidChange")

let token = center.addObserver(
    forName: name,
    object: sender,
    queue: nil
) { notification in
    if let title = notification.userInfo?["title"] as? String {
        print("收到通知：\(title)")
    }
}

print("发送前")
center.post(
    name: name,
    object: sender,
    userInfo: ["title": "新标题"]
)
print("发送后")

center.removeObserver(token)
```

输出顺序为“发送前 → 收到通知：新标题 → 发送后”。因为 `post` 同步调用回调，回调执行多久，发送线程就等待多久。回调内部再次调用 `post` 时，会形成嵌套调用。有多个观察者时，业务逻辑不要依赖观察者之间的执行顺序。

### 移除观察关系与闭包捕获

使用闭包接口时，通知中心持有返回的 token 和回调闭包。调用方把自己保存的 token 变量设为 `nil` 后，通知中心仍保留这次观察关系，回调照常执行。取消观察的方法是：不再需要接收事件时，对注册时使用的通知中心调用 `removeObserver(token)`。

通知中心持有回调闭包，闭包强引用的对象也会因此一直存活。例如闭包强引用 `self` 时，`self` 无法释放，`self` 的 `deinit` 也不会执行，写在 `deinit` 中的 `removeObserver` 就不会被调用。解决方法是：闭包用 `[weak self]` 弱引用 `self`，并在明确的生命周期结束点调用 `removeObserver`。`[weak self]` 只解除闭包对 `self` 的强引用；观察关系仍要靠 `removeObserver` 取消。

## 响应式关系与 OpenCombine

**响应式编程：** 代码先声明“数据或事件变化后要执行哪些代码”，之后每次变化都会触发这些代码。例如，View 先订阅点赞数；点赞数变成 13 时，View 收到 13，把 Label 文字更新为“13”。又如，View 先订阅定位目标；定位目标准备好后被发送出来，View 收到后安排滚动。修改普通变量不会触发任何代码；数据源要把变化暴露为可订阅的发布者，接收方还要写好收到值后的处理代码。

OpenCombine 是开源的 Combine 兼容实现，提供发布、订阅、转换、过滤、组合数据和事件的工具。使用 OpenCombine 类型时导入 `OpenCombine`；使用系统 Combine 类型时导入 `Combine`。

```text
PassthroughSubject.send(_:) 发送值
    → sink 注册的接收闭包收到值
```

**订阅与生命周期：**

- `PassthroughSubject` 只把值发给发送时已经订阅、且有接收需求的订阅者。`PassthroughSubject` 不保存值，后来的订阅者收不到之前发送的值；没有订阅者时，发送的值直接丢弃。
- `.sink { value in ... }` 建立订阅并处理收到的数据。
- `sink` 返回 `AnyCancellable`。调用方持有 `AnyCancellable` 期间，订阅保持有效；调用 `cancel()` 或 `AnyCancellable` 被释放后，订阅取消，订阅者不再接收事件。`AnyCancellable` 可以保存在变量中，也可以用 `.store(in: &cancels)` 存入集合。

## 事件值、失败与流结束

事件流是同一个通道先后发送的一串数据，例如下载进度 `10% → 40% → 70%`。`PassthroughSubject<Output, Failure>` 中，`Output` 是普通数据的类型，`Failure` 是失败结束时携带的错误类型。

| 发送动作 | sink 中接收的位置 | 之后能否继续收到该流的数据 |
|-|-|-|
| `send(value)` | `receiveValue` | 可以，直到流结束或订阅取消 |
| `send(completion: .finished)` | `receiveCompletion` 的 `.finished` 分支 | 不可以，已正常结束 |
| `send(completion: .failure(error))` | `receiveCompletion` 的 `.failure` 分支 | 不可以，已失败结束 |

下面的代码模拟下载进度：

```swift
import OpenCombine

enum DownloadError: Error {
    case networkDisconnected
}

let progress = PassthroughSubject<Int, DownloadError>()
let subscription = progress.sink(
    receiveCompletion: { completion in
        switch completion {
        case .finished:
            print("下载正常结束")
        case .failure(let error):
            print("下载失败：\(error)")
        }
    },
    receiveValue: { value in
        print("下载进度：\(value)%")
    }
)

progress.send(10)
progress.send(40)
progress.send(completion: .failure(.networkDisconnected))
progress.send(70) // 该流已经结束，不再触发 receiveValue
```

订阅保持有效时，预期依次输出“下载进度：10%”“下载进度：40%”“下载失败：networkDisconnected”。失败通知进入 `receiveCompletion` 的 `.failure` 分支，`error` 的值是 `DownloadError.networkDisconnected`；`receiveValue` 只接收普通值。

失败由下载代码主动调用 `send(completion: .failure(...))` 报告；Subject 只转发调用方发送的内容，自身不检测断网或其他业务错误。`.failure` 会结束这条流，之后再调用 `send(70)` 也不会触发 `receiveValue`。

类型改为 `PassthroughSubject<Int, Never>` 时，普通数据仍是 `Int`。`Never` 是没有任何值的类型，代码无法构造 `Never` 值，因此这条流无法以 `.failure` 结束。这条流仍可以 `send(10)`，也可以用 `send(completion: .finished)` 正常结束；把 `.networkDisconnected` 作为失败值发送会产生类型错误。`Never` 只表示这条流没有失败结束这种情况，流仍然可以正常结束。

## 无载荷事件：PassthroughSubject<Void, Never>

`PassthroughSubject<Output, Failure>` 允许代码主动发送值和结束通知。`Output` 是普通值的类型，`Failure` 是失败结束时携带的错误类型，且必须遵循 `Error`。

`PassthroughSubject<Void, Never>` 的普通事件携带空元组 `()`，不带业务字段；`Failure` 为 `Never`，这条事件流不会以 `.failure` 结束。

```swift
import OpenCombine

let refreshSubject = PassthroughSubject<Void, Never>()
let subscription = refreshSubject.sink { _ in
    print("收到刷新事件")
}
refreshSubject.send(())
refreshSubject.send(completion: .finished)
```

`send(())` 发送了一个值，这个值是没有字段的空元组。`Failure` 为 `Never` 时流仍可以正常结束，因此可以发送 `.finished`。示例用 `subscription` 变量持有订阅，使订阅保持有效。`PassthroughSubject` 不保存值，后来的订阅者收不到之前的事件，因此适合表达“发生了一次操作”。

## @Published：状态订阅与赋值时序

`@OpenCombine.Published` 是属性包装器。属性包装器是标注在属性声明上、为属性读写附加逻辑的类型。`@Published` 标注在类的属性上后，属性每次被赋值都会发布新值。读取 `count` 得到属性当前值；访问 `$count` 得到可订阅的 Publisher，即向订阅者提供值的发布者。因此，订阅 `@Published` 属性直接使用 `$count`，不需要另建 `PassthroughSubject`。

```swift
import OpenCombine

final class Counter {
    @OpenCombine.Published var count = 0
}

let counter = Counter()
let subscription = counter.$count.sink { newValue in
    print(newValue, counter.count)
}
counter.count = 1
// 直接同步订阅的预期输出：
// 0 0
// 1 0
```

直接使用 `sink` 订阅 `$count` 后，订阅者先收到当前值 `0`。执行 `counter.count = 1` 时，`@Published` 先发布新值 `1`，然后才更新属性存储。这种顺序称为 `willSet` 语义。因此，赋值期间同步执行的接收闭包拿到 `newValue == 1`，此时再读取 `counter.count` 仍得到 `0`。

接收闭包读到 `0`，是因为属性存储还没有更新，与 Swift 属性观察器的 `oldValue` 无关；`oldValue` 是 `didSet` 观察器中表示旧值的参数名。

链路中用 `receive(on:)` 把值异步交给接收闭包时，接收闭包执行得更晚。此时读取属性，可能读到本次的新值，也可能读到之后又一次赋值的结果。闭包参数 `newValue` 始终是本次发布的值，因此处理通知时应使用 `newValue`。

`@Published` 适合表达可观察状态，`PassthroughSubject` 适合表达不需要保存当前值的事件。两者都用 `sink` 订阅，但初始值、是否保存当前值和发送方式都不同。

### 状态与事件：为什么都用 sink 仍不相同

状态表示“现在是什么”。例如 count 当前等于 3，即使没有订阅者，这个当前值仍然有意义。可观察状态既保存当前值，也允许订阅者接收赋值时发布的新值。

事件表示“发生了一次什么”。例如按钮被点击一次；点击之后才订阅的对象，通常不需要收到这次点击。`PassthroughSubject` 只把事件发给发送时已有的订阅者，不保存事件。

| 对比项 | @Published | PassthroughSubject |
|-|-|-|
| 适合表达的数据 | 当前状态，例如当前计数是 3 | 一次事件，例如按钮被点击一次 |
| 初始值 | 属性需要初始化，例如 count = 0 | 创建时不需要初始值 |
| 当前值保存 | 保存当前值，可以直接读取 count | 不保存当前值，也不补发之前发送的普通事件 |
| 发送方式 | 给属性赋值，例如 model.count = 2 | 主动调用 send，例如 model.click.send(()) |
| 直接使用 sink 开始订阅 | 先收到属性的当前值，再接收后续赋值发布的新值 | 不会因订阅自动收到一个初始值；订阅有效期间接收之后发送的新事件 |

新订阅者先收到的是订阅时的当前值。例如 count 从 0 改成 1 之后才开始订阅，新订阅者先收到 1，而不是声明时的初始值 0。

下面的示例先修改状态、发送事件，再建立订阅，并用变量持有 `sink` 返回的订阅。注释为预期输出。

```swift
import OpenCombine

final class Model {
    @OpenCombine.Published var count = 0
    let click = PassthroughSubject<Void, Never>()
}

let model = Model()

// 此时还没有订阅者。
model.count = 1
model.click.send(())

let countSubscription = model.$count.sink { value in
    print("当前计数：\(value)")
}
// 立即输出：当前计数：1

let clickSubscription = model.click.sink { _ in
    print("发生了一次点击")
}
// 此时没有输出；订阅前发送的点击不会补发。

model.count = 2
// 输出：当前计数：2

model.click.send(())
// 输出：发生了一次点击
```

读取 `model.count` 得到属性的当前值；访问 `model.$count` 得到该属性的发布者。`sink` 建立订阅，并在收到值时执行接收闭包。`model.$count` 和 `model.click` 都用 `sink` 订阅，但 `model.$count` 保存当前值，并在订阅时立即发送当前值；`model.click` 不保存值，只发送订阅之后的事件。

### CurrentValueSubject：可主动发送的当前状态

`CurrentValueSubject<Output, Failure>` 是保存当前值的 Subject。创建时必须提供初始值；读取 `value` 得到当前值；调用 `send(_:)` 发送新值，并把新值保存为当前值。新订阅者订阅时立即收到当前值，之后再接收新发送的值。

下面的示例使用系统 Combine，注释为预期输出。

```swift
import Combine

let notes = CurrentValueSubject<[String], Never>([])
notes.send(["笔记 A"])

let subscription = notes.sink { value in
    print(value)
}
// 立即输出：["笔记 A"]
```

| 对比项 | CurrentValueSubject | PassthroughSubject |
| --- | --- | --- |
| 初始值 | 创建时必须提供 | 不需要 |
| 当前值 | 保存，可通过 `value` 读取 | 不保存 |
| 新订阅者 | 立即收到当前值，再接收之后发送的值 | 只接收订阅之后发送的值 |

Kotlin 的 `StateFlow` 与 `CurrentValueSubject` 相近，对应关系和区别见[StateFlow 与 MutableStateFlow](../android/stateflow-and-lifecycle-collection.md#stateflow-与-mutablestateflow)。

## receive(on:) 与主队列异步调度

调度器是决定任务在哪里、何时执行的对象。`DispatchQueue.main` 是主队列，主队列中的任务在主线程执行。`DispatchQueue.main.ocombine` 把主队列包装成 OpenCombine 的调度器，任务仍提交到同一个主队列。

`.receive(on: DispatchQueue.main.ocombine)` 把下游接收值和结束通知的动作安排到主队列。下游指链路中位于该操作符之后、接收事件的代码；后面再出现其他调度操作符时，从那个操作符开始，执行位置再次改变。`receive(on:)` 只影响下游；上游的网络请求、计算或属性赋值仍在原来的线程执行。

OpenCombine 的 DispatchQueue 适配器在 `schedule` 方法中调用 `queue.async`。因此 `receive(on: DispatchQueue.main.ocombine)` 总是异步提交下游接收动作：即使 `send` 已经在主线程调用，接收闭包也会作为新任务提交到主队列，不在当前调用栈中执行。

没有设置延迟时，接收动作仍然是异步的：主队列执行完当前任务、轮到接收任务时，才调用接收闭包。这个时刻没有固定延迟，也不对应“下一帧”或“下一轮 RunLoop”。总是异步提交是这个调度器的行为；其他调度器上的 `receive(on:)` 是否异步，取决于各自的实现。

```swift
import Dispatch
import OpenCombine
import OpenCombineDispatch

let subject = PassthroughSubject<Void, Never>()
let subscription = subject
    .receive(on: DispatchQueue.main.ocombine)
    .sink { print("C") }

DispatchQueue.main.async {
    print("A")
    subject.send(())
    print("B")
}
// 预期顺序：A、B、C
```

这个顺序成立的条件是：订阅保持有效，主队列能正常执行任务，`print("A")`、`send`、`print("B")` 位于同一个主队列任务中。执行过程是：`send` 把接收任务提交到主队列后立即返回；当前任务继续打印 `B` 并结束；主队列随后执行接收任务，打印 `C`。

在后台线程调用 `send` 时，后台线程和主线程并行运行。主线程可能在后台线程执行 `send` 之后的语句之前，就已经执行接收闭包。异步提交只保证接收闭包在主线程执行，不保证接收闭包晚于后台线程的后续语句。

GCD 基础概念见[iOS 线程、GCD 与同步](threads-and-scheduling.md#gcd-的队列与提交方式)。

## CombineX 的订阅关系与事件传递

CombineX 与 OpenCombine 是两个不同的开源 Combine 兼容实现。下文用 CombineX 说明发布订阅的公共模型；模块名称和平台适配接口以实际使用的实现为准。

| 对象或协议 | 职责 |
| --- | --- |
| `Publisher` | 发布者协议：接受订阅，并向订阅者提供值或结束通知。 |
| `Subscriber` | 订阅者协议：接收订阅关系、普通值和结束通知。 |
| `Subscription` | 表示一次订阅关系，提供请求接收数量和取消订阅的接口。 |
| `Subject` | 支持主动发送的发布者协议；调用方可以使用 `send` 发送值或结束通知。 |

`sink` 创建一个订阅者，订阅者保存接收闭包；`sink` 再把订阅者交给上游发布者。发布者建立订阅关系，调用订阅者的 `receive(subscription:)`，传入 `Subscription`。随后 `sink` 创建的订阅者通过 `Subscription` 请求 `.unlimited`，表示不限制接收值的数量。

接收需求 `Demand` 表示订阅者允许上游继续发送多少个普通值。`Subscription` 是发布者和订阅者之间的连接，订阅者通过 `Subscription` 请求数量或取消订阅。`AnyCancellable` 是 `sink` 返回给调用方的对象，调用方通过 `AnyCancellable` 取消订阅。

以直接连接的 `PassthroughSubject → sink` 为例：调用方执行 `send(value)` → Subject 将值交给有接收需求的订阅者 → 订阅者执行 `receive(_:)` → `sink` 保存的接收闭包执行。发布者通过订阅关系主动把值推给订阅者，订阅者不需要反复查询是否有新值。

操作符是对事件流进行转换、过滤或组合的接口。链路中的“上游”指提供输入的一侧，“下游”指接收输出的一侧。例如 `subject.map { $0 * 2 }.sink { print($0) }` 会先把输入乘以 2，再把结果交给 `sink`。

`PassthroughSubject → sink` 链路中没有调度或延迟操作符时，接收闭包在调用 `send` 的线程上同步执行。需要异步执行或在主线程执行时，要在链路中加入 `receive(on:)` 等调度操作符。

不同发布者的事件来源不同：Subject 由 `send` 触发；`@Published` 在属性赋值时发布新值；KVO 发布者把受支持的属性观察通知转换成事件流；NotificationCenter 发布者把通知中心收到的通知转换成事件流。KVO 和 NotificationCenter 只是其中两种事件来源。

句柄是调用方用来间接标识、访问或控制某个资源的值，可以是整数、指针，也可以是对象。`AnyCancellable` 常被称为取消句柄，因为调用方通过 `AnyCancellable` 取消订阅。

调用 `cancel()` 或 `AnyCancellable` 被释放时，`AnyCancellable` 取消订阅。需要持续接收事件时，应持有 `sink` 返回的 `AnyCancellable`，例如保存为属性或存入 `Set<AnyCancellable>`。闭包式 `NotificationCenter.addObserver` 返回的 token 被释放时不会取消观察，规则见[移除观察关系与闭包捕获](#移除观察关系与闭包捕获)。

## CombineX 与 NotificationCenter 的选择

CombineX 是构建和组合事件流的框架；NotificationCenter 是把通知分发给匹配观察者的对象。两者解决不同层次的问题，可以配合使用。下表中的 NotificationCenter 指传统的 `post(name:object:userInfo:)` 和 `addObserver` 接口。

| 对比项 | CombineX | 传统 NotificationCenter 接口 |
| --- | --- | --- |
| 接收关系 | 订阅者订阅某个发布者；能否共享一次上游工作取决于发布者和链路。 | 观察者在同一个通知中心按通知名称和可选来源对象匹配通知。 |
| 数据类型 | 通过 `Output` 和 `Failure` 明确普通值与失败的类型。 | 回调接收 `Notification`；`userInfo` 中的业务字段通常需要提取和类型转换。 |
| 事件处理 | 可用 `map`、`filter`、`removeDuplicates` 等操作符组织处理步骤。 | 通知中心负责分发，业务转换和过滤由接收代码完成。 |
| 当前状态 | `CurrentValueSubject`、`@Published` 可以表达当前状态；`PassthroughSubject` 不保存当前值。 | 通知表达一次发生的事情，不自动保存业务当前状态，也不向新观察者补发历史通知。 |
| 通信范围 | 可以向多个订阅者发送值；具体关系由对象和订阅链路决定。 | 可以向多个观察者广播；既可使用 `default`，也可创建独立通知中心。 |

界面需要随状态更新，或数据需要连续执行转换、过滤、组合时，可以使用 CombineX。例如 ViewModel 发布当前计数，View 订阅后刷新显示。

接收系统已经提供的键盘、应用生命周期等通知时，可以使用 NotificationCenter。接收后还需要组合事件流或安排下游调度时，可以用所用框架提供的适配接口，把通知转换为发布者后继续处理。

**系统键盘通知的接入方式：** UIKit 通过 NotificationCenter 发布 `UIResponder.keyboardWillShowNotification` 等系统键盘通知。CombineX 要接收键盘通知，必须从 NotificationCenter 取得通知：UIKit 只把键盘通知发给 NotificationCenter，不会发给业务代码自己创建的 `PassthroughSubject`。

```text
UIKit 发布键盘通知
    → NotificationCenter
    → CombineX 的通知发布者
    → sink 接收闭包
```

通知适配层在 NotificationCenter 上注册观察，把收到的通知传给 CombineX 订阅者。业务代码用 `sink` 接收通知，并可以继续使用操作符。业务代码虽然没有显式调用 `addObserver`，适配层内部仍在 NotificationCenter 上注册了观察。

因此，接收系统键盘通知时，CombineX 依赖 NotificationCenter。键盘通知由 UIKit 发出，接收方只需订阅，不需要自己调用 `post`。

## removeDuplicates() 与连续重复值

无参数的 `removeDuplicates()` 要求输出类型遵循 `Equatable`，用相等比较判断当前值是否与前一个值相同，从而过滤连续重复值。`removeDuplicates()` 每收到一个值就立即比较并决定是否传给下游，没有时间窗口，也不会先收集一段时间的数据再批量去重。

四次发送 `1、1、2、1` 是四个不同的事件，其中前两个事件携带相同的值。直接使用 `sink` 会收到四次；在 `sink` 前添加 `removeDuplicates()` 后，下游依次收到 `1、2、1`。

| 本次输入 | 与前一个值的关系 | 是否传给下游 |
| --- | --- | --- |
| 第一个 `1` | 尚无前一个值 | 传递 `1` |
| 第二个 `1` | 与前一个 `1` 相等 | 不传递 |
| `2` | 与前一个 `1` 不相等 | 传递 `2` |
| 最后的 `1` | 与前一个 `2` 不相等 | 传递 `1` |

`removeDuplicates()` 接收上游发送的每个值，比较后再决定是否传给下游。这里的“重复”指相邻两个事件的值相等，每个事件都只被投递了一次。

为了比较，`removeDuplicates()` 为每次订阅单独保存前一个值。重新建立订阅时，新订阅还没有前一个值，收到的第一个值总会传给下游。

订阅有效期间，两次相同值之间没有其他值到达时，无论两次相隔多久，后一次都会被过滤。`removeDuplicates()` 只保存前一个值，只过滤连续重复的值；之前出现过、但中间隔着其他值的相同值，仍会传给下游。

`CurrentValueSubject` 保存的当前值供读取和新订阅者使用；`removeDuplicates()` 保存的前一个值只用于比较。`CurrentValueSubject` 发送与当前值相同的新值时，订阅者仍会收到；需要过滤时，在链路中加入 `removeDuplicates()`。

下面的示例使用 CombineX，注释为预期输出。

```swift
import CombineX

let subject = PassthroughSubject<Int, Never>()
let subscription = subject
    .removeDuplicates()
    .sink { print($0) }

subject.send(1) // 输出 1
subject.send(1) // 不输出
subject.send(2) // 输出 2
subject.send(1) // 输出 1

subscription.cancel()
```

## RxSwift：输入组合与按钮状态绑定

RxSwift 是组合和处理事件流的 Swift 库，适用于表单校验、搜索输入和多个控件联动。事件流是随时间产生的一系列事件，例如文本依次变成 a、ab、abc。RxCocoa 是 RxSwift 项目中的配套模块，为 UIKit 控件提供 `rx` 扩展，例如 `usernameField.rx.text`、`loginButton.rx.isEnabled`。通用响应式关系见 [响应式关系与 OpenCombine](#响应式关系与-opencombine)；RxSwift 与 Combine 系列库的类型和接口各自独立，不能混用。

下面的示例要求用户名非空、密码至少 6 个字符；两个条件都满足时，登录按钮才可以点击。示例省略了控件布局代码。

```swift
import UIKit
import RxSwift
import RxCocoa

final class LoginViewController: UIViewController {
    let usernameField = UITextField()
    let passwordField = UITextField()
    let loginButton = UIButton(type: .system)
    private let disposeBag = DisposeBag()

    override func viewDidLoad() {
        super.viewDidLoad()

        Observable.combineLatest(
            usernameField.rx.text.orEmpty,
            passwordField.rx.text.orEmpty
        ) { username, password in
            !username.isEmpty && password.count >= 6
        }
        .bind(to: loginButton.rx.isEnabled)
        .disposed(by: disposeBag)
    }
}
```

`.rx.text.orEmpty` 先发出当前文本，之后在用户输入时发出新文本，并把 `nil` 转为空字符串。`rx.text` 监听的是用户编辑产生的控件事件；代码直接给 `text` 属性赋值时，`rx.text` 不一定发出事件。

`combineLatest` 在两个来源都至少发出一个值之后开始工作：此后任一来源发出新值，`combineLatest` 就用两边的最新文本调用闭包。闭包返回 `Bool`，`bind(to:)` 把每次结果写入 `loginButton.isEnabled`。

例如用户名为 xiaoming、密码为 123 时，结果为 false；密码输入为 123456 后，结果为 true；再清空用户名，结果重新变为 false。这个规则只检查输入格式，账号密码是否正确由服务器登录接口验证。

`disposed(by:)` 把本次订阅的取消凭据交给 `disposeBag` 管理。示例把 `disposeBag` 保存为控制器属性；控制器释放时，`disposeBag` 随之释放，并取消 `disposeBag` 管理的全部订阅。页面离开屏幕时，控制器可能还没有释放，订阅也就仍然有效。如果把 `DisposeBag` 声明为方法内的局部变量，方法返回时 `DisposeBag` 被释放，订阅随即被取消。

依赖接入时，需要同时添加 RxSwift 和 RxCocoa，见 [Swift 第三方库与依赖接入](../engineering/xcode-and-testing.md#swift-第三方库与依赖接入)。

## RxSwift：Observable 与订阅

`Observable<Element>` 是 RxSwift 表示可观察事件序列的泛型类。`Element` 指定每次普通数据事件携带的值类型。这里的序列指按时间先后发生的一组事件，可以只有一个值，也可以没有值；序列中的事件在不同时刻陆续到达，数组则一次性保存所有元素。

`Observable<[String]>` 的 `Element` 是 `[String]`。一次普通数据事件可以携带完整数组，例如 `["Swift 入门", "Swift 并发"]`。接收回调只执行一次，参数是整个数组；数组包含两项，回调也只执行一次。

RxSwift 用 `Event<Element>` 表示事件。一次订阅可以收到零次或多次 `.next`，随后正常结束、失败结束，或者持续等待后续事件。

| 事件 | 含义 | 闭包式订阅的接收位置 |
| --- | --- | --- |
| `.next(value)` | 发送一个 `Element` 类型的值 | `onNext`，参数为 `value` |
| `.error(error)` | 以错误结束事件序列 | `onError`，参数为错误 |
| `.completed` | 正常结束事件序列 | `onCompleted`，没有结果参数 |

`.error` 和 `.completed` 都是结束事件。结束事件之后，同一次订阅不再收到 `.next`。搜索返回空数组时，序列发送的是一次 `.next([])`：订阅者收到了一个值，这个值是空数组；序列要结束，还需要另外发送 `.completed`。

订阅是建立“事件产生后通知哪个观察者”的关系，观察者是接收事件的一方。调用 `subscribe(onNext:onError:onCompleted:)` 时，RxSwift 用传入的闭包创建观察者；事件到达观察者后，RxSwift 调用与事件类型对应的闭包，并传入数据或错误。页面在接收闭包中编写显示结果的代码。

下面的示例用固定数组模拟一次搜索结果，不执行网络请求：

```swift
import RxSwift

let resultEvents: Observable<[String]> = Observable.create { observer in
    print("开始产生结果")
    observer.onNext(["Swift 入门", "Swift 并发"])
    observer.onCompleted()
    return Disposables.create()
}

let subscription = resultEvents.subscribe(
    onNext: { items in
        print(items)
    },
    onCompleted: {
        print("搜索结束")
    }
)
```

创建 `resultEvents` 时，`Observable.create` 保存产生事件的闭包。调用 `resultEvents.subscribe(...)` 后，RxSwift 执行产生事件的闭包。`observer.onNext(...)` 发送数组，并触发 `onNext`；`observer.onCompleted()` 随后触发 `onCompleted`。示例中的事件都在 `subscribe(...)` 调用过程中同步发送，因此两个接收闭包在 `subscribe(...)` 返回之前就已执行。

请求启动代码写在订阅时才执行的闭包中（例如 `Observable.create` 的闭包），请求才会在订阅后发出。返回类型 `Observable<[String]>` 只说明每次发送的值是 `[String]`；`search(keyword:)` 是否在订阅时才启动请求、是否在多个订阅者之间共享请求、是否重放已有结果、总共发送多少个数组，都要看 `search(keyword:)` 的实现。

`subscribe(...)` 返回遵循 `Disposable` 协议的取消凭据。调用取消凭据的 `dispose()` 会结束本次订阅，并执行序列登记的清理代码，例如 `Observable.create` 闭包返回的 `Disposable`。普通 `Disposable` 变量被释放时，RxSwift 不会因此调用 `dispose()`，订阅仍然有效；`DisposeBag` 被释放时会对所管理的取消凭据调用 `dispose()`，见 [RxSwift：输入组合与按钮状态绑定](#rxswift输入组合与按钮状态绑定)。

## RxSwift：debounce 与 distinctUntilChanged

操作符是接收一个事件序列、返回新事件序列的方法，新序列中的事件经过了转换或筛选。上游是向操作符发送事件的一方，下游是接收操作符输出的一方。`debounce` 按事件间隔决定何时发送最近值；`distinctUntilChanged` 按值的比较结果过滤连续重复值。

```swift
let keywords = searchText
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
```

示例假设 `searchText` 的类型是 `Observable<String>`，并且 `keywords` 已经被订阅。调度器决定任务在哪个线程或队列、什么时机执行。`MainScheduler.instance` 让 `debounce` 在主线程执行计时到期后的发送工作。

### debounce 的发送时机

上游还在发送事件时，每当新文本事件到达，`debounce` 保存新文本，丢弃上一份待发送文本，并重新开始等待 300 毫秒。等待的 300 毫秒内没有新事件到达时，`debounce` 发送保存的文本一次。主线程繁忙时，实际发送时刻会晚于计划时刻。

例如，0 毫秒输入 `s`，100 毫秒输入 `sw`，200 毫秒输入 `swift`。计时任务正常执行时，`debounce` 在约 500 毫秒发送 `swift`。之后没有新事件，`debounce` 就不再发送，不会每隔 300 毫秒重复发送 `swift`。`debounce` 只看有没有新事件到达，不比较文本内容；上游重复发送相同文本，也会让 `debounce` 重新开始等待。

上游结束时：上游正常完成，RxSwift 的 `debounce` 先发送尚未发送的最近值，再发送完成事件；上游失败，`debounce` 丢弃待发送值并转发错误。

### distinctUntilChanged 的比较范围

无参数的 `distinctUntilChanged()` 要求元素类型遵循 `Equatable`，即支持相等比较。`distinctUntilChanged()` 只保存前一个值，过滤与前一个值相同的值，不保存全部历史值。每次订阅各自保存自己的前一个值。

| 实际到达 distinctUntilChanged 的值 | 输出 |
| --- | --- |
| `swift → swift → swift` | `swift` |
| `swift → swiftq → swift` | `swift → swiftq → swift` |
| `sw → swift → swift` | `sw → swift` |

`swift → swiftq → swift` 中的最后一个 `swift` 会被发送，因为最后一次比较的前一个值是 `swiftq`。通用的连续去重机制见 [removeDuplicates() 与连续重复值](#removeduplicates-与连续重复值)；`removeDuplicates()` 属于 Combine 系列库，`distinctUntilChanged()` 属于 RxSwift，两者不能混用。

`debounce` 位于 `distinctUntilChanged` 前面时，`distinctUntilChanged` 只比较 `debounce` 已经发送的值。例如第一个 `swift` 已经发送，用户随后快速输入 `swiftq` 又删除 `q`，中间没有停顿 300 毫秒。`debounce` 不会发送短暂出现的 `swiftq`；`distinctUntilChanged` 实际收到 `swift → swift`，因此过滤第二个 `swift`。

把 `distinctUntilChanged` 放到 `debounce` 前面时，短暂出现的 `swiftq` 也会参与比较；重复的相同文本先被 `distinctUntilChanged` 过滤，不再到达 `debounce`，也就不会让 `debounce` 重新开始等待。因此，最终输出由原始输入和操作符顺序共同决定。

## RxSwift：observe(on:) 与回调线程

`observe(on:)` 返回一个新的 `Observable`，把经过 `observe(on:)` 的事件通知安排到指定调度器执行。`observe(on: MainScheduler.instance)` 让下游接收事件的代码在主线程执行，直到下游再次改变调度方式。

```swift
search(keyword: keyword)
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { items in
        print(items)
    })
    .disposed(by: disposeBag)
```

示例假设 `search(keyword:)` 返回 `Observable<[String]>`，`disposeBag` 由页面持有。执行顺序为：调用 `search(keyword:)` 取得搜索序列 → 订阅建立后，搜索序列发送结果数组 → `observe(on:)` 把结果通知交给主线程 → RxSwift 在主线程调用 `onNext`，并传入数组。

`observe(on:)` 只切换事件通知的执行位置。`subscribe(...)` 方法调用本身，以及上游的请求启动、网络传输和数据解析，仍在原来的线程执行。订阅和取消订阅动作在哪个调度器执行，由 `subscribe(on:)` 控制；网络库使用的回调队列由网络库的配置决定。

`observe(on:)` 的位置决定转换代码在哪个线程执行。例如 `map` 是对每个值调用转换闭包的方法：放在 `observe(on:)` 后面的 `map`，闭包随事件在指定调度器执行；放在 `observe(on:)` 前面的 `map`，闭包不受后面的调度切换影响。

当前已在主队列上、且满足立即执行条件时，`MainScheduler.instance` 直接在当前调用栈执行任务，不会等到“下一轮”才执行。OpenCombine 的主队列调度器总是异步提交，见 [receive(on:) 与主队列异步调度](#receiveon-与主队列异步调度)；两个库的调度实现不同。

## RxSwift：flatMapLatest 与 flatMap

`flatMapLatest` 对收到的每个输入值调用转换闭包，转换闭包返回一个事件序列。`flatMapLatest` 订阅新返回的序列，同时取消对上一个序列的订阅，因此只转发当前订阅序列的事件。`flatMap` 同样调用转换闭包并订阅返回的序列，但保留尚未结束的旧订阅，所有仍有效搜索的结果都会被转发。

### 返回类型与闭包执行时机

下面的示例假设 `searchText` 是 `Observable<String>`，`search(keyword:)` 返回 `Observable<[String]>`。示例还假设每次搜索在被订阅时启动，成功时发送一个结果数组并正常完成。示例省略了 `search(keyword:)` 的实现。

```swift
let latestResults: Observable<[String]> = searchText
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { keyword -> Observable<[String]> in
        let requestResults = search(keyword: keyword)
            .observe(on: MainScheduler.instance)
        return requestResults
    }

latestResults
    .subscribe(onNext: { items in
        print(items)
    })
    .disposed(by: disposeBag)
```

`flatMapLatest(...)` 把输入序列和转换闭包组合成一个新的 `Observable<[String]>`，`latestResults` 保存这个组合序列。转换闭包每执行一次，局部变量 `requestResults` 就保存本次关键词对应的搜索序列；每次闭包调用中的 `requestResults` 各自对应当次关键词的搜索序列。

`flatMapLatest` 订阅搜索序列，再把搜索序列发出的数组转发给页面，所以页面收到的是 `[String]`。因此返回类型是 `Observable<[String]>`，而不是 `Observable<Observable<[String]>>`。调用 `flatMapLatest` 只组装关系；转换闭包要等订阅建立、关键词到达之后才执行。

在同一次页面订阅中，执行顺序为：

1. 页面订阅 `latestResults`，RxSwift 建立输入处理和结果接收关系。
2. `sw` 通过 `debounce` 与 `distinctUntilChanged`，RxSwift 用 `sw` 调用转换闭包，取得并订阅 `sw` 搜索序列。
3. `swift` 随后通过两个过滤步骤，RxSwift 用 `swift` 调用转换闭包，解除 `sw` 搜索订阅并订阅 `swift` 搜索序列。
4. `swift` 搜索发送数组，结果通知经过主线程调度。`flatMapLatest` 转发数组，页面的 `onNext` 接收数组。

切换搜索时，`latestResults` 本身不变，不需要重新赋值；变化的是页面这次订阅内部持有的搜索订阅。“最新”指最近一个到达 `flatMapLatest` 的关键词：`swift` 还在 `debounce` 中等待时，最新关键词仍是 `sw`，`flatMapLatest` 照常转发 `sw` 的结果。切换之前已经转发给页面的 `sw` 结果，页面已经收到，无法撤回。

两个搜索各发送一次数组时，使用 `flatMap`，`onNext` 会收到两次结果，顺序取决于结果到达的先后。使用 `flatMapLatest`，`onNext` 在整个页面生命周期中也可能收到多次结果：旧搜索可能在切换前就发送了结果，之后的每次搜索也会发送结果。任意一个仍有效的搜索发送错误、且错误没有被处理时，错误会结束组合序列，之后的关键词不再触发这次订阅中的搜索。

### 每次订阅的内部状态

下文的类名来自 RxSwift 的内部实现，用于说明实现分工；业务代码不应依赖这些类，库的不同版本也可能调整这些类。

调用 `flatMapLatest` 得到的是组合序列；每次订阅组合序列时，RxSwift 另外创建一个内部实例管理这次订阅的状态。同一个组合序列可以被订阅多次；没有额外的共享操作时，每次订阅有各自的内部实例，各自管理自己的当前搜索。

| 操作符 | 订阅时创建的内部实例 | 搜索订阅的管理方式 |
| --- | --- | --- |
| `flatMapLatest` | `MapSwitchSink`，继承 `SwitchSink` | 从 `SwitchSink` 继承 `innerSubscription: SerialDisposable`，替换当前取消凭据时取消旧搜索订阅 |
| `flatMap` | `FlatMapSink`，继承 `MergeSink` | 使用 `group: CompositeDisposable` 管理多个取消凭据，新搜索不会自动取消尚未结束的旧搜索订阅 |

`SerialDisposable` 是保存单个当前取消凭据的容器。给 `SerialDisposable.disposable` 赋新值时，容器对旧凭据调用 `dispose()`。`CompositeDisposable` 是管理多个取消凭据的容器，可以一起取消所管理的订阅。

`MapSwitchSink` 收到关键词后，创建新的搜索订阅并替换旧的搜索订阅。每次搜索对应一个 `SwitchSinkIter`；`SwitchSinkIter` 接收本次搜索的事件，并检查本次搜索是否仍应转发。`SwitchSinkIter` 转发给最终观察者的数组，就是页面 `onNext` 收到的参数。

### 取消订阅、取消请求与编号检查

取消搜索订阅后，`flatMapLatest` 不再接收这次搜索的事件。底层网络任务是否停止，取决于 `search(keyword:)` 的取消逻辑是否调用了网络任务的取消方法。网络任务继续运行时，已取消的搜索订阅也不再把网络结果转发给页面。

`flatMapLatest` 的内部实现还保存最新搜索编号。每次搜索的 `SwitchSinkIter` 保存创建时的编号。结果到达后，`SwitchSinkIter` 比较本次搜索编号与最新编号；编号不一致时，`SwitchSinkIter` 不转发结果。

编号检查用于丢弃切换过程中到达的旧事件。更新最新编号和取消旧订阅是两个分开的步骤；关键词事件和搜索结果事件来自不同线程时，可能出现以下交错：

1. 线程 A 收到 `swift`，把最新编号从 `1` 更新为 `2`，随后释放保护编号的锁。
2. 线程 A 尚未执行取消 `sw` 订阅的代码。
3. 线程 B 把 `sw` 结果交给编号为 `1` 的 `SwitchSinkIter`。
4. `SwitchSinkIter` 发现 `1 != 2`，丢弃 `sw` 结果。
5. 线程 A 继续取消 `sw` 订阅，并订阅 `swift` 搜索。

这只是并发时可能出现的一种顺序。编号检查让旧结果在取消完成之前到达时也能被丢弃。已经开始执行的回调，取消动作无法撤回。

事件通知和取消订阅都在同一个串行调度器上执行时，取消操作返回后，被取消的订阅不会再调用接收回调。串行调度器让安排的工作逐个依次执行；关键词通知、搜索结果通知和取消动作都在主线程执行时，上述两个线程的交错不会发生。
