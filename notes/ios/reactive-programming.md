# 响应式编程与事件通知

[返回笔记目录](../README.md)

## 本文目录

- [RxSwift：输入组合与按钮状态绑定](#rxswift输入组合与按钮状态绑定)
- [RxSwift：Observable 与订阅](#rxswiftobservable-与订阅)
- [RxSwift：debounce 与 distinctUntilChanged](#rxswiftdebounce-与-distinctuntilchanged)
- [RxSwift：observe(on:) 与回调线程](#rxswiftobserveon-与回调线程)
- [RxSwift：flatMapLatest 与 flatMap](#rxswiftflatmaplatest-与-flatmap)
- [NotificationCenter 与通知回调](#notificationcenter-与通知回调)
- [响应式关系与 OpenCombine](#响应式关系与-opencombine)
- [事件值、失败与流结束](#事件值失败与流结束)
- [无载荷事件：PassthroughSubject<Void, Never>](#无载荷事件passthroughsubjectvoid-never)
- [@Published：状态订阅与赋值时序](#published状态订阅与赋值时序)
- [receive(on:) 与主队列异步调度](#receiveon-与主队列异步调度)
- [CombineX 的订阅关系与事件传递](#combinex-的订阅关系与事件传递)
- [CombineX 与 NotificationCenter 的选择](#combinex-与-notificationcenter-的选择)
- [removeDuplicates() 与连续重复值](#removeduplicates-与连续重复值)

本篇记录事件通知、发布订阅、订阅生命周期，以及事件流的值、失败与结束。线程、队列、RunLoop 和定时回调的执行机制见[iOS 线程与任务调度](threads-and-scheduling.md)。

## NotificationCenter 与通知回调

`NotificationCenter` 是 Foundation 提供的进程内通知分发对象。发送方把事件交给通知中心，通知中心调用符合条件的观察者回调。回调是观察者注册时提供、由通知中心在事件发生时执行的代码。

一次通知由 `Notification` 表示，主要包含三个字段：

| 字段 | 含义 |
| --- | --- |
| `name` | 通知名称，用于区分事件 |
| `object` | 与事件关联的对象，通常由发送方传入自身；也可以为 `nil` |
| `userInfo` | 可选的附加数据字典，例如新的标题 |

### 注册、匹配与分发

观察者调用 `addObserver(forName:object:queue:using:)`，提交名称、对象过滤条件、执行队列和回调闭包。通知中心保存这次观察关系，并返回用于移除观察关系的标识对象，通常称为 token。

发送方调用同一个通知中心的 `post`。通知中心匹配已注册的观察关系，然后调用对应回调。注册时指定 `name` 就只接收该名称；指定 `object` 就只接收与该对象匹配的通知。两项都指定时，两项都要匹配。某项为 `nil` 表示不按该项过滤。

可以把这套机制理解为“保存观察关系 → 接收通知 → 按条件查找 → 执行回调”。这只是工作模型，不能据此断言系统内部一定使用某一种哈希表。不同的 `NotificationCenter` 实例各自管理观察关系；向一个实例发送通知，不会自动通知另一个实例中的观察者。

普通的通知观察关系不会补发注册之前的历史通知。需要接收某次通知时，应在发送之前完成注册。

### 回调在哪个线程执行

使用上述闭包接口且 `queue` 为 `nil` 时，通知中心在发送通知的线程上同步调用回调。同步表示回调完成后，当前这次 `post` 才返回。注册观察者的线程不决定回调线程；在主线程注册，也可能在后台线程收到回调。

`queue` 的类型是 `OperationQueue?`，用于指定回调的执行队列。传入 `.main` 表示让回调在主队列执行，但不能据此认定 `post` 一定立即返回或所有通知都会异步投递。需要理解任务队列和线程的区别时，见[线程与任务调度](threads-and-scheduling.md#operation-与-operationqueue)。更新 UIKit 界面的代码需要在主线程执行。

下面的示例只有一个匹配的观察者，且回调没有再次发送通知：

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

输出顺序为“发送前 → 收到通知：新标题 → 发送后”。回调较耗时时，同步发送的线程也会等待；回调再次发送通知时，还可能产生嵌套调用。多个观察者的执行顺序不应作为业务逻辑的前提。

### 移除观察关系与闭包捕获

对于这个闭包接口，通知中心会持有观察者 token 和回调闭包。仅把自己保存的 token 设为 `nil`，不会自动取消观察。应在不再需要接收事件时，对注册时使用的通知中心调用 `removeObserver(token)`。

如果闭包强引用了某个对象，通知中心持有闭包也会间接延长该对象的生命周期。例如闭包强引用 `self` 时，不能只依赖 `self` 的 `deinit` 来移除观察，因为该对象可能一直无法释放。可以根据需要使用 `[weak self]`，并在明确的生命周期结束点移除观察关系。弱捕获只改变对象的持有关系，不会自动取消观察。

## 响应式关系与 OpenCombine

**响应式编程：** 先建立“数据／事件发生变化后，哪些代码需要执行”的关系，再由变化触发处理。例如先订阅点赞数变化，收到 13 时把 Label 更新为“13”；或者先监听定位目标，目标准备好后发送，View 收到再安排滚动。普通变量修改不会自动刷新界面，仍需暴露可观察的变化并写好接收处理。

OpenCombine 是开源的 Combine 兼容实现，提供发布、订阅、转换、过滤、组合数据和事件的工具。使用 OpenCombine 类型时导入 `OpenCombine`；使用系统 Combine 类型时导入 `Combine`。

```text
PassthroughSubject.send(_:) 发送值
    → sink 注册的接收闭包收到值
```

**订阅与生命周期：**

- `PassthroughSubject` 将值传给当前有接收需求的订阅者，不保存最近值供后来的订阅者补收。没有订阅者时发送的值会丢弃。
- `.sink { value in ... }` 建立订阅并处理收到的数据。
- 返回的 `AnyCancellable` 用于保持与取消订阅；可以保存在变量中，也可以使用 `.store(in: &cancels)` 存入集合。取消订阅后该订阅者不再接收事件。

## 事件值、失败与流结束

把事件流理解为同一个通道先后发送的一串数据，例如下载进度 `10% → 40% → 70%`。`PassthroughSubject<Output, Failure>` 的第一个类型表示普通数据，第二个表示可能报告的错误类型。

| 发送动作 | sink 中接收的位置 | 之后能否继续收到该流的数据 |
|-|-|-|
| `send(value)` | `receiveValue` | 可以，直到流结束或订阅取消 |
| `send(completion: .finished)` | `receiveCompletion` 的 `.finished` 分支 | 不可以，已正常结束 |
| `send(completion: .failure(error))` | `receiveCompletion` 的 `.failure` 分支 | 不可以，已失败结束 |

下面是示例中的模拟下载进度代码，没有发起真实网络请求，也未编译运行：

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

预期依次输出“下载进度：10%”“下载进度：40%”“下载失败：networkDisconnected”。失败通知进入 `receiveCompletion`，其中的 `error` 是 `DownloadError.networkDisconnected`，不会作为普通值进入 `receiveValue`。前提是订阅仍然有效。

错误是下载业务主动调用 send 报告的；Subject 不会自行检测断网，也不会自动捕获所有业务错误。`.failure` 表示终止该流，不是发送一条错误提示后还能继续发送原流的数据。

若类型改为 `PassthroughSubject<Int, Never>`，普通数据仍是 Int，但 `Never` 没有可构造的值，因此这个通道没有可以报告的失败值。它可以 `send(10)`，也可以 `send(completion: .finished)`；把 `.networkDisconnected` 用作它的失败值会类型不匹配。Never 不表示永不结束，也不表示业务绝不会出错。

## 无载荷事件：PassthroughSubject<Void, Never>

`PassthroughSubject<Output, Failure>` 允许代码主动发送值和结束通知。`Output` 是普通值的类型，`Failure` 是失败结束时携带的错误类型，且必须遵循 `Error`。

`PassthroughSubject<Void, Never>` 表示：普通事件携带空元组 `()`，不带业务字段；该事件流不会以 `.failure` 报告失败。`Never` 不是“发生失败但错误信息为空”，也不会自动捕获或吞掉业务错误。

```swift
import OpenCombine

let refreshSubject = PassthroughSubject<Void, Never>()
let subscription = refreshSubject.sink { _ in
    print("收到刷新事件")
}
refreshSubject.send(())
refreshSubject.send(completion: .finished)
```

`send(())` 仍然发送了一个值，只是这个值没有字段。`Never` 不妨碍正常结束，因此可以发送 `.finished`。示例通过 `subscription` 保持订阅有效。`PassthroughSubject` 不保存最新值供后来的订阅者补收，适合表达“发生了一次操作”。

## @Published：状态订阅与赋值时序

`@OpenCombine.Published` 是属性包装器：声明在类的属性上，为该属性提供发布值的能力。读取 `count` 得到属性当前值；访问 `$count` 得到可订阅的 Publisher，即向订阅者提供值的发布者。因此，已有 `@Published` 属性时，不需要再手动创建 `PassthroughSubject` 才能订阅该属性。

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

直接使用 `sink` 订阅 `$count` 后，订阅者先收到当前值 `0`。执行 `counter.count = 1` 时，`@Published` 先发布新值 `1`，然后更新属性存储。这种顺序称为 `willSet` 语义。因此，赋值期间同步执行的接收闭包拿到 `newValue == 1`，但此时再次读取 `counter.count` 仍得到 `0`。

这里读取的是尚未更新的属性，不是读取 Swift 属性观察器中的 `oldValue` 参数。`oldValue` 是 `didSet` 观察器可用的旧值名称，不能用它解释所有“读到旧值”的情况。

如果链路通过 `receive(on:)` 异步投递到之后才执行的闭包，再次读取属性时，属性可能已经更新，甚至已变成更晚一次赋值的结果。处理某次通知时，应优先使用闭包收到的 `newValue`，因为它对应本次发布的值。

`@Published` 适合表达可观察状态，`PassthroughSubject` 适合表达不需要保存当前值的事件。两者都能被订阅，但初始值、状态保存和发送方式不同，不能仅因都使用 `sink` 就视为相同。

### 状态与事件：为什么都用 sink 仍不相同

状态表示“现在是什么”。例如 count 当前等于 3，即使没有订阅者，这个当前值仍然有意义。可观察状态既保存当前值，也允许订阅者接收赋值时发布的新值。

事件表示“发生了一次什么”。例如按钮被点击一次，后续才开始订阅的对象通常不需要再次收到之前的点击。PassthroughSubject 将事件值发送给当前有接收需求的订阅者，不保留事件值供后续订阅者补收。

| 对比项 | @Published | PassthroughSubject |
|-|-|-|
| 适合表达的数据 | 当前状态，例如当前计数是 3 | 一次事件，例如按钮被点击一次 |
| 初始值 | 属性需要初始化，例如 count = 0 | 创建时不需要初始值 |
| 当前值保存 | 保存当前值，可以直接读取 count | 不保存当前值，也不补发之前发送的普通事件 |
| 发送方式 | 给属性赋值，例如 model.count = 2 | 主动调用 send，例如 model.click.send(()) |
| 直接使用 sink 开始订阅 | 先收到属性的当前值，再接收后续赋值发布的新值 | 不会因订阅自动收到一个初始值；订阅有效期间接收之后发送的新事件 |

“先收到当前值”不等于“总是收到声明时的初始值”。如果 count 从 0 改成 1 后才开始订阅，新订阅者先收到 1。

下面的示例先修改状态、发送事件，再建立订阅。示例沿用本文的 OpenCombine 写法，直接使用 sink，并通过变量保留订阅令牌。注释为预期结果，未编译运行。

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

读取 model.count 得到属性的当前值；访问 model.\$count 得到该属性的发布者，即向订阅者提供值的对象。sink 建立订阅，并在收到值时执行接收闭包。两者都能使用 sink，只说明两者都能向订阅者发送值，不代表两者保存当前值的方式相同，也不代表新订阅者收到值的时机相同。

## receive(on:) 与主队列异步调度

调度器是安排动作在何处、何时执行的对象。`DispatchQueue.main` 表示主队列；主队列的任务在主线程执行。`.ocombine` 把这个队列适配为 OpenCombine 的调度器接口，不会创建另一个队列。

`.receive(on: DispatchQueue.main.ocombine)` 把下游接收值和结束通知的动作安排到主队列。这里“下游”指该操作符之后接收通知的代码；如果后面又使用其他调度操作符，后续执行位置还会再次改变。该调用不会自动把上游网络请求、计算或属性赋值移到主线程。

OpenCombine 的 DispatchQueue 适配器在普通 `schedule` 中调用 `queue.async`。所以这个具体组合会异步提交下游接收动作；即使当前已经在主线程，也会提交任务，而不是直接在当前调用栈执行接收闭包。

未设置延迟不表示同步执行。这里没有指定固定等待时长，主队列在能够执行该任务时调用闭包。这个行为不保证固定延迟、下一帧或下一轮 RunLoop；也不能推广为所有调度器上的 `receive(on:)` 都异步执行。

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

上述顺序的前提是订阅持续有效、主队列能够正常处理任务，且 `A`、`send`、`B` 位于同一个主队列任务中。`send` 提交接收任务后返回，当前主队列任务继续打印 `B` 并结束，主队列才执行接收任务打印 `C`。

如果发送发生在后台线程，主线程可能在发送线程继续执行后续语句之前就接收事件。因此，“异步提交”不能单独证明后台发送代码后面的语句一定先于主线程接收闭包执行。

GCD 基础概念见[iOS 线程与任务调度](threads-and-scheduling.md)。

## CombineX 的订阅关系与事件传递

CombineX 与 OpenCombine 是不同的开源 Combine 兼容实现。下文使用 CombineX 说明发布订阅的公共模型；具体模块名称和平台适配接口应按所用实现区分。

| 对象或协议 | 职责 |
| --- | --- |
| `Publisher` | 发布者协议：接受订阅，并向订阅者提供值或结束通知。 |
| `Subscriber` | 订阅者协议：接收订阅关系、普通值和结束通知。 |
| `Subscription` | 表示一次订阅关系，提供请求接收数量和取消订阅的接口。 |
| `Subject` | 支持主动发送的发布者协议；调用方可以使用 `send` 发送值或结束通知。 |

`sink` 创建一个保存接收闭包的订阅者，并将该订阅者交给上游发布者。发布者建立订阅关系，通过 `receive(subscription:)` 把 `Subscription` 交给订阅者。`sink` 创建的订阅者随后请求 `.unlimited`，表示不限制接收值的数量。

接收需求 `Demand` 表示订阅者允许上游继续发送多少个普通值。`Subscription` 负责传递请求和取消动作；`sink` 返回的 `AnyCancellable` 是供调用方管理取消动作的对象，两者不是同一个概念。

以直接连接的 `PassthroughSubject → sink` 为例：调用方执行 `send(value)` → Subject 将值交给有接收需求的订阅者 → 订阅者执行 `receive(_:)` → `sink` 保存的接收闭包执行。订阅关系使发布者能够找到接收方，不需要接收方反复查询是否有新值。

操作符是对事件流进行转换、过滤或组合的接口。链路中的“上游”指提供输入的一侧，“下游”指接收输出的一侧。例如 `subject.map { $0 * 2 }.sink { print($0) }` 会先把输入乘以 2，再把结果交给 `sink`。

在没有切换调度或延迟操作符的直接 `PassthroughSubject → sink` 链路中，接收闭包在发送线程同步执行。发布订阅本身不表示异步执行，也不自动保证主线程。

不同发布者的事件来源不同：Subject 由 `send` 触发；`@Published` 在属性赋值时发布新值；KVO 发布者把受支持的属性观察通知转换成事件流；NotificationCenter 发布者把通知中心收到的通知转换成事件流。KVO 和 NotificationCenter 都不是所有发布者共同依赖的通知机制。

句柄是调用方用来间接标识、访问或控制某个资源的值。句柄不限定为整数、指针或内存地址。在订阅场景中，把 `AnyCancellable` 称为取消句柄，是强调调用方可以通过该对象取消订阅。

`AnyCancellable` 在调用 `cancel()` 或对象销毁时执行取消动作。需要持续接收事件时，应保留 `sink` 返回的对象，例如保存为属性或存入 `Set<AnyCancellable>`。传统闭包式 `NotificationCenter.addObserver` 返回的 token 使用不同的生命周期规则，见[移除观察关系与闭包捕获](#移除观察关系与闭包捕获)。

## CombineX 与 NotificationCenter 的选择

CombineX 是构建和组合事件流的框架；NotificationCenter 是将通知分发给匹配观察者的通知中心。两者不是同一层次的工具，也可以配合使用。下面的对比限定于 NotificationCenter 的传统 `post(name:object:userInfo:)` 和 `addObserver` 接口。

| 对比项 | CombineX | 传统 NotificationCenter 接口 |
| --- | --- | --- |
| 接收关系 | 订阅者订阅某个发布者；能否共享一次上游工作取决于发布者和链路。 | 观察者在同一个通知中心按通知名称和可选来源对象匹配通知。 |
| 数据类型 | 通过 `Output` 和 `Failure` 明确普通值与失败的类型。 | 回调接收 `Notification`；`userInfo` 中的业务字段通常需要提取和类型转换。 |
| 事件处理 | 可用 `map`、`filter`、`removeDuplicates` 等操作符组织处理步骤。 | 通知中心负责分发，业务转换和过滤由接收代码完成。 |
| 当前状态 | `CurrentValueSubject`、`@Published` 可以表达当前状态；`PassthroughSubject` 不保存当前值。 | 通知表达一次发生的事情，不自动保存业务当前状态，也不向新观察者补发历史通知。 |
| 通信范围 | 可以向多个订阅者发送值；具体关系由对象和订阅链路决定。 | 可以向多个观察者广播；既可使用 `default`，也可创建独立通知中心。 |

界面需要随状态更新，或数据需要连续执行转换、过滤、组合时，可以使用 CombineX。例如 ViewModel 发布当前计数，View 订阅后刷新显示。

接收系统已经提供的键盘、应用生命周期等通知时，可以使用 NotificationCenter。若接收后还需要组合事件流或安排下游调度，可以通过所用框架提供的适配接口，把通知转换为发布者后继续处理。

**系统键盘通知的接入边界：** 监听 UIKit 发布的 `UIResponder.keyboardWillShowNotification` 等系统键盘通知时，CombineX 需要通过 NotificationCenter 接入事件。仅创建 `PassthroughSubject` 并调用 `sink` 建立订阅，不会让 UIKit 自动向该 Subject 发送键盘事件。

```text
UIKit 发布键盘通知
    → NotificationCenter
    → CombineX 的通知发布者
    → sink 接收闭包
```

通知适配层负责观察 NotificationCenter 的通知，并把收到的通知传给 CombineX 订阅者。业务代码可以使用 `sink` 接收通知，并继续使用操作符；业务代码没有显式调用 `addObserver`，不表示这条链路不依赖 NotificationCenter。

这里“不能单独使用 CombineX”限定为：不能仅靠创建发布者和建立订阅，就绕过通知中心取得这条系统键盘通知。通知由 UIKit 发出，接收方不需要自己调用 `post`。

## removeDuplicates() 与连续重复值

无参数的 `removeDuplicates()` 要求输出类型遵循 `Equatable`，使用相等比较过滤连续重复值。该操作符没有默认时间窗口，不会先收集一段时间的数据再批量去重。

四次发送 `1、1、2、1` 是四个不同的事件，其中前两个事件携带相同的值。直接使用 `sink` 会收到四次；在 `sink` 前添加 `removeDuplicates()` 后，下游依次收到 `1、2、1`。

| 本次输入 | 与前一个值的关系 | 是否传给下游 |
| --- | --- | --- |
| 第一个 `1` | 尚无前一个值 | 传递 `1` |
| 第二个 `1` | 与前一个 `1` 相等 | 不传递 |
| `2` | 与前一个 `1` 不相等 | 传递 `2` |
| 最后的 `1` | 与前一个 `2` 不相等 | 传递 `1` |

操作符仍然接收上游发送的值，只是在比较后决定是否继续传给下游。因此，“重复”指相邻事件的值相等，不表示某一次发送被系统重复投递。

为完成比较，`removeDuplicates()` 为经过该操作符的每次订阅分别保存比较所需的前一个值。保存比较状态是该操作符的需要，不是所有订阅都会自动保存历史值。重新建立订阅时，新订阅从没有比较值的状态开始。

只要订阅仍然有效，且中间没有不同的值到达操作符，即使两次相同值相隔很久，后一次仍会被过滤。`removeDuplicates()` 只判断连续值；不会像集合一样记住所有出现过的值。

`CurrentValueSubject` 保存当前状态，与 `removeDuplicates()` 保存比较状态是两件事。使用 `CurrentValueSubject` 本身不会自动过滤相同值；需要过滤时仍应添加相应操作符。

以下最小示例使用 CombineX。注释说明预期结果，示例未编译运行。

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

RxSwift 是组合和处理事件流的 Swift 库，适用于表单校验、搜索输入和多个控件联动。事件流是随时间产生的一系列事件，例如文本依次变成 a、ab、abc。RxCocoa 是同一项目中提供 UIKit 控件相关能力的配套模块。通用响应式关系见 [响应式关系与 OpenCombine](#响应式关系与-opencombine)；不同库的类型和接口不能直接混用。

下面要求用户名非空，密码至少包含 6 个字符；两个条件都满足时，登录按钮才可以点击。示例省略控件布局，仅展示数据与按钮状态的关系。

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

`.rx.text.orEmpty` 提供初始文本及后续用户输入变化，并将 nil 转为字符串空值。不能据此假定直接给 text 属性赋值也一定发出同样的输入事件。

`combineLatest` 等两个来源都提供值后，每当任一来源产生新值，就用两边的最新文本执行闭包。闭包产生 Bool 值，`bind(to:)` 建立接收关系，将每次结果写入按钮的 isEnabled。

例如用户名为 xiaoming、密码为 123 时，结果为 false；密码输入为 123456 后，结果为 true；再清空用户名，结果重新变为 false。这个规则只判断输入是否符合示例条件，不能验证账号密码是否正确，实际登录仍需服务器验证。

`disposed(by:)` 将本次订阅的取消对象交给 DisposeBag 管理。示例将 DisposeBag 保存为控制器属性；控制器和 DisposeBag 释放后，DisposeBag 取消所管理的订阅。页面离开屏幕不一定表示控制器已经释放。将 DisposeBag 仅声明为短生命周期的局部变量，会让订阅过早被取消。

依赖接入时，需要同时添加 RxSwift 和 RxCocoa，见 [Swift 第三方库与依赖接入](../engineering/xcode-and-testing.md#swift-第三方库与依赖接入)。

## RxSwift：Observable 与订阅

`Observable<Element>` 是 RxSwift 表示可观察事件序列的泛型类。`Element` 指定每次普通数据事件携带的值类型。序列表示按时间先后发生的事件，不要求产生多个值，也不等于数组。

`Observable<[String]>` 的 `Element` 是 `[String]`。一次普通数据事件可以携带完整数组，例如 `["Swift 入门", "Swift 并发"]`。接收回调在一次调用中取得整个数组，不会因为数组包含两项而自动执行两次。

RxSwift 用 `Event<Element>` 表示事件。一次订阅可以收到零次或多次 `.next`，随后正常结束、失败结束，或者持续等待后续事件。

| 事件 | 含义 | 闭包式订阅的接收位置 |
| --- | --- | --- |
| `.next(value)` | 发送一个 `Element` 类型的值 | `onNext`，参数为 `value` |
| `.error(error)` | 以错误结束事件序列 | `onError`，参数为错误 |
| `.completed` | 正常结束事件序列 | `onCompleted`，没有结果参数 |

`.error` 和 `.completed` 都是结束事件。结束事件之后，同一次订阅不能继续收到 `.next`。搜索返回空数组仍然是一次 `.next([])`，不等于没有事件，也不等于 `.completed`。

订阅是建立“事件产生后通知哪个观察者”的关系。观察者是接收事件的一方。调用 `subscribe(onNext:onError:onCompleted:)` 时，RxSwift 用调用方传入的闭包建立观察者；事件到达观察者后，RxSwift 调用匹配的闭包，并传入数据或错误。页面通常通过接收闭包使用结果，`Observable` 不会代替页面执行显示逻辑。

下面的示例模拟一次结果发送，不执行网络请求。示例仅说明执行关系，未编译运行。

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

创建 `resultEvents` 时，`Observable.create` 保存产生事件的闭包。调用 `resultEvents.subscribe(...)` 后，RxSwift 执行产生事件的闭包。`observer.onNext(...)` 发送数组，并触发 `onNext`；`observer.onCompleted()` 随后触发 `onCompleted`。示例没有安排异步执行，因此接收闭包可以在 `subscribe(...)` 返回之前执行。

只有把请求启动代码放在订阅时执行的逻辑中，才能说“订阅后才发出请求”。仅凭返回类型 `Observable<[String]>`，无法确定 `search(keyword:)` 是否延迟启动请求、是否共享请求、是否重放已有结果，以及总共发送多少个数组。

`subscribe(...)` 返回遵循 `Disposable` 协议的取消凭据。调用取消凭据的 `dispose()` 会结束本次订阅并执行相应清理。`DisposeBag` 的管理方式见 [RxSwift：输入组合与按钮状态绑定](#rxswift输入组合与按钮状态绑定)。释放一个普通 `Disposable` 变量不应被当作已经调用 `dispose()`。

## RxSwift：debounce 与 distinctUntilChanged

`debounce` 按事件间隔决定何时发送最近值。`distinctUntilChanged` 按值的比较结果过滤连续重复值。操作符是接收一个事件序列、返回经过转换或筛选的新事件序列的方法；上游是向操作符发送事件的一方，下游是接收操作符输出的一方。

```swift
let keywords = searchText
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
```

示例假设 `searchText` 的类型是 `Observable<String>`，并且 `keywords` 已经被订阅。调度器决定工作在什么执行环境、什么时机运行。`MainScheduler.instance` 把定时到期后的工作安排到主线程。

### debounce 的发送时机

上游保持有效期间，每次新文本事件到达，`debounce` 都会保留新文本并重新开始等待 300 毫秒。等待期间再次收到文本事件时，`debounce` 放弃上一份待发送文本。等待时间到期且没有更新的文本事件时，`debounce` 发送最近文本一次。实际执行可能因为主线程繁忙而延后。

例如，0 毫秒输入 `s`，100 毫秒输入 `sw`，200 毫秒输入 `swift`。假设计时任务正常执行，`debounce` 在约 500 毫秒发送 `swift`。之后没有新事件时，`debounce` 不会每隔 300 毫秒重复发送 `swift`。判断依据是有没有新事件，不是文本框的外观是否变化；上游重复发送相同文本也属于新事件。

上述等待规则针对仍在产生事件的上游。上游正常完成时，RxSwift 的 `debounce` 会先发送尚未发送的最近值，再发送完成事件；上游失败时，`debounce` 丢弃待发送值并转发错误。

### distinctUntilChanged 的比较范围

无参数的 `distinctUntilChanged()` 要求元素类型遵循 `Equatable`，即支持相等比较。`distinctUntilChanged()` 过滤连续相同的值，不保存全部历史值。每次新订阅有各自的比较状态。

| 实际到达 distinctUntilChanged 的值 | 输出 |
| --- | --- |
| `swift → swift → swift` | `swift` |
| `swift → swiftq → swift` | `swift → swiftq → swift` |
| `sw → swift → swift` | `sw → swift` |

`swift → swiftq → swift` 中的最后一个 `swift` 会被发送，因为最后一次比较面对的是 `swiftq`。通用的连续去重机制见 [removeDuplicates() 与连续重复值](#removeduplicates-与连续重复值)；`removeDuplicates()` 和 `distinctUntilChanged()` 属于不同库的 API，不能混写。

当 `debounce` 位于 `distinctUntilChanged` 前面时，比较范围是 `debounce` 已经发送的值。例如第一个 `swift` 已经发送，用户随后快速输入 `swiftq` 又删除 `q`，中间没有停顿 300 毫秒。`debounce` 不会发送短暂出现的 `swiftq`；`distinctUntilChanged` 实际收到 `swift → swift`，因此过滤第二个 `swift`。

如果把 `distinctUntilChanged` 放到 `debounce` 前面，短暂出现的 `swiftq` 也会参与比较；相同文本事件是否重新启动等待也可能改变。操作符顺序会影响行为，不能仅凭原始输入判断最终输出。

## RxSwift：observe(on:) 与回调线程

`observe(on:)` 返回一个新的 `Observable`，将经过 `observe(on:)` 的事件通知安排到指定调度器。`observe(on: MainScheduler.instance)` 让接收这些事件的后续代码在主线程执行，直到后续代码再次改变调度方式。

```swift
search(keyword: keyword)
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { items in
        print(items)
    })
    .disposed(by: disposeBag)
```

示例假设 `search(keyword:)` 返回 `Observable<[String]>`，`disposeBag` 由页面持有。调用 `search(keyword:)` 取得搜索序列。订阅建立后，搜索序列发送结果数组。`observe(on:)` 将结果通知交给主线程执行，随后 RxSwift 调用 `onNext`，并传入数组。

切换的是事件通知的执行位置，不是 `subscribe(...)` 方法调用本身。`observe(on:)` 也不会把已经位于上游的请求启动、网络传输或数据解析自动搬到主线程。订阅和取消订阅动作的调度由 `subscribe(on:)` 控制；网络库自己使用的回调队列仍由网络库的配置决定。

`observe(on:)` 的位置也会影响转换代码在哪个线程执行。例如 `map` 是对每个值调用转换闭包的方法：放在 `observe(on:)` 后面的 `map` 闭包随事件在指定调度器执行；放在 `observe(on:)` 前面的 `map` 闭包不受后面的调度切换影响。

`MainScheduler.instance` 在主队列上且满足立即执行条件时，可以直接执行任务，不能把 `observe(on: MainScheduler.instance)` 一概理解为“下一轮才执行”。Combine 系列的调度行为另见 [receive(on:) 与主队列异步调度](#receiveon-与主队列异步调度)，不能把不同库的具体调度实现视为相同。

## RxSwift：flatMapLatest 与 flatMap

`flatMapLatest` 对收到的每个输入值执行转换闭包。转换闭包返回一个事件序列。`flatMapLatest` 订阅新返回的序列，并解除对上一个返回序列的订阅；随后只转发当前订阅的事件。`flatMap` 也执行转换和订阅，但会保留尚未结束的旧订阅，并转发所有仍有效搜索的结果。

### 返回类型与闭包执行时机

下面的示例假设 `searchText` 是 `Observable<String>`，`search(keyword:)` 返回 `Observable<[String]>`。示例还假设每次搜索在被订阅时启动，成功时发送一个结果数组并正常完成。示例展示组合关系，省略搜索函数实现，未编译运行。

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

`flatMapLatest(...)` 返回一个新的 `Observable<[String]>`，并把原始输入序列和转换闭包组合起来。`latestResults` 保存组合后的序列。每次转换闭包执行时，局部变量 `requestResults` 保存本次关键词对应的搜索序列；不同闭包调用中的 `requestResults` 可以指向不同搜索序列。

`flatMapLatest` 将搜索序列发出的数组转交给页面，所以页面收到 `[String]`，不是搜索序列。返回类型是 `Observable<[String]>`，不是 `Observable<Observable<[String]>>`。调用 `flatMapLatest` 组装关系时，不会仅因此执行每个关键词的转换闭包；转换闭包在订阅建立且关键词到达后执行。

在同一次页面订阅中，执行顺序为：

1. 页面订阅 `latestResults`，RxSwift 建立输入处理和结果接收关系。
2. `sw` 通过 `debounce` 与 `distinctUntilChanged`，RxSwift 用 `sw` 调用转换闭包，取得并订阅 `sw` 搜索序列。
3. `swift` 随后通过两个过滤步骤，RxSwift 用 `swift` 调用转换闭包，解除 `sw` 搜索订阅并订阅 `swift` 搜索序列。
4. `swift` 搜索发送数组，结果通知经过主线程调度。`flatMapLatest` 转发数组，页面的 `onNext` 接收数组。

`latestResults` 不需要重新赋值。变化发生在当前页面订阅所管理的搜索订阅中。“最新”指已经到达 `flatMapLatest` 的关键词；如果 `swift` 仍在等待 `debounce`，`flatMapLatest` 仍可能转发 `sw` 的结果。切换之前已经转发的 `sw` 结果不会被撤回。

若两个搜索分别发送一次数组，`flatMap` 可以让 `onNext` 接收两次结果，顺序取决于结果到达顺序。`flatMapLatest` 也不保证整个页面生命周期只接收一次：旧结果可能在切换前已经发送，后续搜索也可以继续发送结果。任意一个仍有效的搜索失败且没有被处理时，错误会结束组合后的序列，之后的关键词不再触发本次订阅的搜索。

### 每次订阅的内部状态

以下内部类名用于说明 RxSwift 的实现分工，属于库的内部实现，不是调用方应依赖的公开接口；不同版本可能调整内部结构。

调用 `flatMapLatest` 得到的组合序列，与订阅建立时创建的状态管理实例，需要分别理解。一个组合序列可以被订阅多次；没有额外共享操作时，各次订阅分别管理自己的当前搜索，不会共用唯一的当前搜索属性。

| 操作符 | 订阅时创建的内部实例 | 搜索订阅的管理方式 |
| --- | --- | --- |
| `flatMapLatest` | `MapSwitchSink`，继承 `SwitchSink` | 从 `SwitchSink` 继承 `innerSubscription: SerialDisposable`，替换当前取消凭据时取消旧搜索订阅 |
| `flatMap` | `FlatMapSink`，继承 `MergeSink` | 使用 `group: CompositeDisposable` 管理多个取消凭据，新搜索不会自动取消尚未结束的旧搜索订阅 |

`SerialDisposable` 是保存单个当前取消凭据的容器。给 `SerialDisposable.disposable` 赋新值时，容器对旧凭据调用 `dispose()`。`CompositeDisposable` 是管理多个取消凭据的容器，可以一起取消所管理的订阅。

`MapSwitchSink` 负责收到关键词后创建并切换搜索订阅。每次搜索对应一个 `SwitchSinkIter`，`SwitchSinkIter` 接收本次搜索事件并检查是否仍应转发。转发到最终观察者的数组会成为页面 `onNext` 的参数。

### 取消订阅、取消请求与编号检查

解除搜索订阅表示结束对应的事件接收关系。底层网络任务是否停止，取决于搜索实现是否在取消逻辑中调用网络任务的取消方法。即使网络任务继续运行，也不代表已经取消的页面订阅应该继续收到网络结果。

`flatMapLatest` 的内部实现还保存最新搜索编号。每次搜索的 `SwitchSinkIter` 保存创建时的编号。结果到达后，`SwitchSinkIter` 比较本次搜索编号与最新编号；编号不一致时，`SwitchSinkIter` 不转发结果。

编号检查用于拒绝切换过程中到达的旧事件。更新最新编号与取消旧订阅是分开的步骤。根据执行顺序，在关键词事件和搜索结果事件来自不同线程时，可能出现以下交错：

1. 线程 A 收到 `swift`，把最新编号从 `1` 更新为 `2`，随后释放保护编号的锁。
2. 线程 A 尚未执行取消 `sw` 订阅的代码。
3. 线程 B 把 `sw` 结果交给编号为 `1` 的 `SwitchSinkIter`。
4. `SwitchSinkIter` 发现 `1 != 2`，丢弃 `sw` 结果。
5. 线程 A 继续取消 `sw` 订阅，并订阅 `swift` 搜索。

这是一种并发执行的可能顺序，不表示每次切换都会出现，也不表示必须在取消完成后才检查编号。取消动作也不能撤回已经开始执行的回调。

如果事件通知和取消订阅都在同一个串行调度器上执行，取消操作返回后，被取消的订阅不会再调用接收回调。串行调度器让安排的工作依次执行；当关键词通知、搜索结果通知和取消动作都在主线程执行时，上述两线程交错不适用。
