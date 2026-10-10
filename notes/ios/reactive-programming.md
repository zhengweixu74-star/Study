# 响应式编程与事件通知

[返回笔记目录](../README.md)

## 本文目录

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

NotificationCenter 通过 `addObserver` 建立观察关系，通过 `post` 发送通知。闭包观察者的 queue 为 nil 时，回调在发送线程同步执行；注册线程不决定回调线程。UI 更新需要满足主线程要求，指定 main 也不能解释为所有投递都必然异步。

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

`AnyCancellable` 在调用 `cancel()` 或对象销毁时执行取消动作。需要持续接收事件时，应保留 `sink` 返回的对象，例如保存为属性或存入 `Set<AnyCancellable>`。传统闭包式 `NotificationCenter.addObserver` 返回的观察者令牌使用另一套生命周期规则，应通过 `removeObserver` 移除观察关系，不能把令牌释放直接等同于自动取消。

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
