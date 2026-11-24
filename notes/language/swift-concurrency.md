# Swift 并发：actor、Sendable 与 Task

[返回笔记目录](../README.md)

本文说明 Swift 并发中的 actor 隔离与重入、MainActor 与主队列、Sendable 与 @Sendable 的检查规则、Swift 6 的并发检查与版本条件，以及 Task、Task.detached、async let 和 TaskLocal。

## 本文目录

- [actor 的默认隔离与重入](#actor-的默认隔离与重入)
- [MainActor、主队列与异步调用](#mainactor主队列与异步调用)
  - [跨隔离调用与同步调用](#跨隔离调用与同步调用)
  - [任务挂起与线程释放](#任务挂起与线程释放)
  - [单线程并发与挂起恢复的触发者](#单线程并发与挂起恢复的触发者)
  - [同步函数发起主 actor 任务](#同步函数发起主-actor-任务)
- [Sendable 与 @Sendable](#sendable-与-sendable)
  - [结构体与类的检查条件](#结构体与类的检查条件)
  - [@Sendable 的捕获与执行要求](#sendable-的捕获与执行要求)
  - [具名函数与方法引用](#具名函数与方法引用)
- [Swift 6 的并发检查与版本条件](#swift-6-的并发检查与版本条件)
  - [Swift 6.2 的隔离与执行配置](#swift-62-的隔离与执行配置)
- [Task、Task.detached 与任务局部值](#tasktaskdetached-与任务局部值)
  - [任务实例、闭包与自动调度](#任务实例闭包与自动调度)
  - [任务结果类型与 Void、Never](#任务结果类型与-voidnever)
  - [上下文继承与取消](#上下文继承与取消)
  - [async let 创建子任务与 await 等待结果](#async-let-创建子任务与-await-等待结果)
  - [TaskLocal 与普通闭包捕获](#tasklocal-与普通闭包捕获)

## actor 的默认隔离与重入

actor 是 Swift 中保护共享可变状态的引用类型。可变状态是实例中允许更新的数据，例如一个剩余数量属性。actor 隔离规定：同一时刻，最多只有一个任务在执行同一个 actor 实例中受隔离保护的代码，因此这些状态不会被同时修改。

普通 actor 的实例存储属性、计算属性和实例方法默认受到该 actor 的隔离保护，不需要再为每个方法添加 isolated 关键字。隔离与 async 无关：async 只表示函数可以挂起，同步的实例方法同样默认受隔离。nonisolated 可以显式声明不受该 actor 隔离的成员，但这样的成员不能直接读写 actor 隔离的可变属性。

执行器（Executor）是安排并运行任务中可执行代码的组件。串行执行器保证同一时刻只执行一个受其管理的代码片段。普通 actor 关联串行执行器，但默认不会为每个 actor 创建一条专用线程；线程是实际执行代码的资源，执行器与线程不是同一个对象。

```swift
actor Counter {
    private var count = 0

    func increment() {
        count += 1
    }

    func getValue() -> Int {
        count
    }
}

let counter = Counter()
Task {
    await counter.increment()
    print(await counter.getValue())
}
```

从 `counter` 的隔离范围外调用 `increment()` 时，Swift 必须先满足 `counter` 的执行要求，再进入方法体。如果 `counter` 正在执行其他任务的代码，当前任务会挂起等待，原来的线程可以去执行其他工作；进入 actor 时不一定换线程。整个调用仍由同一个任务完成，`await` 不会创建新任务。`increment()` 返回后，任务回到调用方原有的隔离范围继续执行。

串行保护以 actor 实例为单位，覆盖这个实例的全部受隔离成员，而不是每个方法各自串行。任务 A 执行 `counter.increment()` 的同步方法体时，任务 B 不能同时执行同一实例的 `increment()` 或 `getValue()`。同步方法没有挂起点，其执行期间不会被同一 actor 上另一个任务的受隔离代码穿插。

示例只有这一个修改任务，所以打印 `1`。如果其他任务也访问 `counter`，其他调用可以插在 `increment()` 返回与 `getValue()` 开始之间：两次独立的 actor 调用之间不受保护。

await 标记可能挂起的位置。挂起表示任务暂时停止执行，等待条件满足后再继续；挂起不会创建新线程。await 处也不一定每次都真的挂起。actor 的异步方法挂起后，其他任务可以进入同一个 actor 执行受隔离代码。这种在一次方法调用尚未结束时允许其他调用推进的行为称为 actor 重入。

actor 只保证同一实例的受隔离代码不会同时执行。一个包含 await 的方法，在 await 前后是两段代码，两段之间可以插入其他调用。方法挂起后，其他任务既可以进入其他方法，也可以再次进入同一个方法；这些调用是交替推进，而不是同时执行。不同 actor 实例之间互不影响，可以同时执行。

```swift
actor Quota {
    private var remaining = 100

    // 调用约定：amount 为正整数。
    func take(_ amount: Int) async -> Bool {
        guard remaining >= amount else { return false }
        await Task.yield()
        remaining -= amount
        return true
    }
}
```

Task.yield() 为其他任务提供执行机会。上述反例允许出现这样的执行顺序：任务 A 检查 remaining 为 100 后挂起；任务 B 也检查到 100 后挂起；A 扣除 80 得到 20；B 恢复后再扣除 80 得到 -60。这个顺序不一定每次出现，但它说明：挂起前检查通过的条件，恢复后可能已经不成立。

需要作为一个整体完成的检查和状态更新，可以放在同一个没有挂起点的 actor 隔离代码段中。下面的修改先检查并扣除数量，再让出执行机会；两个请求分别取走 80 时，最多只有一个请求成功。

```swift
func take(_ amount: Int) async -> Bool {
    guard remaining >= amount else { return false }
    remaining -= amount
    await Task.yield()
    return true
}
```

这个方法替换 Quota 中原来的 take 方法。如果 await 中的异步步骤可能失败，失败后还要把扣除的数量加回去。另一种做法是在 await 之后重新检查状态，再决定是否更新。

## MainActor、主队列与异步调用

MainActor 是 Swift 内置的全局 actor。全局 actor 是由程序各处共同使用的 actor，其隔离规则可以应用于不同类型中的方法和属性。@MainActor 声明代码和数据归 MainActor 管理；actor 隔离的基本含义见[actor 的默认隔离与重入](#actor-的默认隔离与重入)。

在 iOS、macOS 应用中，MainActor 使用与主队列关联的串行执行器，安排受隔离代码在主线程执行。所有归属于 MainActor 的方法和属性共享这一隔离范围，不会为每个标有 @MainActor 的类型或实例分别创建执行器。与普通 actor 一样，MainActor 上的异步方法挂起期间，其他调用也可以进入。

三者是不同的概念：主线程是实际执行代码的线程；`DispatchQueue.main` 是 GCD 提供的串行队列，提交到主队列的任务在主线程执行；MainActor 是 Swift 的隔离规则，编译器据此检查访问是否合法，运行时把受隔离代码安排到主线程执行。

### 跨隔离调用与同步调用

@MainActor 规定函数在哪里执行；await 标记调用可能挂起的位置。从 MainActor 外部调用受 MainActor 隔离的函数时，必须写 await：Swift 要先把调用安排到 MainActor，期间调用方可能挂起。被调用函数即使没有声明 async，跨隔离调用也需要 await。

```swift
@MainActor
func updateUI() {
    print("更新界面")
}

Task.detached {
    await updateUI()
    print("更新完成")
}
```

Task.detached 创建不继承创建位置 actor 隔离的任务。上述任务等待 updateUI() 返回后，才打印“更新完成”。updateUI() 在 MainActor 上执行，但后面的 print 不会因为上一行调用而获得 MainActor 隔离。

| 调用情况 | 行为 |
| --- | --- |
| MainActor 外部通过 await 调用 updateUI() | Swift 安排函数体在 MainActor 上执行，并等待调用完成 |
| 非隔离同步函数直接调用 updateUI() | 在正常隔离检查下编译报错，不会自动把同步调用转换成异步调度 |
| 调用方已经归属于 MainActor，直接调用同步的 updateUI() | 合法，不需要 await，也不需要额外切换 |
| 调用没有 actor 隔离要求的普通同步函数 | 函数本身在调用线程执行，不会仅因普通调用而切换主线程 |

await 本身不会切到主线程。只有目标函数受 MainActor 隔离时，函数体才保证在主线程执行；隔离可以显式声明，也可以从所在类型推断。nonisolated 同步函数不属于任何 actor，在调用方所在的线程执行，可能是主线程，也可能是后台线程。调用是否合法由编译器按隔离规则检查，与函数名和当前线程无关。

旧语言模式、兼容接口或 Objective-C 调用可能绕过编译期的隔离检查。绕过检查的同步调用不会被自动切到主线程：函数可能直接在调用线程执行，也可能在运行时检查隔离时失败。

### 任务挂起与线程释放

任务是 Swift 管理的一段异步工作，记录执行进度和继续执行需要的状态。线程是实际执行代码的资源。同一个任务在执行过程中可以先后由不同线程执行；任务等待某个调用完成时，原来的线程不必停在那里等待。

```swift
Task.detached {
    print("A")
    await updateUI()
    print("B")
}
```

假设任务最初在线程 T 上执行，并需要转到 MainActor，过程可以如下：

1. 线程 T 执行 print("A")。
2. 到达 await updateUI() 时，Swift 保存继续执行需要的状态，并安排 MainActor 执行 updateUI()。
3. 等待调度期间，任务可以挂起，线程 T 可以执行其他工作；当前任务的 print("B") 仍然不能执行。
4. 主线程执行 updateUI()。函数返回后，Swift 安排当前任务继续执行 print("B")。
5. 继续执行时不保证使用原来的线程 T，也不保证仍在主线程。

A → updateUI() → B 的先后顺序不变。任务挂起只是让原线程去做其他工作，任务自己的 print("B") 仍要等 updateUI() 返回后才执行。从后台线程调用 MainActor 的方法，一定要转到主线程执行；普通 actor 没有专用线程，所以调用普通 actor 的方法不一定换线程。

“不阻塞原线程”只描述等待期间的情况。updateUI() 执行时仍然占用主线程；函数内部如果有大量同步计算或阻塞调用，界面仍会卡顿。

### 单线程并发与挂起恢复的触发者

并发表示多个任务的执行过程在时间上重叠；并行表示多个任务在同一时刻执行代码。单个线程可以交替推进多个异步任务，但同一时刻只能执行其中一个任务的代码。任务挂起时，Swift 保存恢复执行所需的状态，然后暂停任务；挂起的任务不占用线程。

在 iOS、macOS 应用中，两个归属于 MainActor 的任务可以并发推进，但它们在 MainActor 上的代码片段串行执行。下面用 Task.sleep 展示等待时间可以重叠的情况：

```swift
Task { @MainActor in
    print("A 开始")
    try await Task.sleep(nanoseconds: 1_000_000_000)
    print("A 结束")
}

Task { @MainActor in
    print("B 开始")
    try await Task.sleep(nanoseconds: 1_000_000_000)
    print("B 结束")
}
```

假设 A 先执行：A 开始 → A 等待并挂起 → 主线程执行 B → B 等待并挂起 → 主线程可执行其他工作。各任务的等待条件满足后，Swift 再安排后续代码执行。两个任务的开始和恢复顺序没有保证；示例中的“结束”打印以等待正常返回、没有抛错为前提。

挂起和恢复由异步操作的实现与 Swift 并发运行时配合完成。并发运行时是程序运行期间管理任务、保存挂起状态并安排任务继续执行的系统组件。开发者调用异步 API 指定要等待的操作；await 只标记可能挂起的位置，本身不会让任务暂停。

| 环节 | 直接执行者与动作 |
| --- | --- |
| 指定等待 | 开发者编写 await Task.sleep 等调用 |
| 挂起任务 | 被调用操作的实现通过运行时登记等待并挂起任务 |
| 允许继续 | 等待时间到达、操作完成或相应错误发生后，完成处理代码通知运行时任务可以继续 |
| 实际恢复 | 运行时通过适用的执行器安排后续代码；属于 MainActor 的代码需要获得主线程执行机会 |

任务被通知可以继续后，还要等执行器安排，才会真正执行。开发者用 continuation 包装回调接口时，在完成回调中调用 resume(returning:) 或 resume(throwing:)。continuation 是把结果交回给等待中的异步调用、让任务继续执行的对象；每个 continuation 必须恰好恢复一次。调用 resume 只是通知任务可以继续，后续代码不一定在调用 resume 的线程上执行。

操作系统为调度其他线程而暂停当前线程，与 Swift 任务挂起是两回事：线程被暂停时，任务并没有让出 MainActor，其他 MainActor 任务不能执行。所以耗时的同步循环会一直占用主线程，直到循环结束。

### 同步函数发起主 actor 任务

普通同步函数不能直接使用 await。如果函数不归属于 MainActor，直接同步调用 updateUI() 会违反隔离要求。需要从这样的函数发起界面更新时，可以创建一个指定在 MainActor 执行的新任务。

```swift
nonisolated func backgroundCallback() {
    // updateUI() // 编译错误：非隔离同步调用

    Task { @MainActor in
        updateUI()
    }

    print("回调继续执行")
}
```

Task { ... } 创建新任务；@MainActor in 指定大括号中的代码归 MainActor 管理。新任务执行时已经处于 MainActor，所以可以直接调用同步的 updateUI()。

backgroundCallback() 创建任务后继续执行，不等待新任务完成。“更新界面”和“回调继续执行”的打印顺序不确定。如果后续代码必须等界面更新结束再执行，应在同一个异步任务中写 await updateUI()。

## Sendable 与 @Sendable

`Sendable` 是一个标记协议：类型遵循 `Sendable`，表示这个类型的值可以安全地跨隔离边界传递。标记协议没有需要实现的方法。隔离边界是两段分别由不同 actor 或不同并发执行部分负责访问状态的代码之间的分界，例如把值从一个 actor 传给另一个 actor。数据竞争指多个执行部分在没有必要同步的情况下访问同一块内存，且至少一个访问是写入。

编译器在编译时检查类型及其使用是否符合并发安全规则，运行时不做监控。声明 `Sendable` 不会添加锁，类型本身必须已经满足安全条件。下面的示例以 Swift 6 语言模式、默认 actor 隔离为 `nonisolated` 为前提；受 actor 隔离的类型按隔离规则另行处理。

### 结构体与类的检查条件

```swift
struct CounterSnapshot: Sendable {
    var value: Int
}
```

上述结构体可以遵循 `Sendable`，因为 `Int` 符合 `Sendable`，结构体按值语义使用。值传给另一个任务时会复制一份，两边分别修改自己的 `value`，互不影响。但多个任务仍然不能在没有保护的情况下同时修改同一个 `CounterSnapshot` 变量。

将同样的可写属性放入普通类，结果不同：

```swift
final class CounterSnapshot: Sendable {
    var value: Int = 0 // 编译错误：普通 Sendable 类存在可写存储属性
}
```

传递类引用时只复制引用，多个任务拿到的是同一个对象，可以同时修改对象中的 `value`，因此编译器拒绝这个声明。普通的非 actor 隔离类要遵循 `Sendable`，需要满足：类是 `final`；所有存储属性都是 `let`，且类型符合 `Sendable`；除 `NSObject` 外不继承其他类。

```swift
final class CounterSnapshot: Sendable {
    let value: Int

    init(value: Int) {
        self.value = value
    }
}
```

以上三个同名类型是互相替代的示例，不能在同一作用域一起声明。`final` 禁止继承，`let` 禁止重新赋值。如果 `let` 保存的是另一个类实例，这个实例的类型也必须符合 `Sendable`，因为 `let` 只保证引用不变，不保证被引用对象的内容不变。

actor 自动符合 `Sendable`，其可变状态由 actor 隔离保护。受 `@MainActor` 等全局 actor 隔离的类型也可以通过隔离规则保护可变状态。`@unchecked Sendable` 让编译器跳过对存储属性的安全检查，由开发者自己保证安全，例如在类内部用锁保护所有可变属性；这个声明本身不会添加任何同步。

### @Sendable 的捕获与执行要求

`@Sendable` 是函数类型属性，表示函数值及其捕获环境可以安全地跨并发隔离边界传递。捕获环境是函数为之后执行而保存的外部局部变量或值。`@Sendable` 只是一项编译期要求，不会创建任务、切换线程或添加锁；函数体访问 actor 或共享状态时，仍要遵守隔离规则。

对于没有 actor 隔离保护的 `@Sendable` 闭包，编译器要求捕获的值都符合 `Sendable`，并且禁止捕获外部的 `var` 变量。原因是捕获 `var` 时，闭包与外部共享同一个变量存储，即使变量类型是 `Int` 这类 `Sendable` 类型，两边同时读写也会产生数据竞争。

```swift
func captureExample() {
    var count = 0

    // 不能把下面的闭包作为普通非隔离的 @Sendable 函数值：
    // let change: @Sendable () -> Void = { count += 1 }

    let read: @Sendable () -> Int = { [count] in
        count
    }

    count = 10
    print(read()) // 0
}
```

`[count]` 在创建闭包时复制当时的整数值，闭包不再与外部共享 `count` 变量，因此外部之后把 `count` 改成 10，`read()` 仍返回 0。捕获类实例时，`[object]` 只复制对象引用，不会复制对象，对象的类型仍需符合 `Sendable`。捕获列表的基础机制见[默认捕获与捕获列表](swift-basics.md#默认捕获与捕获列表)。

可以捕获符合 `Sendable` 的 actor 引用，但访问 actor 的受隔离成员仍要满足隔离规则。例如，下列代码使用[前文的 Counter](#actor-的默认隔离与重入)：

```swift
func makeAction(counter: Counter) -> @Sendable () async -> Void {
    return {
        await counter.increment()
    }
}
```

`@Sendable` 约束的是函数值及其捕获内容，不要求函数的参数、返回值和内部局部值都符合 `Sendable`。参数或返回值真正跨越隔离边界时，编译器会单独检查这次传递是否安全；某些情况下，非 `Sendable` 值也可以被安全转移，规则见 [Swift 6 的并发检查与版本条件](#swift-6-的并发检查与版本条件)。

### 具名函数与方法引用

`@Sendable` 同样适用于具名函数。嵌套函数可以捕获外层局部值，因此也需要接受捕获检查。

```swift
func makeReader(value: Int) -> @Sendable () -> Int {
    @Sendable
    func read() -> Int {
        value
    }
    return read
}

let reader = makeReader(value: 10)
print(reader()) // 10
```

`read` 捕获的 `value` 是不可变的 `Int`，符合传递要求。`return read` 返回函数值，不执行函数体；函数类型与调用写法见[函数签名与回调类型](swift-basics.md#函数签名与回调类型)。闭包表达式写法 `return { value }` 也可以由返回类型推断为 `@Sendable`。

Swift 6 可以推断全局函数、静态方法等函数引用的可发送性。普通非隔离的实例方法引用，例如 `object.read`，会绑定接收者 `object`；其能否作为 `@Sendable` 函数值使用，还取决于接收者是否符合 `Sendable`。接收者不符合 `Sendable` 时，显式写 `@Sendable` 也无法通过检查；方法原有的 actor 隔离要求也依然有效。

全局函数不捕获局部变量，但函数体读写全局变量时，编译器仍会检查这些访问是否受隔离或同步保护。

## Swift 6 的并发检查与版本条件

Swift 6 延续 Swift 5.5 引入的 `async`、`await`、Task 和 actor 模型。主要变化是更完整地检查数据竞争风险，并逐步调整并发代码的隔离和执行规则。编译器版本、语言模式和单独启用的特性是三项独立设置。例如，用 Swift 6 编译器编译的模块，仍然可以使用 Swift 5 语言模式。

| 版本或配置 | 变化与条件 |
| --- | --- |
| Swift 5.10，完整并发检查 | 可通过 `-strict-concurrency=complete` 提前检查隔离、捕获和共享状态；Swift 5 语言模式下相关迁移诊断通常是警告 |
| Swift 6 语言模式 | 完整并发安全检查成为语言规则，相关违规通常按错误处理；检查范围不只包含 `async` 函数，还包括共享全局和静态状态等 |
| Swift 6.0，区域隔离分析 | 编译器跟踪非 `Sendable` 值及其关联引用；能够证明传递后不会被两侧并发访问时，可以允许跨隔离传递 |
| Swift 6.0，`sending` | 参数或返回值可以声明安全转移要求；调用方或返回方必须满足相应的后续访问限制 |
| Swift 6.0，函数可发送性推断 | 更多具名函数和方法引用可以根据函数及接收者条件推断 `@Sendable` |
| Swift 6.0，动态 actor 隔离检查 | 部分旧代码或 Objective-C 互操作等静态检查不充分的调用路径，会在运行时验证 actor 隔离；检查失败可能终止执行，不会自动改成异步切换 |

区域隔离分析中的“区域”是编译器把可能互相关联的数据归为一组的分析单位，与物理内存无关。别名是指向同一份数据的另一个变量或引用。一个非 `Sendable` 对象转移到另一个隔离范围后，如果原范围还能通过原变量或其他别名访问这个对象，两边就可能同时访问它，编译器因此拒绝这次转移。

`Sendable` 描述类型的性质：这个类型的所有值都可以安全传递。`sending` 描述某一次参数或返回值传递的约束：传出之后，传出方不再访问这个值。`sending` 不会让类型变成 `Sendable`，也不会复制对象；编译器只需确认传递后两侧不会同时访问同一份数据。

### Swift 6.2 的隔离与执行配置

Swift 6.2 的以下行为需要在项目设置中启用或在代码中显式声明，升级编译器后不会自动生效。

| 设置或声明 | 行为 |
| --- | --- |
| 默认 actor 隔离设为 `MainActor` | 没有显式指定其他隔离方式的声明，默认归属于 MainActor；显式标为 `nonisolated` 或其他 actor 的声明不受影响 |
| 启用 `NonisolatedNonsendingByDefault` | 没有显式选择执行方式的 `nonisolated async` 函数，在调用方所在的 actor 上执行；调用方不属于任何 actor 时，函数也不属于任何 actor |
| 在异步函数上声明 `@concurrent` | 函数离开调用方的 actor，在全局并发执行器上执行；不会创建新任务，也不会分配专用线程 |

`nonisolated` 表示声明不属于任何 actor。旧规则下，`nonisolated async` 函数在全局并发执行器上执行；启用上述新规则后，这类函数改为在调用方的 actor 上执行。因此，判断一段异步代码在哪里运行，要同时看隔离声明、编译器版本、语言模式和启用的特性。

迁移可以按模块推进。`@preconcurrency` 等兼容手段只放宽编译检查或调整互操作行为，不会给代码添加同步，原本不安全的访问依然不安全。

## Task、Task.detached 与任务局部值

### 任务实例、闭包与自动调度

Task 表示 Swift 管理的一段异步工作。Task 是结构体类型；创建出的值称为 Task 实例（结构体的值也称为实例）。Task 实例代表一个已经创建的任务，可以用来等待结果、查询状态和发出取消请求。类型与实例的基本区别见[类型、实例与对象引用](swift-memory-and-references.md#类型实例与对象引用)。

```swift
let task = Task {
    print("Hello")
}
```

Task { ... } 调用 Task 的初始化器，创建任务并返回 Task 实例，task 常量保存这个实例。初始化器是创建类型实例的入口。大括号中的闭包是任务要执行的代码。闭包与任务不同：闭包只描述要执行的内容；任务负责执行这段内容，并记录执行进度、结果和取消状态。

这里使用尾随闭包语法，即把最后一个闭包参数写在圆括号外。改写成普通参数形式如下，两种写法效果相同：

```swift
let task = Task(operation: {
    print("Hello")
})
```

任务创建后由 Swift 自动安排执行，不需要调用 start，不需要提交到 DispatchQueue，也不需要 await 才能启动。任务在哪个执行器上运行由隔离规则决定；Swift 不会为每个 Task 创建一条新线程。

创建任务的调用立即返回，不等待任务完成。不保存返回的 Task 实例，任务也照常运行；只有之后要等待结果或取消任务时，才需要保存这个实例。

### 任务结果类型与 Void、Never

`Task<Success, Failure>` 中，`Success` 表示任务成功完成时返回的值类型，`Failure` 表示任务失败时抛出的错误类型。`Failure` 必须符合 `Error` 协议。

`Void` 是空元组 `()` 的类型，表示没有需要交给调用方的结果数据，具体定义见[元组、字典与空元组 Void](swift-basics.md#元组字典与空元组-void)。`Never` 是没有任何值的类型；用于 `Failure` 时，表示任务闭包不会向外抛出错误。`Task<Void, Never>` 因此表示“不返回有意义的数据，也不向外抛出错误”的任务。

```swift
let task: Task<Void, Never> = Task {
    print("完成")
}
```

闭包执行 `print` 后正常结束，结果是 `()`。这里的 `Never` 只表示任务不会抛错；任务照样会结束，也照样可以被取消。

下面的方法接收一个已经创建的任务，并等待它的结果。任务在创建时已经开始运行，等待不会启动任务。

```swift
func waitForCompletion(_ task: Task<Void, Never>) async {
    await task.value
}
```

对于 `Task<Void, Error>`，等待结果需要写 `try await task.value`，因为闭包可以向外抛出错误。例如，`try await Task.sleep(...)` 在任务被取消时抛出错误。如果闭包用 `do-catch` 处理了全部错误、不再向外抛出，这个闭包就可以用于 `Task<Void, Never>`。取消请求如何生效见[上下文继承与取消](#上下文继承与取消)。

### 上下文继承与取消

这里的上下文指 actor 隔离、任务优先级和任务局部值。actor 隔离的机制见[actor 的默认隔离与重入](#actor-的默认隔离与重入)。

| 对比项 | Task { ... } | Task.detached { ... } |
| --- | --- | --- |
| actor 隔离 | 继承闭包定义位置的 actor 隔离；没有隔离时，不会凭空获得 | 不自动继承创建位置的 actor 隔离 |
| 优先级 | 未显式指定且存在当前任务时，继承当前任务优先级；否则由运行时确定 | 不继承当前任务优先级，可以显式指定 |
| 任务局部值 | 继承创建时适用的任务局部值绑定 | 不自动继承任务局部值绑定 |
| 创建函数返回 | 可以继续运行 | 可以继续运行 |
| 创建者取消 | 不会因此自动取消 | 不会因此自动取消 |

两者都是非结构化任务：任务的存活时间不受创建它的函数作用域约束，任务与创建者之间也没有父子关系。`Task { ... }` 虽然继承上下文，但不是子任务。只有 async let 和 withTaskGroup 等结构化方式创建的子任务，才受作用域管理，并接收父任务的取消。

调用 task.cancel() 只是发出取消请求，不会强行中断正在执行的代码。任务要自己检查 Task.isCancelled、调用 Task.checkCancellation()，或调用能响应取消的异步 API，才会真正停下。一个非结构化任务被另一个任务等待时，两者之间也不会因此建立取消关系。

```swift
@MainActor
final class Counter {
    var count = 0

    func start() {
        Task {
            self.count += 1
        }

        Task.detached {
            await self.increment()
        }
    }

    func increment() {
        count += 1
    }
}
```

Task 闭包继承 MainActor 隔离，可以直接修改 count。Detached 闭包没有继承这个隔离，不能直接修改 count。increment() 属于 @MainActor 类型，受主 actor 隔离，所以 detached 闭包要用 await 调用它，由主 actor 执行方法并修改 count。在 detached 闭包里直接写 `self.count += 1` 仍然无法编译，await 只能用于调用，不能让属性赋值跨隔离执行。

start() 中没有挂起点。start() 创建两个任务后继续执行到结束，期间主 actor 上的新任务不会插入执行。两个任务谁先增加 count 不确定，与创建顺序无关。

Task { ... } 不会把工作移到后台：闭包继承 MainActor 时，其中的大量同步计算仍会占用主线程。Task.detached 也不会创建专用线程，其中的代码仍可通过 await 进入某个 actor 执行。是否继承隔离，看的是闭包写在哪里，与创建任务时代码碰巧运行在哪条线程无关。跨隔离调用、挂起和线程释放的完整流程见[MainActor、主队列与异步调用](#mainactor主队列与异步调用)。

### async let 创建子任务与 await 等待结果

`async` 声明函数可以在执行过程中挂起。挂起时，任务保存继续执行所需的状态并暂停，线程可以去做其他工作。调用 `async` 函数不会创建新任务，被调用函数在当前任务中执行。

`async let` 声明会创建一个受当前作用域管理的子任务，由子任务计算等号右侧的表达式。子任务在声明处就被安排执行，不需要等到后面的 `await`。但子任务开始执行的时间不确定，两个子任务的开始和完成顺序也不确定。

下面的示例适用于 iOS 15 及以上。调用方提供两个有效的请求地址；两个请求各自返回数据和响应对象。

```swift
import Foundation

func loadBoth(firstURL: URL, secondURL: URL) {
    Task { @MainActor in
        do {
            async let first = URLSession.shared.data(from: firstURL)
            async let second = URLSession.shared.data(from: secondURL)

            let (firstResult, secondResult) = try await (first, second)
            print(firstResult.0.count, secondResult.0.count)
        } catch {
            print(error)
        }
    }
}
```

执行顺序是：创建父任务 → 两条 `async let` 声明分别创建子任务 → 父任务在 `await` 处等待两个结果 → 两个结果都成功返回后打印。两个请求并发推进，但不保证各自有一条专用线程，也不保证同一时刻都在执行。

`await` 标记读取结果时可能挂起的位置。结果还没就绪时，父任务挂起等待，子任务继续执行。父任务挂起不占用主线程，所以主线程可以处理其他工作。如果两个结果已经就绪，读取时不一定挂起。

`try` 表示读取可能抛出错误。两个请求都成功，才会完成元组赋值并执行 `print`；任一请求抛错，就进入 `catch`，不必等另一个请求结束。

结构化子任务受声明所在作用域管理。作用域退出前，Swift 会等待所有子任务结束；没有被 await 读取的子任务会先收到取消请求。子任务需要配合检查取消，才会提前停止。`@MainActor` 让父任务的代码在主 actor 上执行；网络等待期间父任务挂起，不占用主线程。但闭包里的同步计算仍在主线程执行，Swift 不会自动把它移到后台。相关执行规则见[MainActor、主队列与异步调用](#mainactor主队列与异步调用)。

### TaskLocal 与普通闭包捕获

任务局部值是用 @TaskLocal 声明的值，可以在一段执行范围内绑定为指定的值。绑定只影响这段范围内（以及其中创建并继承绑定的任务）读到的结果，不会修改所有任务共享的全局变量。范围内的任何函数都可以直接读取绑定的值，不必把值逐层作为参数传递。

普通局部变量与任务局部值不同，需要通过闭包捕获才能在任务中使用。Task 和 Task.detached 都可以捕获普通值，但捕获的值必须满足并发安全要求；捕获共享的可变变量时，Swift 不会自动添加同步保护。闭包捕获的具体规则见[默认捕获与捕获列表](swift-basics.md#默认捕获与捕获列表)。

```swift
enum Context {
    @TaskLocal static var logID = "默认"
}

Context.$logID.withValue("A") {
    let number = 10

    Task {
        print(number)        // 10：捕获普通局部常量
        print(Context.logID) // A：继承任务局部值绑定
    }

    Task.detached {
        print(number)        // 10：也可以捕获普通局部常量
        print(Context.logID) // 默认：没有继承绑定 A
    }
}
```

withValue 在闭包执行期间把 logID 绑定为 A。闭包中创建的 Task 继承这个绑定，即使任务在 withValue 返回后才执行，读到的仍是 A。Detached 任务不继承绑定，所以读到默认值。两个任务的执行顺序不确定。

Context 是枚举类型名。Context.logID 读取当前绑定的 String 值；Context.$logID 是 @TaskLocal 自动生成的配套成员，类型是 TaskLocal<String>，提供 withValue 方法。$logID 是成员名称，与闭包参数简写 $0 无关。

Context.$logID.withValue("A") { ... } 先访问静态成员 $logID，再调用 withValue。字符串 A 是第一个参数，尾随闭包是 operation 参数。使用普通参数形式可写成下面这样。

```swift
Context.$logID.withValue("A", operation: {
    print(Context.logID)
})
```

任务局部值通过 withValue 绑定，不能像普通可写静态属性那样直接赋值。嵌套绑定在自己的范围内生效，返回外层后继续使用外层绑定；没有可用绑定时读取声明时的默认值。
