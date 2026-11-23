# iOS 线程与任务调度

[返回笔记目录](../README.md)

## 本文目录

- [进程与线程](#进程与线程)
  - [内核的职责与进程信息](#内核的职责与进程信息)
  - [CPU 核心、逻辑 CPU 与并行上限](#cpu-核心逻辑-cpu-与并行上限)
  - [并发、并行与等待](#并发并行与等待)
  - [线程接口与任务调度接口](#线程接口与任务调度接口)
- [GCD 的任务、队列与提交方式](#gcd-的任务队列与提交方式)
  - [sync 的执行机制](#sync-的执行机制)
  - [async 的执行机制](#async-的执行机制)
  - [barrier 与单写多读](#barrier-与单写多读)
  - [全局并发队列为什么没有屏障效果](#全局并发队列为什么没有屏障效果)
  - [Swift 单写多读示例](#swift-单写多读示例)
  - [Thread.sleep 的作用与边界](#threadsleep-的作用与边界)
  - [死锁、直接崩溃与系统终止](#死锁直接崩溃与系统终止)
  - [数据竞争、对象生命周期与 UI 线程要求](#数据竞争对象生命周期与-ui-线程要求)
- [DispatchGroup 与 DispatchSemaphore](#dispatchgroup-与-dispatchsemaphore)
  - [任务登记与完成通知](#任务登记与完成通知)
  - [信号量的初始值与配对规则](#信号量的初始值与配对规则)
  - [同时限制访问数量并跟踪整体完成](#同时限制访问数量并跟踪整体完成)
- [Operation 与 OperationQueue](#operation-与-operationqueue)
  - [并发限制、依赖与执行顺序](#并发限制依赖与执行顺序)
  - [最小使用示例](#最小使用示例)
  - [取消是协作式的](#取消是协作式的)
  - [异步下载不能只套普通闭包限流](#异步下载不能只套普通闭包限流)
- [i++ 的执行过程与线程安全](#i-的执行过程与线程安全)
- [CAS 的机制、重试与 ABA](#cas-的机制重试与-aba)
- [常见锁与分类维度](#常见锁与分类维度)
  - [获取锁失败后的等待行为](#获取锁失败后的等待行为)
  - [自旋等待、内核态与线程阻塞](#自旋等待内核态与线程阻塞)
  - [优先级反转与优先级继承](#优先级反转与优先级继承)
- [RunLoop 与运行模式](#runloop-与运行模式)
  - [Source0 与 Source1：事件来源、数量与回调](#source0-与-source1事件来源数量与回调)
  - [Source0 示例：后台压缩完成后刷新预览](#source0-示例后台压缩完成后刷新预览)
  - [Source1 示例：通过 Mach 端口接收完成消息](#source1-示例通过-mach-端口接收完成消息)
- [Timer 的调度与重复触发](#timer-的调度与重复触发)
  - [Timer 的停止、引用与重新创建](#timer-的停止引用与重新创建)
- [CADisplayLink 的原理与适用场景](#cadisplaylink-的原理与适用场景)
  - [按经过时间计算动画进度](#按经过时间计算动画进度)
  - [停止回调与强引用关系](#停止回调与强引用关系)
  - [弱引用代理](#弱引用代理)
  - [运行模式与工作量](#运行模式与工作量)
  - [用途与取舍](#用途与取舍)

## 进程与线程

进程是操作系统为运行程序创建的运行实例，拥有相应的地址空间和资源。线程是进程中由内核安排执行的单位。一个进程可以包含多个线程；CPU 实际执行的是被调度线程的指令。

同一进程中的线程共享进程的地址空间和资源。每条线程有自己的栈和执行上下文。执行上下文是线程暂停时需要保存的状态，包括寄存器的值和下一条要执行的指令地址。内核切换线程时，先把当前线程的寄存器值存到内存，再把下一条线程保存的值装回寄存器，所以多条线程可以轮流使用同一组物理寄存器。

### 内核的职责与进程信息

内核是操作系统中负责管理硬件资源、进程和访问权限的核心程序。内核是软件，CPU 和内存是硬件；内核在 App 启动之前就已经运行。

- 进程管理：维护进程及其编号、创建时间等信息。
- CPU 调度：从可以运行的线程中选择线程，为线程安排 CPU 执行机会。
- 内存管理：维护进程的地址空间与访问边界，防止进程随意访问其他进程的内存。
- 资源访问控制：检查程序对受保护资源的访问权限。

App 可以通过系统接口向内核查询进程信息。查询在 App 运行期间进行，查到的信息可以是更早发生的事件，例如进程的创建时间。

### CPU 核心、逻辑 CPU 与并行上限

物理核心是 CPU 内部能够执行指令的硬件单元。逻辑 CPU 是操作系统能够独立调度线程的硬件执行上下文。一个物理核心可以提供一个或多个逻辑 CPU，具体取决于硬件能力和启用配置。

如果设备有 L 个可用的逻辑 CPU，同一时刻最多有 L 个线程正在执行。这个上限限制的是线程，不是进程：这 L 个线程可以属于同一个进程，也可以属于不同进程。

假设设备有 4 个物理核心，每个核心只提供一个逻辑 CPU，且 4 个逻辑 CPU 均可用。同一时刻可以出现以下分配：

| 线程分配 | 正在执行代码的进程数量 | 正在执行的线程数量 |
| --- | --- | --- |
| 4 个进程各执行 1 个线程 | 4 个 | 4 个 |
| 2 个进程各执行 2 个线程 | 2 个 | 4 个 |
| 1 个进程执行 4 个线程 | 1 个 | 4 个 |

超线程是 Intel 的同时多线程技术。启用超线程后，一个物理核心向操作系统提供两个逻辑 CPU，所以线程并行的上限要按逻辑 CPU 计算。但同一核心上的两个逻辑 CPU 共享这个核心的执行单元，逻辑 CPU 数量翻倍，性能并不会翻倍。

### 并发、并行与等待

并发是多个任务在同一段时间内都有进展；并行是多个任务在同一时刻都在执行。并发可以靠交替执行实现，也可以包含并行。单个逻辑 CPU 通过线程切换可以并发推进多个任务，但同一时刻只能执行一个线程。

系统中的进程和线程可以远多于逻辑 CPU。就绪线程已经可以运行，只是还没分到 CPU；阻塞线程在等待网络、文件等事件，事件发生前不能运行。内核按调度策略从就绪线程中选择执行，各线程分到的执行时间不一定相同。

多创建线程不会增加 CPU。线程数量超过逻辑 CPU 数量时，多出的线程要排队等待；有线程阻塞时，内核把 CPU 让给其他就绪线程。

### 线程接口与任务调度接口

`pthread` 是提供线程创建、等待和同步等操作的 C 接口，Foundation 的 `Thread` 是面向对象的线程接口。使用这两种接口时，应用要自己创建线程、指定线程执行的代码，并管理线程何时结束。

使用 GCD 和 `OperationQueue` 时，应用只提交任务，由系统决定用哪条线程执行。Swift Concurrency 是 Swift 的异步任务与隔离机制，用 `Task`、`async`、`await` 等写异步流程，任务挂起时释放线程。

| 方式 | 应用主要管理的对象 | 适用需求 |
| --- | --- | --- |
| `pthread`、`Thread` | 线程及线程上的工作 | 明确需要控制线程生命周期或常驻线程 |
| GCD | 任务闭包及队列 | 简单提交工作、串行访问和回到主线程 |
| `OperationQueue` | `Operation` 任务及依赖 | 管理依赖、并发数量和协作式取消 |
| Swift Concurrency | 异步任务及隔离范围 | 等待异步结果、结构化子任务和保护共享状态 |

任务、队列和线程的数量之间没有固定的对应关系。串行队列只保证同一时刻最多执行一个任务，不同任务可以由不同的后台线程执行；并发队列允许多个任务同时推进，但不会给每个任务分配一条专用线程。

下面用 GCD 把计算提交到全局队列，算完后把结果交给主队列。实际项目中可以在主队列闭包里更新 UIKit 界面，这里只打印结果。

```swift
import Foundation

DispatchQueue.global(qos: .userInitiated).async {
    let total = (1...1_000).reduce(0, +)
    DispatchQueue.main.async {
        print(total) // 500500，在主线程执行
    }
}
```

后台闭包先算出 `total`，再把打印闭包提交给主队列；提交后立即返回，打印要等主线程空闲时才执行。`.userInitiated` 是服务质量等级，系统据此安排优先级，它不指定线程。

`Task { ... }` 不一定在后台执行：在 `MainActor` 隔离范围内创建的 `Task` 会继承这个隔离，其中的同步计算仍在主线程执行。`async let` 在声明处就创建子任务，后面的 `await` 只是等待结果。具体机制和两个请求的示例见[async let 创建子任务与 await 等待结果](../language/swift-basics.md#async-let-创建子任务与-await-等待结果)。

## GCD 的任务、队列与提交方式

GCD（Grand Central Dispatch）是 Apple 提供的任务调度机制，由 libdispatch 库实现。任务是要执行的工作，队列决定任务按什么规则执行，线程负责实际执行代码。队列不是线程；除主队列外，队列一般不固定使用某条线程。

在 Swift 的 GCD 调用中，block 指传给 sync 或 async 的闭包，闭包的内容就是任务要执行的代码。这里的“任务”是一般概念，与 Swift 的 [Task](../language/swift-basics.md#tasktaskdetached-与任务局部值) 类型无关。

| 概念 | 对执行的约束 |
| --- | --- |
| sync 同步提交 | 调用方等待闭包执行完成后，调用才返回 |
| async 异步提交 | 调用方提交闭包，不等待闭包执行完成 |
| 串行队列 | 同一时刻最多执行队列中的一个任务 |
| 并发队列 | 允许多个任务重叠执行，不保证一定同时执行 |
| 主队列 | 特殊的串行队列，任务始终在主线程执行 |

同步／异步描述调用方是否等待，串行／并发描述队列能否同时执行多个任务，两者可以任意组合。DispatchQueue.async 是提交闭包的方法，与 Swift 并发中声明异步函数的 async 关键字无关。

```swift
import Foundation

let serialQueue = DispatchQueue(label: "example.serial")
let concurrentQueue = DispatchQueue(
    label: "example.concurrent",
    attributes: .concurrent
)
```

### sync 的执行机制

```swift
print("A")
serialQueue.sync {
    print("B")
}
print("C")
// 顺序固定为 A → B → C
```

sync 提交时，libdispatch 先检查队列当前能否执行这个任务。能执行时，一般直接在调用 sync 的线程上执行闭包，省去线程切换；不能执行时，调用线程等待，之后闭包可能在调用线程执行，也可能在其他线程执行。提交给主队列的任务一定在主线程执行。

sync 不会“切换到队列对应的线程”。主线程用 sync 向普通队列提交耗时任务时，闭包往往就在主线程执行；即使闭包在其他线程执行，主线程也要等它结束，界面同样会卡住。

sync 只等闭包返回。如果闭包只是发起一个异步网络请求，闭包返回时请求还没完成，sync 不会等待网络响应。

### async 的执行机制

```swift
print("A")
serialQueue.async {
    print("B")
}
print("C")
// A 最先输出，B 与 C 的先后没有保证
```

async 把闭包连同捕获的内容保存下来，交给队列安排执行，然后立即返回。普通队列的任务由系统管理的工作线程执行，线程会被复用，每次 async 不一定创建新线程。提交到主队列的任务在主线程执行。

向并发队列异步提交 A、B、C、D 时，可能的一种安排是：线程 1 先执行 A 再执行 C，线程 2 先执行 B 再执行 D。A 与 B 同时执行；一条线程可以先后执行多个任务，但同一时刻只执行一个。并发队列不保证所有任务同时开始，使用的线程数量也不固定。

上例中 B 与 C 的先后不确定，是因为 B 可能在另一条线程上执行：B 有可能在 async 返回之前就开始，也可能在 C 之后才开始。async 不会打断当前线程正在执行的代码。

### barrier 与单写多读

单写多读是指：多个读取可以同时进行；同一时刻最多只有一个写入；写入期间，其他读取和写入都要等待。“单写”限制的是同时进行的写入数量，写入可以由任意线程提交。

barrier（屏障）任务是提交到自建并发队列时带 `.barrier` 标记的任务。队列先等屏障之前提交的任务全部结束，再单独执行屏障任务；屏障任务结束后，才继续执行后面的任务。读取用普通任务提交、写入用屏障任务提交，就实现了单写多读。

使用条件：

- 对这个变量的所有读取和写入都必须提交到同一个队列。屏障只约束同一队列中的任务，不知道闭包访问了哪个变量，也挡不住其他队列直接访问这个变量。
- 写入必须在屏障闭包返回前完成。如果闭包只是发起另一个异步操作，屏障闭包返回后，这个异步操作就不再受屏障保护。

### 全局并发队列为什么没有屏障效果

DispatchQueue.global() 返回系统提供的共享并发队列，进程中所有模块都可以使用。向全局并发队列提交 async(flags: .barrier) 时，这个任务按普通异步任务执行，没有屏障效果。只有自建的并发队列支持屏障。

屏障会让同一队列中的所有任务等待，而不只是访问某个变量的任务。假设图片解码、读取 value、屏障写入 value、写日志这四个任务依次提交到全局队列：如果全局队列支持屏障，写入就要等图片解码和读取结束，写日志也要等写入结束，与 value 无关的图片解码和日志也被拖慢。

可以这样理解：全局队列不支持屏障，避免了一个模块的屏障让其他模块的任务等待。这是根据效果做出的解释，Apple 文档只规定了全局队列没有屏障效果，没有说明原因。

所以应创建一个专门负责访问 value 的并发队列，屏障只约束这个队列中的读写任务，其他队列的任务不受影响。自建队列也不会占用专用线程；屏障只控制队列中任务的执行时机，不会暂停其他线程。

### Swift 单写多读示例

下面的代码放在 Swift 命令行程序的 main.swift 中运行。读取和写入都异步提交，通过打印观察执行顺序。代码用短暂休眠延长每个任务的时间，便于看到读取同时进行、写入单独执行。

```swift
import Foundation

final class SingleWriterDemo: @unchecked Sendable {
    private var value = 0
    private let queue = DispatchQueue(
        label: "demo.reader-writer",
        attributes: .concurrent
    )

    func read(_ name: String) {
        queue.async {
            print("\(name) 开始读取")
            Thread.sleep(forTimeInterval: 0.5)
            print("\(name) 读取结束，value = \(self.value)")
        }
    }

    func write(_ newValue: Int) {
        queue.async(flags: .barrier) {
            print("开始写入 \(newValue)")
            Thread.sleep(forTimeInterval: 0.5)
            self.value = newValue
            print("写入结束")
        }
    }

    // 仅从该队列之外调用，用于命令行示例等待已提交任务。
    func waitUntilFinished() {
        queue.sync(flags: .barrier) {}
    }
}

let demo = SingleWriterDemo()
demo.read("读任务 A")
demo.read("读任务 B")
demo.write(100)
demo.read("读任务 C")
demo.read("读任务 D")
demo.waitUntilFinished()
```

执行顺序：A、B 读取结束 → 写入 100 → C、D 开始读取。A、B 都读到 0，C、D 都读到 100。A 和 B 同时读取，但两者谁先开始、谁先结束不确定。连续提交多个屏障写入时，写入之间也不会同时执行。

write 返回只表示写入任务已经提交，写入不一定完成。waitUntilFinished 在已提交的任务之后再提交一个同步屏障，调用方等这个屏障结束后才返回，防止命令行程序提前退出。这个方法不等待之后新提交的任务；也不能在 queue 正在执行的任务中调用，否则任务会等待自己所在的队列，形成死锁。

@unchecked Sendable 表示由开发者保证这个类可以安全地在线程间传递，编译器不检查（Sendable 的含义见 [Sendable 与 @Sendable](../language/swift-basics.md#sendable-与-sendable)）。示例的安全性来自私有队列和屏障：value 只在 queue 中被访问；声明 @unchecked Sendable 本身不提供任何同步。

### Thread.sleep 的作用与边界

Thread.sleep(forTimeInterval: 0.5) 让执行这行代码的线程休眠 0.5 秒。休眠期间，这个线程不执行后续代码，也不占用 CPU，其他线程照常执行。0.5 秒到期后，线程还要等系统调度才能继续，所以实际恢复时间会稍晚。

读取任务休眠时，另一个读取任务照常执行。屏障写入任务休眠时，屏障闭包还没结束，所以同一队列后面的任务都要等待。这种等待是屏障造成的，sleep 只让当前线程休眠，不会暂停整个队列。

示例中的 sleep 只用来模拟耗时，与同步无关；删掉 sleep，单写多读的顺序依然成立。不要在主线程调用 sleep 做界面延时，主线程休眠期间无法处理任何界面事件。

### 死锁、直接崩溃与系统终止

死锁是多个任务互相等待、谁都无法继续执行的状态。死锁不一定立即崩溃：libdispatch 能检测出部分死锁并直接终止进程，没检测出的死锁会一直卡住。主线程卡住太久时，系统的 watchdog 还会终止应用；watchdog 是系统监控应用是否及时完成启动、响应等必要工作的机制。

在主线程向主队列同步提交任务：主线程等待这个任务结束，而这个任务只能由主线程执行，于是主线程在等自己：

```swift
// 错误示例：当前在主线程执行
DispatchQueue.main.sync {
    print("任务")
}
```

串行队列中的任务向同一个队列同步提交任务，也会死锁：外层任务等内层任务结束，内层任务要等外层任务结束才能开始执行：

```swift
let queue = DispatchQueue(label: "example.serial")
queue.async {
    queue.sync {
        print("内层任务")
    }
}
```

libdispatch 发现当前线程已经在执行目标队列的任务时，会主动触发崩溃，崩溃信息中可能包含“dispatch_sync called on queue already owned by current thread”。

两个不同的串行队列也会形成循环等待：

```swift
let queueA = DispatchQueue(label: "example.a")
let queueB = DispatchQueue(label: "example.b")
queueA.async {
    queueB.sync {
        queueA.sync {
            print("无法执行到这里")
        }
    }
}
```

A 的外层任务等 B 的任务结束，B 的任务等 A 的内层任务结束，A 的内层任务又要等 A 的外层任务结束，形成一个环。libdispatch 可能检测出来并终止进程，也可能一直卡住。注意这类问题只针对串行队列：向并发队列中同一个队列普通 sync 提交任务，通常不会以这种方式死锁。判断是否死锁，要看实际的等待关系。

只用 async 也可能死锁。例如主线程用 async 提交后台任务，然后用信号量等待后台任务的结果；后台任务又用 sync 向主队列提交任务。信号量是让线程等待其他任务发出信号的同步工具。此时主线程在等后台任务，后台任务在等主线程，双方都无法继续。

### 数据竞争、对象生命周期与 UI 线程要求

数据竞争是多个线程在没有同步的情况下访问同一块内存，且至少有一个线程在写入。多个并发任务同时修改同一个数组变量，可能导致结果错误、内存损坏或崩溃；Array 的值语义只保证复制后互不影响，不保证同一个变量可以被并发修改。

sync 只让调用方等待，不会让并发队列中的任务互斥：不同线程分别向同一个并发队列调用 sync，几个闭包仍可能同时执行。用串行队列保护共享数据的条件见 [i++ 的执行过程与线程安全](#i-的执行过程与线程安全)。

异步闭包还要考虑捕获对象的生命周期。默认的强捕获让对象保持存活；unowned 不保持对象存活，对象释放后再通过 unowned 访问会触发运行时错误；weak 在对象释放后变成 nil，对 nil 强制解包同样会崩溃。复制 UnsafePointer 等指针，不会延长指向内存的有效期，内存释放后再读写指针会非法访问内存。

在后台线程调用必须在主线程执行的 UIKit API，可能导致显示异常、数据损坏或崩溃，而且不一定每次都立即崩溃。这些 UI 操作要放到主线程执行。Thread Sanitizer 用于发现数据竞争，Address Sanitizer 用于发现释放后访问等内存错误，Main Thread Checker 用于检查系统 API 的主线程要求。

## DispatchGroup 与 DispatchSemaphore

DispatchGroup 把一组任务当作一个整体跟踪，用来判断组内登记的任务是否全部完成。DispatchSemaphore 用一个计数决定 wait 调用能否通过，可以用来等待另一个任务发出完成信号，也可以限制同时访问资源的任务数量。两者都不会创建线程。

| 对比项 | DispatchGroup | DispatchSemaphore |
| --- | --- | --- |
| 管理内容 | 尚未完成的一组任务 | 可用执行名额以及等待信号的调用 |
| 登记与完成 | enter() 登记任务，leave() 标记任务完成 | wait() 获取名额，signal() 提供名额或唤醒一个等待者 |
| 等待条件 | 登记的任务全部完成 | 当前这次 wait() 获得一个名额 |
| 等待方式 | notify 注册完成闭包，不阻塞调用方；wait 在组不为空时阻塞调用线程 | wait 在没有可用名额时阻塞调用线程，有名额时直接通过 |
| 数据保护 | 不自动让组内任务互斥，不自动保护共享数据 | 初始值为 1，且所有相关访问正确配对使用同一个信号量时，可以提供互斥访问 |

这里的阻塞是指当前线程停在 wait 调用处，不能执行后面的代码。带超时的 wait 超时后返回 `.timedOut`，这时任务并没有完成，也没有取得名额。

### 任务登记与完成通知

用 queue.async(group: group) 提交闭包时，GCD 自动登记这项任务，闭包返回后自动标记完成。手动管理时，在异步工作开始前调用 enter()，工作真正结束后调用且只调用一次 leave()；成功、失败等每一条结束路径都要调用 leave()。

notify(queue: .main) 注册一个在组内任务全部完成后执行的闭包。notify 立即返回，不等待；组内任务全部完成后，GCD 把这个闭包提交到主队列执行。如果注册时组内已经没有任务，闭包会立即被安排执行，所以要先登记任务，再注册 notify。

.main 只决定通知闭包在哪个队列执行，不改变组内任务在哪个队列执行。group.wait() 则让调用线程阻塞，直到组内任务全部完成。如果组内任务还需要这个被阻塞的线程或串行队列才能完成，就会死锁；相关机制见[死锁、直接崩溃与系统终止](#死锁直接崩溃与系统终止)。

### 信号量的初始值与配对规则

信号量的初始值与线程数量无关。如果所有受限的工作都先调用 wait、结束后调用一次 signal，初始值 N 表示最多有 N 个工作同时处在 wait 与 signal 之间。初始值为 1 时，同一时刻只有一个访问者，可以保护需要独占修改的数据；初始值大于 1 时，多个访问者同时访问，不能保护这类数据。

初始值为 0 表示一开始没有名额，wait 要等到有人调用 signal 才能通过。信号量只影响调用了 wait 的代码，所以另一段不调用 wait 的代码可以正常执行，并在完成后调用 signal。

```swift
import Foundation

let semaphore = DispatchSemaphore(value: 0)

DispatchQueue.global().async {
    print("执行任务 A")
    semaphore.signal()
}

DispatchQueue.global().async {
    semaphore.wait()
    print("A 已发出信号，继续执行任务 B")
}
```

如果 B 先执行到 wait，B 所在的线程等待，直到 A 调用 signal。如果 A 先调用 signal，信号量记下一个名额，B 之后执行到 wait 时直接通过。两个闭包谁先开始不确定，但 B 的打印一定在 A 调用 signal 之后。

如果初始值为 0，而所有任务都写成“wait → 执行工作 → signal”，又没有其他代码调用 signal，那么所有任务都会卡在 wait：负责调用 signal 的代码在 wait 之后，永远执行不到。

### 同时限制访问数量并跟踪整体完成

下面在 iOS 应用中提交 5 个任务，最多允许 2 个任务同时执行 wait 与 signal 之间的同步工作。group 跟踪这 5 个闭包的完成情况。

```swift
import Foundation

let group = DispatchGroup()
let semaphore = DispatchSemaphore(value: 2)
let queue = DispatchQueue.global()

for index in 1...5 {
    queue.async(group: group) {
        semaphore.wait()
        defer { semaphore.signal() }
        print("执行任务 \(index)")
    }
}

group.notify(queue: .main) {
    print("5 个任务全部完成")
}
```

初始值 2 不表示只用两条线程：其他线程也可能已经开始执行闭包，只是停在 wait 处等待。这个例子中，group 负责全部完成后的通知，信号量负责限制同时执行的数量。

如果闭包只是发起另一个异步操作，闭包返回时操作还没结束。要跟踪操作真正完成，应在发起前调用 enter，在操作的完成回调中调用 leave；要限制同时进行的操作数量，也应在完成回调中调用 signal，而不是闭包一返回就释放名额。每次 enter 必须对应一次 leave，每次 wait 必须对应一次 signal。

## Operation 与 OperationQueue

OperationQueue 是 Foundation 提供的任务调度队列。Operation 表示一个任务，应用把工作封装成 Operation 加入队列，队列根据任务是否就绪、优先级和并发上限安排执行。

BlockOperation 是用闭包封装工作的 Operation 子类，普通的同步任务直接使用 BlockOperation 即可；只有需要自己管理执行状态（例如异步任务）时，才需要自定义 Operation 子类。OperationQueue 不是线程，加入任务也不会每次创建新线程。

### 并发限制、依赖与执行顺序

`maxConcurrentOperationCount` 限制队列中同时执行的 Operation 数量。例如设为 2，最多同时执行两个，其余任务等待。这个值与线程数、CPU 核心数无关。

设为 1 时，同一时刻只执行一个 Operation，但执行顺序不一定等于加入顺序：队列会先选择已就绪的任务，依赖和优先级都会影响选择。queuePriority 只影响就绪任务之间谁先开始，不会打断已经开始的任务。

taskB.addDependency(taskA) 让 taskB 等 taskA 完成后才开始。依赖只规定先后：taskA 失败了 taskB 也会开始，taskA 的结果或错误也不会自动传给 taskB，taskB 要自己判断是否继续。

自己创建的 OperationQueue 在后台线程执行任务，OperationQueue.main 在主线程执行任务。图片计算可以在后台队列完成，UIKit 视图更新应回到主线程。

### 最小使用示例

```swift
import Foundation

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2

let taskA = BlockOperation {
    print("执行任务 A")
}
let taskB = BlockOperation {
    print("执行任务 B")
}

queue.addOperations([taskA, taskB], waitUntilFinished: false)
```

上例创建两个同步任务，队列最多同时执行两个。waitUntilFinished 为 false 表示 addOperations 立即返回，不等任务完成；A 和 B 的打印顺序不确定。闭包执行完，对应的任务就完成了。

### 取消是协作式的

`cancel()` 只是把 Operation 的 isCancelled 设为 true。协作式取消是指：任务的代码要自己检查 isCancelled，在可以停下的地方主动结束；cancel() 不会强行中断正在执行的代码。

Operation 在开始前就被取消时，会直接跳过工作并结束。任务开始后，要在每个耗时步骤之前检查 isCancelled。如果正在调用一个不支持取消的同步函数（例如解码函数），只能等它返回，再停止后续步骤或丢弃结果。自定义 Operation 还要在取消后正确设置结束状态。

加入 OperationQueue 的任务不能直接从队列中移除，只能调用 cancel() 取消指定任务，或调用 cancelAllOperations() 取消全部任务；被取消的任务在结束前仍留在队列中。

### 异步下载不能只套普通闭包限流

在普通 BlockOperation 中调用 URLSessionTask.resume() 后，闭包通常很快返回，但网络下载仍在进行。BlockOperation 此时已经完成，OperationQueue 接着启动下一个任务。所以这种写法下，maxConcurrentOperationCount 限制不了同时进行的下载数量。

要限制下载并发，可以让图片加载器维护正在执行的下载数量和待调度请求，在网络完成回调中释放名额；也可以使用正确实现异步状态的 Operation，在网络任务真正结束后才设置完成状态。下载取消仍需等待实际结束，不能立即释放名额。

下载与解码分别限流、预取需求和取消流程见[长列表图片任务调度](image-loading-decoding-and-memory.md#长列表图片任务调度)。

## i++ 的执行过程与线程安全

本节用 `i++` 表示普通整数自增。多个线程同时对同一个变量自增时，普通自增不是原子操作。原子操作是指在其他线程看来不可分割的操作：要么完全没开始，要么已经全部完成，看不到中间状态。

自增分三步：读取旧值 → 计算旧值加 1 → 写回新值。CPU 把变量读入寄存器，在寄存器中做加法，再把结果写回变量的地址。寄存器是 CPU 内部供指令直接使用的存储位置。读写可能先发生在 CPU 缓存中，不一定每次都访问内存。

这三步不一定对应三条机器指令，具体指令取决于 CPU 架构和编译优化。即使某个平台用一条指令完成内存自增，这条指令也不一定是原子的。

**丢失更新示例：** 初始值为 0。线程 A 和 B 都读到 0，各自算出 1，再各自写回 1，两次自增最后只加了 1。这只是说明风险的一种交错；在 Swift 等语言中，数据竞争属于未定义行为，结果不一定是 1 或 2，程序也可能崩溃。

线程安全是指多个线程按约定访问同一份数据时，结果仍然正确，数据保持一致。下面几种方式的保护范围和使用条件各不相同，用了其中某个 API 不代表整个对象就线程安全了。

| 线程安全方式 | 保证条件 |
| --- | --- |
| 避免共享可变数据 | 数据不可变，或每个任务使用独立数据；复制容器时还要检查内部是否继续共享可变对象 |
| 互斥锁 | 用同一把锁包住“读取、计算、写回”整个过程，所有读写这个变量的代码都使用这把锁。常用 API 有 NSLock、pthread_mutex、OSAllocatedUnfairLock 和 Mutex，注意各自支持的系统版本 |
| 原子自增 | 使用 Synchronization.Atomic 等提供的原子“读取—修改—写入”操作；分别做一次原子读取和一次原子写入，整个自增仍不是原子的 |
| 串行队列 | 所有读写都提交到同一个串行队列，队列之外不能直接读写这个变量；sync 本身不提供互斥 |
| actor / MainActor | 变量由 actor 隔离保护；需要保持一致的“检查 + 更新”要写在同一段没有 await 的代码中 |
| 自建并发队列与 barrier | 所有读写通过同一个自建并发队列，写入用屏障任务，规则见[barrier 与单写多读](#barrier-与单写多读) |
| 初始值为 1 的信号量 | 所有读写使用同一个信号量：先 wait，再访问，最后 signal 一次；wait 可能阻塞线程 |

Swift 整数的 count += 1 同样不是原子操作。“检查后修改”必须整个放在同一个保护范围内：只保护写入、不保护读取，或者把检查和更新分别加锁，都可能出错。actor 在 await 前后状态可能改变，见[actor 的默认隔离与重入](../language/swift-basics.md#actor-的默认隔离与重入)。

用 let 保存类实例，只固定了引用，实例的 var 属性仍可被修改。Objective-C 的 atomic 属性只保证单次读取或赋值不被打断，不保证“先读再改”整体原子，也不保证多个属性之间一致。

单独的计数器可以用原子自增；需要多个相关字段同时保持一致时，用锁或串行队列更合适。另外，线程安全只保证结果不出错，读取最终计数前，还要等两个自增任务都执行完。

## CAS 的机制、重试与 ABA

CAS（Compare-And-Swap，比较并交换）是一个原子操作：调用方提供变量地址、期望值和新值；变量当前值等于期望值时，写入新值并返回成功；不相等时，不写入并返回失败。比较和写入作为一个整体完成，中间不会被其他线程插入。

例如，A、B 都读到 0。A 用 CAS 把 0 改成 1，成功；B 也以 0 为期望值执行 CAS，此时当前值已是 1，失败。B 重新读到 1，算出 2，再执行 CAS；如果又被其他线程抢先，就继续重试。

CAS 每次只做一次比较和写入，失败后的重试循环由调用方自己写。自己写 `if 当前值 == 期望值 { 赋值 }` 不是 CAS，因为比较和赋值之间可能插入其他线程的修改，必须使用语言或库提供的原子 API。

**ABA 问题：** 值从 A 变成 B 又变回 A，CAS 只比较当前值，发现不了中间的变化。如果算法需要知道值是否被改过，可以给值加一个版本号，每次修改都加 1，让 CAS 同时比较值和版本号。另外，竞争激烈时 CAS 会反复失败重试，持续消耗 CPU。

## 常见锁与分类维度

锁的各种名称描述的是不同方面：“互斥”描述同一时刻允许几个线程访问；“自旋”描述拿不到锁时怎样等待；“可重入”描述同一线程能否再次获取自己已经持有的锁。所以一把锁可以同时是互斥锁和自旋锁。

| 类型 | 行为 | 条件与代价 |
|-|-|-|
| 互斥锁 | 同一时刻只允许一个持有者访问受保护区域 | 适合共享可变状态；具体实现可能先短暂自旋，再阻塞等待 |
| 自旋锁 | 获取失败时持续尝试，等待期间占用 CPU | 等待极短时可能减少休眠与唤醒开销；长时间等待会浪费 CPU |
| 读写锁 | 多个读者可同时持有读锁，写锁与其他读写锁互斥 | 读多写少时可能受益，实际效果取决于锁开销与竞争程度 |
| 递归锁／可重入锁 | 同一线程可以重复获取自己持有的锁 | 每次成功获取都需要对应释放，计数归零才完全解锁 |

乐观和悲观描述的是并发控制策略：悲观策略先加锁，再访问数据；乐观策略不加锁，先读取和计算，提交更新时再检查有没有冲突，有冲突就重试或放弃。CAS 可以用来实现乐观策略，但只是其中的一步。

### 获取锁失败后的等待行为

锁被其他线程持有时，调用方是否等待、怎样等待，取决于调用的是哪种获取接口。同一把锁通常同时提供下面几种接口。

| 获取方式 | 无法立即获取锁时的行为 | 适用场景 |
| --- | --- | --- |
| 阻塞式获取 | 调用暂不返回，线程按锁的实现等待，成功获取后继续执行 | 当前操作必须取得锁才能继续 |
| 尝试获取（try-lock） | 立即返回失败，调用方决定跳过、稍后重试或执行其他工作 | 不希望当前线程等待 |
| 带超时的获取 | 在约定期限内等待；成功获取或等待超时后返回 | 允许等待，但需要限制等待时间 |

对于会让线程休眠的阻塞式互斥锁，可以按以下顺序理解：线程 B 尝试获取线程 A 持有的锁 → B 进入等待 → A 释放锁 → 锁的实现唤醒符合条件的等待线程 → 被唤醒线程获得调度后继续获取锁。

阻塞等待期间，线程不会反复检查锁状态，不占用 CPU；有些实现会在阻塞前先短暂自旋一会儿。线程被唤醒后还要重新竞争锁，不一定能拿到；只有获取接口成功返回，才表示已经持有锁。

被唤醒的线程还要等 CPU 调度才能运行。所以“锁被释放”“等待线程被唤醒”“等待线程开始执行受保护代码”是先后三个时刻。

### 自旋等待、内核态与线程阻塞

纯用户态自旋锁用原子操作反复尝试获取锁，获取和等待都不需要系统调用。原子操作由 CPU 指令直接完成，不需要进入内核。

用户态和内核态是 CPU 执行代码时的两种权限级别：用户态代码不能直接访问受保护的资源，内核态代码可以。系统调用是应用请求内核服务的入口：同一条线程从用户态进入内核执行代码，完成后再回到用户态。

进入内核态、线程阻塞、线程切换是三件不同的事。线程阻塞是线程暂停运行，等待某个条件满足；线程切换是调度器停下当前线程，改为执行另一条线程。一次不需要等待的系统调用，进入内核后直接返回，线程既不阻塞，也不切换。

| 等待方式 | 等待线程做什么 | CPU 使用与调度 |
| --- | --- | --- |
| 自旋等待 | 持续检查锁状态或重试原子操作 | 线程获得执行机会时持续消耗 CPU |
| 让出执行机会 | 调用调度接口，允许调度器安排其他线程 | 不等于已经进入等待锁释放的阻塞状态，也不保证锁持有者马上执行 |
| 阻塞等待 | 通过锁或等待接口等待条件满足 | 等待期间不持续轮询；条件满足并恢复可运行状态后，仍需等待调度 |

“循环等待”只说明代码在重复执行，不能判断是否进入内核：循环中只有原子操作时，一直在用户态；循环中调用了休眠、让出或阻塞等待接口时，就会进入内核。很多锁先短暂自旋，再转为阻塞。另外，即使自旋代码没有系统调用，执行期间也可能因中断或被抢占而进入内核。

### 优先级反转与优先级继承

优先级反转是指高优先级线程在等待低优先级线程持有的锁，结果实际上比中等优先级的线程执行得还晚。等待一把已被占用的互斥锁时，线程处于阻塞状态，无论优先级多高都不能运行。

设单个逻辑 CPU 上运行 H、M、L 三条线程，优先级满足 H > M > L，并使用按优先级抢占的调度策略。L 先持有一把锁；H 随后尝试获取同一把锁并阻塞；此时 M 变为就绪并获得 CPU。M 虽然不需要这把锁，却可以持续延迟 L 的执行，进而延迟 L 解锁和 H 恢复。

H 优先级再高，在等锁期间也不能运行，调度器只会从可以运行的线程中选择；提高 H 的优先级也没有用，因为 H 要等的是 L 释放锁。

优先级继承是缓解优先级反转的机制：高优先级线程等待锁时，锁临时把持有者的优先级提高到等待者的水平。上例中，L 临时获得 H 的优先级，于是优先于 M 运行，执行完受保护代码并释放锁，H 就能继续运行。

L 释放锁后恢复原来的优先级。优先级继承只是让 L 尽快运行，不会让受保护代码执行得更快，也解决不了死锁。并非所有锁都支持优先级继承：os_unfair_lock、pthread_mutex 支持，DispatchSemaphore 不支持，因为信号量不知道是哪个线程会调用 signal。

## RunLoop 与运行模式

RunLoop 是在线程上反复等待事件、再分发事件的循环。UIKit 应用的主线程 RunLoop 由框架创建并运行；其他线程默认不运行 RunLoop，需要接收事件（例如 Timer）时才自己启动。没有事件时，RunLoop 让线程休眠；当前模式中没有任何输入源和 Timer、到达运行期限或收到停止请求时，RunLoop 的这次运行结束。某个回调执行很久，会占住线程，后面的事件都要等。

运行模式（mode）是一组输入源和 Timer 的集合，RunLoop 每次运行只处理一个模式中的事件。`default` 是一个具体的模式；`common` 不是模式，而是一个标记：注册到 `common` 的输入源或 Timer，会被加入所有标记为 common 的模式中。

主线程在列表滚动期间切换到 tracking 模式。只注册在 default 模式的 Timer，滚动期间不会触发；注册到 common 后，因为 tracking 模式也标记为 common，滚动期间也能触发。common 并不包含所有模式；注册到 common 也不会换线程，主线程繁忙时回调照样会延迟。

### Source0 与 Source1：事件来源、数量与回调

输入源（source）是加入 RunLoop 的事件来源对象，每个输入源记录事件怎样就绪，以及就绪后调用哪个回调。Source0 和 Source1 是 CFRunLoopSource 的两种版本，数字 0 和 1 只是版本号，与优先级和执行顺序无关。

| 比较项 | Source0 | Source1（Apple 平台的 Mach 端口源） |
|-|-|-|
| 创建时的版本 | version = 0 | version = 1 |
| 事件就绪方式 | 调用方或框架手动标记 source | 对应 Mach 端口收到消息 |
| 等待中的线程如何被唤醒 | Signal 本身不唤醒；需要及时处理时，再唤醒目标 RunLoop | 端口消息使内核中相关的等待满足唤醒条件 |
| 回调由谁执行 | 正在运行且当前模式包含该 source 的 RunLoop，在所属线程执行 | RunLoop 收到对应端口消息后，在所属线程执行 source 的回调 |
| CFRunLoopSourceSignal 的效果 | 标记指定的 Source0 为待处理 | 不用于触发 Source1，对 Source1 无效 |

一个 RunLoop 可以注册任意多个 Source0 和 Source1，数量只受内存、端口等资源限制。一个模式可以包含多个 source；同一个 source 可以加入多个模式，在同一个模式中重复添加同一个 source 不会产生副本。

CFRunLoopSourceSignal(refreshSource) 只标记 refreshSource，不影响 RunLoop 中的其他 Source0。refreshSource 必须有效，并且属于 RunLoop 当前的模式，才会被处理。Timer 是另一类 RunLoop 对象，不属于 Source0 或 Source1。

回调在创建输入源时就设置好了。Source0 使用 CFRunLoopSourceContext，其中 version 为 0，perform 是处理回调。Source1 使用 CFRunLoopSourceContext1，其中 getPort 返回端口，perform 处理收到的消息。用 CFMachPort 创建 Source1 时，框架内部先接收消息，再调用创建 CFMachPort 时注册的回调。

发送方只负责发出就绪标记或消息，不需要传递回调函数。RunLoop 找到对应的 source，再调用 source 上注册的回调。回调在哪个线程执行，取决于处理这个 source 的 RunLoop 属于哪个线程。

### Source0 示例：后台压缩完成后刷新预览

场景：后台线程压缩图片，完成后通知主线程刷新预览。主线程事先创建 refreshSource，把刷新逻辑设为 perform 回调，并把 refreshSource 加入主线程 RunLoop 的 common 模式。下面只展示后台线程的触发代码，省略 source 的创建、图片数据的保存与同步，以及销毁清理。

```swift
// 后台任务已完成，refreshSource 已注册到主线程 RunLoop
CFRunLoopSourceSignal(refreshSource)
CFRunLoopWakeUp(CFRunLoopGetMain())
```

1. 后台线程先把压缩结果保存好（并做好同步），再调用 CFRunLoopSourceSignal(refreshSource)，把 refreshSource 标记为待处理。
2. 后台线程调用 CFRunLoopWakeUp(CFRunLoopGetMain())。主线程如果在休眠，会被唤醒；如果正在执行其他代码，这次调用不会打断它。
3. 主线程 RunLoop 执行到处理 Source0 的阶段，并且当前模式包含 refreshSource 时，调用 perform 回调。
4. perform 回调读取压缩结果，在主线程刷新预览。

Signal 只做标记，不会立即执行 perform，也不会把数据传给回调。标记不是计数器：处理之前连续 Signal 多次，回调可能只执行一次。如果每项任务都要处理，应把任务数据另外保存在一个加锁的队列中，由回调逐项取出。

Swift 可以直接调用 CFRunLoopSourceSignal、CFRunLoopWakeUp 这些 Core Foundation 的 C 函数，原理见[Swift 与 C 接口互操作](../language/swift-basics.md#swift-与-c-接口互操作)。只有自己创建的 Source0 才需要手动 Signal；使用框架封装好的功能时，框架会负责标记和唤醒。CFRunLoopWakeUp 要唤醒的是处理这个 source 的 RunLoop（这里是主线程的），而不是发送线程自己的 RunLoop。

### Source1 示例：通过 Mach 端口接收完成消息

Mach 端口是内核管理的消息通信端点，与 TCP/UDP 的端口号无关。发送方持有端口的发送权限，接收方持有接收权限，内核把发到端口的消息交给接收方。每条消息由头部和消息体组成，头部记录接收端口、消息编号等信息。

仍以“后台压缩完成后刷新预览”为例。主线程创建 CFMachPort 并注册接收回调，再用 CFMachPortCreateRunLoopSource 创建关联这个端口的 Source1，加入主线程 RunLoop。下面只描述事件经过的步骤，不是完整的 Mach 通信代码。

1. 主线程 RunLoop 在相关模式中等待消息。
2. 后台线程完成压缩，按双方约定向端口发送 Mach 消息，例如把 msgh_id 设置为 1001，表示压缩完成。
3. 消息到达端口后，内核唤醒正在等待这个端口消息的主线程。主线程什么时候真正开始运行，还要看线程调度。
4. 主线程 RunLoop 根据收到消息的端口找到 Source1，调用对应的回调。回调根据消息编号和数据判断发生了什么，然后刷新预览。

这个过程不需要调用 CFRunLoopSourceSignal，发送方也不需要调用 CFRunLoopWakeUp：消息传递和唤醒由内核完成，回调由主线程的 RunLoop 执行。主线程繁忙或当前模式不包含这个 source 时，回调会延迟。

消息编号写在消息头里，与 Source1 无关。每条 Mach 消息的头部 mach_msg_header_t 都有 msgh_id 字段；1001 是本例中发送方和接收方约定的值，系统不规定它的含义。

例如，同一个端口、同一个 Source1 可以接收 1001（压缩完成）、1002（保存完成）、1003（任务取消）三种消息。处理分两步：RunLoop 先按接收端口找到 source；回调再按消息编号选择处理分支。

## Timer 的调度与重复触发

Timer 保存触发时间和回调，RunLoop 负责在触发时间到达后执行回调，内核的定时器负责在时间到达时唤醒线程。系统不会为每个 Timer 创建线程计时，RunLoop 也不会反复检查时间是否已到。

1. scheduledTimer 把 Timer 加入当前线程 RunLoop 的 default 模式，记录首次触发时间、重复间隔和回调。
2. 在 Apple 开源的 Core Foundation 实现中，RunLoop 用 mach_absolute_time 的计数表示触发时间，并调用 mk_timer_arm 向内核设置一个定时通知。
3. RunLoop 没有事件要处理时，调用 mach_msg 让线程休眠。时间到达后，内核发送消息唤醒线程。内核发出通知、线程分到 CPU、回调开始执行，是先后三个时刻。
4. 线程被唤醒后，RunLoop 找出已经到期的 Timer，在这个线程上执行回调。主线程繁忙、当前模式不包含这个 Timer，都会让回调延迟。
5. 重复 Timer 按最初的时间表安排下一次。原定在 1、2、3 秒触发，第一次延迟到 1.3 秒、且回调很快结束，第二次仍在 2 秒触发。如果一次延迟跨过了好几个触发点，错过的回调不会补执行，Timer 只执行一次，再按时间表安排下一个尚未到达的触发点。

设置 tolerance 允许系统把多个 Timer 的唤醒合并在一起以节省电量，因此回调可能在 tolerance 范围内稍晚执行。第 2、3 步的函数来自 Apple 开源的历史版本，只用来说明原理，当前 iOS 版本的内部实现可能不同。

### Timer 的停止、引用与重新创建

RunLoop 强引用加入其中的 Timer。只把保存 Timer 的属性设为 nil，RunLoop 仍持有 Timer，重复 Timer 会继续触发。正确做法是先调用 invalidate()，再清空属性：

```swift
timer?.invalidate()
timer = nil
```

invalidate() 让 Timer 失效，并把 Timer 从 RunLoop 中移除。失效的 Timer 不能重新启动，要用时需要创建新的。invalidate() 应在添加 Timer 的那个线程调用；下面的界面计时器都在主线程创建和停止。

Timer 强引用回调闭包；闭包又强捕获持有 Timer 的对象时，就形成引用环。target/selector 形式的 Timer 也强引用 target。闭包用 [weak 捕获](../language/swift-basics.md#weak-与-unowned-的使用条件) 可以打破引用环，但仍要调用 invalidate()：否则对象释放后，RunLoop 还持有 Timer，Timer 继续空转触发。应在功能结束或页面不再需要计时时主动调用 invalidate()，deinit 中的调用只作为兜底。

只注册在 default 模式的主线程 Timer，列表滚动期间不会触发。可以用 `Timer(timeInterval:repeats:block:)` 创建一个尚未加入 RunLoop 的 Timer，再调用 RunLoop.main.add(timer, forMode: .common)，这样滚动期间也会触发。但这只是让 Timer 在更多模式下触发，仍不保证准时。

倒计时每次回调时，应该用“截止时间 − 当前时间”重新算剩余时间，而不是把剩余值减 1：回调每次都可能延迟，减 1 的误差会越积越多。Date 是日历时间，适合有具体日期的截止时间，但用户修改系统时间时 Date 会跟着变；只计算经过了多长时间时，应使用单调时钟，即不受系统时间修改影响、只会向前走的时间。Cell 的可见性与计时器配合见 [Cell 中的倒计时](uikit-layout-and-mvvm.md#cell-中的倒计时与可见性)。

## CADisplayLink 的原理与适用场景

`CADisplayLink` 是 Core Animation 提供的、随屏幕刷新触发回调的对象。每次屏幕刷新前，系统调用创建时指定的 `target` 的 `selector` 方法，并把这个 `CADisplayLink` 作为参数传入。`target` 是接收回调的对象，`selector` 是回调方法的 Objective-C 方法名。

创建 `CADisplayLink` 后要把它加入某个 `RunLoop`，系统才会开始回调，回调在这个 `RunLoop` 所属的线程执行。回调中要修改 UIKit 视图时，就加入主线程 `RunLoop`，并在主线程创建和停止。

```text
显示刷新时序
    → 系统安排 CADisplayLink 回调
    → 注册的 RunLoop 所在线程执行目标方法
    → 应用计算动画进度并修改视图
    → 系统为后续画面准备内容
```

`CADisplayLink` 只提供回调时机，不计算任何内容；例如做自动滚动时，应用要自己算出位置并设置 `contentOffset`。普通 `Timer` 的触发时间与屏幕刷新不同步；`CADisplayLink` 与屏幕刷新同步，但主线程繁忙、模式不匹配、系统降低帧率时，回调仍会延迟。另外，RunLoop 循环一次不等于屏幕刷新一次。

### 按经过时间计算动画进度

动画进度应按实际经过的时间计算，而不是按“回调次数 × 固定增量”计算。`CACurrentMediaTime()` 返回单调时间，不受系统时间修改的影响。开始时间和当前时间必须用同一个函数获取。

```swift
// duration 已确定为大于 0 的动画时长。
let elapsed = CACurrentMediaTime() - startTime
let progress = min(elapsed / duration, 1)
let offset = startOffset + (endOffset - startOffset) * progress
```

例如，动画时长 0.5 秒，经过 0.2 秒时进度为 40%。即使中间少了几次回调，下一次回调仍按实际经过的时间计算，动画总时长不变。这种算法让动画速度与回调次数无关，但掉帧时画面仍会跳一下。

`targetTimestamp` 是下一帧预计显示的时间，可以用来计算下一帧应该显示的状态。回调中剩余的可用时间是 `targetTimestamp` 减去当前时间，比整个刷新间隔短。

### 停止回调与强引用关系

`RunLoop` 会持有已注册的 `CADisplayLink`，`CADisplayLink` 会强持有 `target`。若控制器保存 `CADisplayLink`，同时把自己作为 `target`，就会形成强引用环：

```text
控制器 → CADisplayLink → 控制器
RunLoop → CADisplayLink
```

强引用环是多个对象的强引用首尾相连，引用计数都无法降到零。所以不能只在控制器的 `deinit` 中停止 `CADisplayLink`：控制器被 `CADisplayLink` 强引用，引用计数不会归零，`deinit` 根本不会执行。

任务完成、任务取消或页面停止使用动画时，应主动执行：

```swift
displayLink?.invalidate()
displayLink = nil
```

`invalidate()` 让 `CADisplayLink` 失效，把它从 `RunLoop` 中移除，并释放它对 `target` 的强引用。只把属性设为 `nil`，`RunLoop` 仍持有 `CADisplayLink`，回调会继续。

`isPaused = true` 只暂停回调，`CADisplayLink` 仍留在 `RunLoop` 中，仍强引用 `target`；设回 `false` 就恢复回调。调用过 `invalidate()` 的 `CADisplayLink` 不能再启动，要用就重新创建。

### 弱引用代理

弱引用代理是一个中间对象：它代替控制器作为 `target` 接收回调，再通过 `weak` 属性把回调转给控制器。`weak` 引用不阻止控制器释放，控制器释放后 `weak` 属性自动变为 `nil`。引用关系变为：

```text
控制器 → CADisplayLink → 代理 --weak→ 控制器
RunLoop → CADisplayLink
```

下面用一个页面的透明度动画演示。`@MainActor` 表示这些类的代码在主线程执行；本例把 `CADisplayLink` 加入主线程 RunLoop，所有 UIKit 访问都在主线程进行。相关并发概念见 [MainActor、主队列与异步调用](../language/swift-basics.md#mainactor主队列与异步调用)。

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

系统调用代理的 `tick(_:)`，代理再调用控制器的 `updateAnimation(_:)`。控制器释放后，`owner` 变为 `nil`，代理不再转发。`CADisplayLink` 强引用代理，所以控制器不需要再保存代理。

弱引用代理只是让 `CADisplayLink` 不再强引用控制器，任务结束时仍要调用 `invalidate()` 停止回调。上例在动画完成和 `viewDidDisappear` 时停止；如果还有取消、用户接管等情况，也要在对应位置调用 `stopAnimation()`。不同展示方式是否触发消失回调，见 [push、present 与出现消失回调](uikit-layout-and-mvvm.md#pushpresent-与出现消失回调)。

### 运行模式与工作量

只加入主线程 `.default` 模式时，列表滚动期间回调会暂停；加入 `.common` 后，滚动期间也会回调。模式的定义见 [RunLoop 与运行模式](#runloop-与运行模式)。

`.common` 只决定在哪些模式下回调，不会在用户拖动时自动停止动画。如果用户开始拖动后应该由用户控制滚动，应用要在手势或滚动回调中主动停止自动动画。

每次回调只做轻量的进度计算和必要的界面更新。60 Hz 屏幕的刷新间隔约 16.7 毫秒，120 Hz 约 8.3 毫秒，这段时间还要留给系统的布局和渲染。`CADisplayLink` 不会让大图解码、复杂布局变快，也不会把工作移到后台。

### 用途与取舍

`CADisplayLink` 适合需要应用每一帧自己计算画面的任务，例如自定义物理动画、波浪曲线、音频波形显示、序列帧动画，以及把高频输入合并后按屏幕刷新节奏更新界面。音频采集和解码有自己的节奏，不应放进显示回调。播放 GIF 时，每次回调检查当前时间是否到了下一帧，到了才换图，不必每次回调都换；FLAnimatedImage 就是这样做的。

是否使用 `CADisplayLink`，要看画面是否持续变化、是否需要应用自己计算，以及现有框架是否已有刷新循环。普通的位移、透明度和弹簧动画交给系统的 UIView 动画即可；`MTKView` 自带绘制循环，不需要再加一个 `CADisplayLink`。

回调频率只能说明回调是否及时，不能说明每次回调产生的画面都显示到了屏幕上。统计方法与观测边界见 [回调 FPS 与卡顿监测](rendering-and-performance.md#回调-fps-与卡顿监测)。
