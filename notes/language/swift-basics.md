# Swift 语言基础

[返回笔记目录](../README.md)

## 本文目录

- [基础语义与内存管理](#基础语义与内存管理)
- [类型安全、值语义与动态能力](#类型安全值语义与动态能力)
- [类型、实例与对象引用](#类型实例与对象引用)
- [符号引用与对象引用](#符号引用与对象引用)
- [声明、定义、实现与引用](#声明定义实现与引用)
- [Swift 与 C 接口互操作](#swift-与-c-接口互操作)
- [存储属性、计算属性与访问器](#存储属性计算属性与访问器)
- [扩展中的类型属性与实例存储限制](#扩展中的类型属性与实例存储限制)
- [协议存在类型与泛型约束](#协议存在类型与泛型约束)
- [逃逸闭包与执行时机](#逃逸闭包与执行时机)
- [闭包参数、捕获列表与可选回调](#闭包参数捕获列表与可选回调)
- [多行字符串与脚本执行](#多行字符串与脚本执行)
- [map、KeyPath、Sequence、Collection 与 Set](#mapkeypathsequencecollection-与-set)
- [元组、字典与空元组 Void](#元组字典与空元组-void)
- [相等规则与对象身份](#相等规则与对象身份)
- [defer 与作用域退出](#defer-与作用域退出)

基础语义与内存管理的小节：[闭包与 Delegate](#闭包与-delegate)。

内存管理补充：[ARC 与 Android GC：引用计数、可达性与引用环](#arc-与-android-gc引用计数可达性与引用环)。

内存管理补充：[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)。

基础语义补充：[普通函数参数与局部变量](#普通函数参数与局部变量)。

属性补充：[实例存储属性与局部变量的初始化](#实例存储属性与局部变量的初始化)；[为什么扩展不能添加实例存储属性](#为什么扩展不能添加实例存储属性)。

补充专题：[inout 与内存独占访问](#inout-与内存独占访问)；[actor 的默认隔离与重入](#actor-的默认隔离与重入)；[数组值语义、写时复制与返回值](#数组值语义写时复制与返回值)；[ArraySlice 的存储保留与索引](#arrayslice-的存储保留与索引)；[类的两阶段初始化与 self 使用时机](#类的两阶段初始化与-self-使用时机)。

## 基础语义与内存管理

**ARC 与引用关系。** ARC 根据强引用关系管理对象生命周期。编译器插入相应引用计数操作；强引用计数归零后对象可被释放，weak 不维持对象存活。分析闭包时需要检查真实的强引用环；捕获 self 也可能只是延长生命周期。ARC 不等于垃圾回收机制 GC。

**协议与继承。** 协议定义能力要求，class、struct、enum 可遵循多个协议。类采用单继承，并通过继承复用实现。协议要求与 extension 中的默认实现需要分别理解。

**可选绑定。**`if let` 解包后的值用于成功分支。`guard let` 失败分支必须退出当前作用域，成功后可在后续代码使用解包值。退出方式取决于作用域，不是所有情况都只能使用 return。

**实例 lazy。** 普通实例的 lazy 存储属性在首次读取时执行初始化并保存结果，后续读取已有值。它适合初始化有成本且未必使用的属性，不能直接套用 `static let` 的单次初始化保证。

**错误处理。**`throws` 表示函数可能抛错；`try` 本身不捕获错误，应使用 `do-catch` 捕获或向外传播。`try?` 在抛错时得到 nil 并丢弃具体错误；`try!` 遇到抛错会触发运行时错误，不等于强制解包返回的 Optional。

### 普通函数参数与局部变量

Swift 的普通函数参数默认是常量，不能在函数内部重新赋值。常量表示初始化后不能被赋予另一个值的名称。因此，即使参数声明没有写 let，直接给参数赋值仍会产生编译错误。

如果只需要在函数内部改变用于计算的值，可以用参数初始化一个 var 局部变量。局部变量是在函数内部声明、只在对应代码范围内使用的变量。

```swift
func increment(_ value: Int) -> Int {
    // value += 1  // 编译错误：不能给普通参数重新赋值
    var value = value
    value += 1
    return value
}

let number = 10
let result = increment(number) // result 为 11，number 仍为 10
```

在 var value = value 中，等号右侧的 value 是函数参数，左侧的 value 是新声明的可变局部变量。声明之后，同一代码范围内的 value 使用这个局部变量。示例修改局部变量后返回 11；调用方的 number 仍为 10。

如果参数的类型是 class，不能给参数重新赋值，只限制参数改为指向另一个对象；只要对象属性可写，函数仍可以修改该对象的属性。创建一个保存同一对象引用的局部变量，也不会自动复制对象。

这里的普通参数不包括 inout 参数。inout 参数允许函数修改传入变量的值，并在函数返回时将修改写回调用方。

### inout 与内存独占访问

inout 参数允许函数修改调用方传入的变量，调用时在变量前加 &。对普通变量的一次修改访问尚未结束时，其他独立访问不能读取或修改同一块内存；这个要求称为内存独占访问。通过同一个 inout 参数完成内部的读取和修改，不属于另一个独立访问。

函数对 inout 参数建立的修改访问持续整个函数调用，不只持续某条赋值语句执行的瞬间。同一个变量同时传给一个函数的两个 inout 参数，会产生指向同一存储位置、持续时间重叠的修改访问。

```swift
func update(_ first: inout Int, _ second: inout Int) {
    first += 1
    second += 2
}

func example() {
    var value = 10
    // update(&value, &value) // 编译错误：访问冲突
}
```

两条加法语句按顺序执行，不会消除参数访问持续时间的重叠。即使函数体完全不使用 second，只要 second 仍声明为 inout，原来的重复传参方式仍然不合法。这类访问冲突可以发生在单线程中；Swift 会根据具体代码在编译时或运行时检查，本例可在编译时发现。

如果目标是连续修改同一个变量，使用一个 inout 参数表达这次修改即可。

```swift
func update(_ value: inout Int) {
    value += 1
    value += 2
}

var value = 10
update(&value)
print(value) // 13
```

如果 second 只接收调用前的整数值，可以将 second 声明为普通 Int 参数，再调用 update(&value, value)。普通参数先完成求值，随后才开始 inout 参数的修改访问，因此这里的读取与修改访问不会重叠。

### ARC 与 Android GC：引用计数、可达性与引用环

Swift 类实例通常由 ARC（自动引用计数）管理。ARC 跟踪对象的强引用计数；当强引用计数归零时，对象可以释放。Android 上由运行时管理的 Kotlin／Java 对象主要使用追踪式垃圾回收（GC）。GC 从一组起点沿对象引用查找仍然可达的对象，并回收已经不可达的对象。这里的比较不包括 Android 原生代码中自行管理的内存。

这里的“原生代码”主要指编译为机器指令执行的 C/C++ 代码。一个 Android 应用可以同时包含 Kotlin／Java 代码和 C/C++ 代码。GC 管理由运行时分配和追踪的 Kotlin／Java 对象，并不因此接管应用申请的所有内存。例如，通过 `malloc` 申请的内存，需要由代码或负责管理该内存的库调用 `free` 释放；不能仅依靠 Kotlin／Java 对象的 GC 自动释放这块内存。

Swift 也有管理范围的区别：普通类实例由 ARC 管理，但通过 `UnsafeMutablePointer.allocate` 手动申请的内存，需要按对应 API 的规则调用 `deallocate()` 释放。管理类实例与管理手动申请的内存是两件事。

强引用是会保持目标对象存活的对象引用。可达性表示能否从垃圾回收起点沿引用链到达某个对象。以下示意中的箭头全部表示强引用。

GC Roots 是垃圾回收器追踪对象时使用的一组起点，例如活动线程栈中仍然有效的对象引用。GC 从这些起点逐层访问其他对象。对象暂时没有执行方法，不代表对象已经不可达。

`GC Root → A → B`

在上面的关系中，GC 能从起点找到 A，再从 A 找到 B，因此 A 和 B 都不能作为不可达对象回收。

引用环是沿对象之间的引用可以回到起点的关系，例如 A 强引用 B，同时 B 强引用 A。假设 A 和 B 都是普通对象，最初只有一条外部强引用连接这个引用环：

`最初：外部强引用 → A ⇄ B`

`断开唯一外部强引用后：A ⇄ B`

| 机制 | 断开唯一外部强引用后的结果 |
| --- | --- |
| Swift ARC | A 和 B 各自仍被对方强引用，强引用计数没有归零，因此不能自动释放。 |
| Android GC | 如果从所有 GC Roots 出发都无法到达 A 和 B，整个引用环可以在后续垃圾回收中回收。回收不保证立即发生。 |

Swift 中可以按对象所有权关系把合适的一条引用改为 weak，或者在业务结束时明确断开强引用。weak 不保持目标对象存活；不能为了消除引用环，把所有需要持有的引用都改成 weak。

内存泄漏是对象已经不再需要，却仍然无法释放而持续占用内存。GC 能回收不可达的引用环，但不能回收仍通过强引用链可达的普通对象。例如：

`GC Root → 长期存活的单例 → 已关闭的 Activity`

只要这条强引用链仍然存在，GC 就不能回收旧 Activity。页面已经关闭是生命周期和业务状态；对象是否可以回收取决于内存管理机制及引用关系。因此，两端都需要检查谁持有页面、闭包或订阅，以及何时解除这些关系。

### 结构体、枚举与值的生命周期

结构体和枚举的普通值也由 Swift 自动管理，不需要为值本身手动释放内存。与类实例不同，普通结构体和枚举值不依靠自身的强引用计数决定生命周期。编译器根据值的使用关系和存储方式安排必要的清理。以下示例均使用普通、可复制的结构体和枚举。

需要区分值本身的存储与值内部持有的资源。只包含整数等简单成员的值，销毁时通常没有需要释放的对象引用；相关存储可以被复用，或随所在存储区域一起释放。结构体和枚举不一定存放在栈上，实际存储位置取决于使用方式和编译器实现。`Array` 虽然是结构体，其内部共享存储仍会涉及引用计数。

```swift
class Image {}

struct Photo {
    let image: Image
}

enum State {
    case loaded(Image)
}
```

`Photo.image` 和 `State.loaded` 的关联值都会持有 `Image` 实例的强引用。销毁某一份 `Photo` 或 `State.loaded` 值时，Swift 会释放这份值持有的强引用。只有当 `Image` 实例不再有其他强引用时，ARC 才会销毁该实例。复制 `Photo` 值不会自动复制 `Image` 实例；两份 `Photo` 值可以引用同一个 `Image` 实例。

**作用域与生命周期。** 作用域是源代码中可以使用某个变量名的范围。生命周期是程序运行时，某一份值从创建到销毁的时间段。变量名离开作用域，不代表返回到外部或保存在其他位置的值也随之销毁。

```swift
func example() {
    let photo = Photo(image: Image())
    print(photo.image)  // 最后一次使用这份局部值
    // 后续代码不再使用 photo
}                      // photo 的作用域结束
```

**局部值的清理。** 如果 `Photo` 只在当前作用域中使用，没有被返回或被外部闭包捕获，退出作用域是常规清理点。在生命周期规则允许的范围内，编译器也可能在最后一次必要使用之后提前清理。不能保证清理恰好发生在 `}`，也不能保证最后一次使用结束后立即清理。编译器在允许的位置生成清理操作；程序执行这些操作时，结束这份值的生命周期并释放成员持有的强引用。

```swift
func replacePhoto() {
    var photo: Photo? = Photo(image: Image())
    print(photo?.image as Any)
    photo = nil
    // 当前作用域还没有结束
}
```

**旧值被替换。** 从赋值语义看，`photo = nil` 移除了变量原来保存的 `Photo` 值，变量不再通过这份值持有 `image` 的强引用。清理不必等到作用域结束。但相关 `Image` 实例是否随之销毁，还取决于是否存在其他强引用；不能把这条赋值语句当作所有构建配置下固定的 `deinit` 执行位置。

```swift
func makePhoto() -> Photo {
    let photo = Photo(image: Image())
    return photo
}

let result = makePhoto()
```

**值被返回或保存在外部。** `makePhoto()` 返回后，局部变量名 `photo` 已不可用，但返回的 `Photo` 值仍由 `result` 保存。`result` 中的 `image` 强引用仍可保持 `Image` 实例存活。如果值保存在对象的存储属性中，属性被替换或所属对象被销毁时会清理对应的旧值。如果一个离开当前作用域后仍然存活的闭包保留了值或捕获的变量存储，相关资源也可以继续存活。创建值的函数结束，不会自动清除这些外部保存关系。

### 闭包与 Delegate

#### 概念与职责

**闭包与 Delegate。** 闭包携带可调用的代码，Delegate 通过协议定义接收事件的对象与接口。应根据事件组织方式和对象关系选型，不能仅按事件是否高频决定。

**概念区别。** 闭包是可以作为值传递的代码块，闭包可以捕获定义位置的变量。在回调场景中，使用组件的一方提供具体行为，组件在约定的时机调用闭包。Delegate 是一种委托模式：一个对象把约定的职责交给另一个对象。Swift 通常通过协议定义这些职责，由遵循协议的对象实现对应方法。

#### 选型依据

**选型原则。** 只需要传入一段具体的回调行为时，优先选择闭包；需要由一个对象统一承担一组相关职责时，优先选择 Delegate。事件频率本身不能决定选型。

| 判断维度 | 更适合闭包 | 更适合 Delegate |
| --- | --- | --- |
| 回调内容 | 一个或少量独立的结果、动作 | 一组共同描述某种职责的方法 |
| 代码组织 | 希望在发起操作的位置直接写回调 | 希望由接收对象集中实现相关方法 |
| 对象关系 | 只需要提供具体行为，无须指定接收对象的角色 | 需要明确某个对象担任事件接收者或决策者 |
| 状态管理 | 回调只使用少量局部变量 | 多个方法需要共享接收对象维护的状态 |

**选择依据。** 判断使用组件的一方需要提供的是“一段回调行为”，还是“一个承担明确职责的对象”。这个区别通常比事件数量和事件频率更有决定作用。

#### 使用场景

**闭包示例：图片导出。** 使用组件的一方发起一次导出，并传入处理导出结果的闭包。导出完成后，导出组件把成功结果或错误传给闭包。使用组件的一方在发起导出的位置直接编写后续行为，代码比较集中。

**Delegate 示例：编辑器交互。** 编辑器需要通知“开始编辑”“内容变化”“结束编辑”，还可能询问“是否允许结束编辑”。这些方法共同构成编辑器委托对象的职责。由同一个对象集中实现这些方法，职责更清楚。

#### 适用边界

**选型边界。** 闭包可以反复调用，持续触发的进度回调仍然可以使用闭包。Delegate 方法可以只调用一次，一次性事件也可以属于某个委托对象的职责。两种方式都可以通知事件，也可以返回决策结果。方法是否返回值不能单独决定选型。

#### 生命周期与持有关系

**持有关系。** 组件保存闭包时，闭包可能强引用捕获的对象，因此需要检查循环引用。Delegate 属性通常声明为 weak，但 weak 不会保持委托对象存活，必须有其他对象负责持有委托对象。

### 协议继承与类继承的区别

协议可以继承一个或多个协议。子协议在已有协议的要求之上增加要求或约束；遵循子协议的类型，也必须满足继承而来的要求和约束。

```swift
protocol Movable {
    func move()
}

protocol Stoppable: Movable {
    func stop()
}

struct Cursor: Stoppable {
    func move() {}
    func stop() {}
}
```

Stoppable 继承 Movable，因此 Cursor 需要满足 move() 和 stop() 两个要求。示例没有提供默认实现，所以 Cursor 自己实现这两个方法。

协议继承本身不会提供父类那样的实例存储和方法实现。协议扩展可以提供默认实现，但那属于协议扩展的能力，不能把“继承了协议”直接理解为“获得了实现”。类继承则可以继承父类的属性和方法实现。

协议要求还需要区分必需和可选。普通 Swift 协议中的方法要求默认是必需的；适用 Objective-C 互操作的协议可以声明 optional 要求。UIKit 的 UICollectionViewDelegate、UIScrollViewDelegate 等协议包含可选回调，遵循协议不表示必须实现每一个回调。

## 类型安全、值语义与动态能力

- **类型安全：** 值和操作受到类型规则约束，例如不能把 String 直接赋给 Int 变量；Optional 要先处理有值／无值的情况，才能作为非可选值使用。它减少一类错误，不表示所有错误都会在编译期被发现。
- **值语义：** struct、enum 以及 Array、String 等常见值类型，在赋值和传参时表现为值的复制，修改一份普通值不应改变另一份。底层可用写时复制等方式共享存储，不要求每次赋值立即深拷贝；值中含 class 成员时，两份值仍可能引用同一对象。
- **编译期约束：** 编译器提前检查参数、返回值、协议遵循、泛型约束等。例如 T: Speaker 使函数内部能够依赖 speak() 的要求。
- **动态能力：** 部分决定留到运行时，例如类多态调用、协议要求调用、动态类型转换，以及 Apple 平台上的 Objective-C 消息派发。Swift 同时支持 class 引用语义，不能概括为“Swift 全是值类型、OC 才有对象”。

### 可选类型、状态表达与并发支持

Optional 用类型表达“有值或无值”，调用方需要通过可选绑定、可选链等方式处理缺失情况。强制解包 nil 仍然会导致运行时错误，因此类型检查不能代替正确使用可选类型。Objective-C 也有静态类型检查和空值标注；Swift 的可选类型把缺失值处理更直接地纳入语言类型规则。

枚举关联值是某个枚举分支携带的数据。例如，同一个加载状态枚举可以声明 idle、loaded([String]) 和 failed(String)，使“成功时有结果、失败时有错误说明”直接体现在状态结构中。switch 的完整性检查有助于覆盖需要处理的状态。相比多个独立布尔值和可选属性，这种表达方式可以减少互相矛盾的状态组合。

类型推断根据上下文确定静态类型，可以减少重复的类型书写；类型推断不会把变量变成任意类型。结构体、枚举、泛型与协议可以共同表达数据和能力约束，具体选择仍取决于是否需要共享对象身份、继承或值语义。

async/await 用语言语法表达可挂起的异步调用流程，减少多层回调对执行关系的遮蔽。await 表示可能挂起的位置，不保证新建线程，也不保证工作在后台执行。actor 通过隔离其可变状态，限制来自外部的并发访问；跨越 await 后，其他任务可能已经改变 actor 状态。使用这些语言能力有助于表达异步关系和隔离规则，但不自动保证程序更快或没有逻辑错误。

### actor 的默认隔离与重入

actor 是 Swift 中保护共享可变状态的引用类型。可变状态是实例中允许更新的数据，例如一个剩余数量属性。actor 隔离要求对这些状态的访问遵循同一个 actor 实例的执行规则，使同一时刻最多只有一个任务执行该实例受隔离保护的代码。

普通 actor 的实例存储属性、计算属性和实例方法默认受到该 actor 的隔离保护，不需要再为每个方法添加 isolated 关键字。async 表示函数可以执行异步操作，不负责声明 actor 隔离；同步实例方法同样可以默认受隔离。nonisolated 可以显式声明不受该 actor 隔离的成员，但这样的成员不能直接读写 actor 隔离的可变属性。

await 标记可能挂起的位置。挂起表示任务暂时停止执行，等待条件满足后再继续，不等于创建新线程，也不保证每次 await 都实际暂停。actor 的异步方法挂起后，其他任务可以进入同一个 actor 执行受隔离代码。这种在一次方法调用尚未结束时允许其他调用推进的行为称为 actor 重入。

actor 保证受隔离代码不会在同一实例上同时执行，但不保证一个跨越 await 的完整方法从头到尾不被其他调用穿插。不同 actor 实例之间也不存在共同的串行执行保证。

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

Task.yield() 为其他任务提供执行机会。上述反例允许出现这样的执行顺序：任务 A 检查 remaining 为 100 后挂起；任务 B 也检查到 100 后挂起；A 扣除 80 得到 20；B 恢复后再扣除 80 得到 -60。这个顺序并非每次都会发生，但说明挂起前成立的数量条件不能直接当作恢复后仍成立。

需要作为一个整体完成的检查和状态更新，可以放在同一个没有挂起点的 actor 隔离代码段中。下面的修改先检查并扣除数量，再让出执行机会；两个请求分别取走 80 时，最多只有一个请求成功。

```swift
func take(_ amount: Int) async -> Bool {
    guard remaining >= amount else { return false }
    remaining -= amount
    await Task.yield()
    return true
}
```

这个方法片段替换 Quota 中的 take 方法。如果实际异步步骤可能失败，还需要按具体操作约定设计失败后的状态恢复。另一种处理方式是在 await 后重新检查状态，再决定是否更新。

## 类型、实例与对象引用

Person 是一个具体类型名；class 是声明类类型的关键字，struct、enum 是其他类型的声明方式。“引用类型”是一类类型的分类；“引用”本身通常说的是指向对象的值，不能把这些层次混为一谈。

```swift
class Person {}
let person = Person()
```

Person 是类型，Person() 创建该类的一个实例，person 变量保存指向这个实例的引用。struct 和 enum 同样有实例，例如 `Point(x: 1, y: 2)` 和 `Direction.left` 都是相应类型的具体值。

Swift 语境中的“对象”通常指 class 实例；“实例”也包括 struct 和 enum 的具体值。

## 符号引用与对象引用

**在代码搜索语境中，符号（symbol）是编译器或 IDE 用来表示代码实体的标识。** 类型、变量、属性、方法和类型别名都可以对应符号。符号不只有名称，还关联所属作用域、类型、声明位置等信息。两个类分别声明同名的 `actionHandler`，对应的是两个不同的属性符号；名称相同不等于同一个符号。

**符号引用（reference）是源码中使用某个已声明符号的位置。** 方法调用属于符号引用，但引用不一定执行方法：

```swift
class ViewModel {
    func reload() {
        // 方法实现
    }
}

let vm = ViewModel()
vm.reload()             // 引用 reload，同时调用方法
let handler = vm.reload // 引用 reload，保存为函数值，此时不执行方法体
```

`func reload() { ... }` 是定义位置；下面两处对 `reload` 的使用是引用位置。属性读取和写入也属于符号引用。注释和普通字符串内容中的同名文字不构成对该符号的引用。

**符号引用与引用类型中的对象引用是不同概念：**

| 概念 | 描述的关系 |
|-|-|
| 符号引用 | 源码中的某处使用了哪个声明，可用于查找使用位置与代码导航 |
| 对象引用 | 运行时的一个值指向哪个对象实例，多个变量可以引用同一个实例 |

```swift
class Box {
    var count = 0
}

let a = Box()
let b = a
```

在 `let b = a` 中，右侧 `a` 是对变量符号 `a` 的引用；运行时，因为 `Box` 是引用类型，`b` 获得指向同一个 `Box` 实例的引用，没有复制出第二个实例。

值类型变量同样可以被源码引用。例如 `let number = 10; print(number)` 中，`print(number)` 使用了变量符号 `number`，但 `Int` 仍是值类型。源码里出现一次符号引用，不等于因此建立了一次 ARC 强引用或弱引用。对象引用与指针的区别另见[引用与指针](swift-objc-runtime.md#对象引用与-unsafepointer)。

## 声明、定义、实现与引用

| 概念 | 方法场景中的含义 |
|-|-|
| 声明（declaration） | 告诉编译器方法名称、参数、返回值等信息 |
| 定义（definition） | 给出方法体；带方法体的定义通常同时也是声明 |
| 实现（implementation） | 方法体中完成任务的具体代码，不是固定独立于“定义”的第三个源码位置 |
| 引用（reference） | 使用已声明方法的位置，包括调用方法或把方法保存为函数值 |

“方法的定义和实现分开”通常应改为“方法的声明和实现分开”。例如 Objective-C 的头文件与实现文件中，可以分别出现同一个方法的声明和定义：

```objc
// Demo.h
#import <Foundation/Foundation.h>
@interface Demo : NSObject
- (void)reload;
@end

// Demo.m
#import "Demo.h"
@implementation Demo
- (void)reload {
    // 具体实现
}
@end
```

当搜索包含声明和定义且索引完整时，两处可以分别形成搜索结果。如果还包含引用，调用位置也会出现；不能保证结果恰好有两条。这个例子是**同一个方法符号对应不同源码位置**。协议中的方法要求和遵循类型的具体实现则是相关但不同的声明，不能直接当作同一个方法在头文件与实现文件中的两处出现。

这与[属性、getter 和 setter](../engineering/xcode-and-testing.md#属性访问器与索引记录)的例子不同：属性与两个访问器是三个不同符号，可以关联到同一源码位置。**搜索结果条数不等于不同符号的数量。**

## Swift 与 C 接口互操作

C 接口指公开函数、类型和调用约定采用 C 兼容形式。例如 CFRunLoopSourceSignal 是一个 C 风格函数，Swift 可以通过导入框架调用它。接口形式描述调用者怎样使用库；实现语言描述库内部怎样编写，两者是不同维度。

Apple 的 Core Foundation 历史设计文档说明其以 C 实现。不过，不能只凭接口是 C 风格，就断言某个当前系统版本的所有内部实现都只使用 C。使用系统框架时，应先掌握公开接口的行为契约，不依赖未经核实的私有实现细节。

调用示例见[iOS 线程与任务调度中的 Source0 示例](../ios/threads-and-scheduling.md#source0-示例后台压缩完成后刷新预览)。

## 存储属性、计算属性与访问器

getter 负责读取属性，setter 负责写入属性。`let x = person.age` 是读取，`person.age = 20` 是写入；`newValue` 是 setter 接收的新值的默认参数名。

```swift
class Person {
    var age = 18
}
```

age 是存储属性，值保存在实例内部。编译器提供相应的访问能力，概念上可理解为“getter 读取 age 的存储，setter 把新值写进去”。下面只是等价语义示意，不是编译器生成的源码，也不表示实际多出两个 age 字段：

```swift
// 以下成员位于 Person 类内部
private var ageStorage = 18

var age: Int {
    get { ageStorage }
    set { ageStorage = newValue }
}
```

计算属性自己不新增对应的实例存储字段，而是运行 get／set 中的代码；代码可以读写其他存储。优化后，简单访问可能直接变成内存读写，不要求机器码里始终存在一次独立的 getter／setter 函数调用。

Xcode 索引可以将实例属性的隐式 getter／setter 分类为 `InstanceMethod`，子类别分别为 `AccessorGetter`／`AccessorSetter`，因此显示 M 图标；属性本身显示 P。声明处与赋值处的具体记录见[P／M 图标与源码位置](../engineering/xcode-and-testing.md#属性访问器与索引记录)。索引中的访问器记录不能单独证明优化后机器码仍保留独立函数调用。

### 实例存储属性与局部变量的初始化

初始化是第一次为属性或变量设置值。实例存储属性由每个实例分别保存；局部变量是在函数或方法内部声明、在对应代码范围内使用的变量。声明时没有写初值，不等于不需要初始化。

非 `lazy` 的实例存储属性必须在实例成功初始化完成前获得初始值。初值可以写在属性声明处，也可以由 `init` 设置。`init` 是创建实例时执行初始化工作的构造器；非可选的 `Int` 属性不会仅因为声明了类型就自动获得 `0`。

```swift
struct Person {
    var age: Int          // 声明时没有写初值

    init(age: Int) {
        self.age = age    // 创建实例时设置初值
    }
}

let person = Person(age: 18)
print(person.age)         // 18
```

这里的 `age` 在声明处没有初值，但 `Person(age: 18)` 会通过 `init` 完成赋值。不能把实例成功创建后的任意一次属性赋值，作为补做初始化的方式。

局部变量可以先声明、后赋值，但首次读取前必须已经初始化。编译器必须能够确认：所有能执行到这次读取的路径，都已经给变量赋值。局部变量如果始终没有被读取，可以一直不赋值；这不表示未初始化的值可以被读取。

```swift
func showAge() {
    var age: Int          // 允许先声明

    // print(age)         // 编译错误：变量尚未初始化

    age = 18              // 第一次赋值，完成初始化
    print(age)            // 允许读取
}
```

可选存储属性也需要区分“没有被包装的值”和“尚未初始化”。例如，普通的 `var nickname: String?` 会默认以 `nil` 初始化。`nil` 表示当前没有 `String` 值，但属性已经处于确定的状态。这个默认规则不能直接套用到 `let nickname: String?`；可选常量仍需要在声明处或初始化过程中明确设置值。

`lazy` 存储属性允许延迟求出初始值。声明中的初始化表达式通常在第一次读取属性时求值，并保存结果；如果第一次读取前已经直接给属性赋值，就使用该值，不再执行原初始化表达式。因此，“实例初始化完成前必须获得初始值”的要求需要区分普通存储属性和 `lazy` 存储属性。

### 类的两阶段初始化与 self 使用时机

Swift 类的两阶段初始化，是先让继承链中各层引入的存储属性取得初始值，再执行进一步的定制操作。指定构造器是负责初始化本类引入的存储属性，并按继承关系调用父类指定构造器的 init。这里讨论普通、非 lazy 的存储属性。

子类的指定构造器必须先初始化子类自己引入的存储属性，再调用 super.init。父类构造器负责初始化父类引入的属性。不能先调用 super.init，再补做子类属性的首次赋值。

```swift
class Parent {
    let name: String

    init(name: String) {
        self.name = name
    }
}

class Child: Parent {
    let age: Int

    init(name: String, age: Int) {
        self.age = age
        super.init(name: name)
    }
}
```

第一阶段完成前，构造器不能调用实例方法、读取实例属性，或把 self 作为一个完整实例传给其他代码。self.age = age 是完成自身属性初始化所需的写入，不等于允许任意使用 self。子类自身属性初始化完成且 super.init 成功返回后，子类构造器可以调用实例方法。

这套顺序使整个实例在开始执行依赖实例状态的方法之前，已经具有有效的初始状态。对于继承得到且允许修改的属性，子类应在 super.init 之后再定制其值，避免先写入的值被父类初始化覆盖。

## 扩展中的类型属性与实例存储限制

对于这里讨论的普通非泛型类型，extension 可以添加 static 存储属性，也可以添加实例或类型计算属性。static 存储属于类型层面的共享状态，不要求每个实例新增字段，因此不违反“扩展不能添加实例存储属性”的限制。不能把普通类型的例子无条件推广到所有泛型或其他声明上下文。

```swift
struct Person {
    var age: Int
}

extension Person {
    static let adultAge = 18
    var isAdult: Bool { age >= Self.adultAge }
}
```

Person 本来的 age 是实例存储；adultAge 是类型属性；isAdult 是计算属性。struct 原始定义可以有实例存储属性，enum 可以通过关联值携带数据。“无法用 extension 添加实例存储字段”不等于“值类型不能保存数据”。

### 为什么扩展不能添加实例存储属性

Swift 的 `extension` 可以添加实例计算属性，但不能添加实例存储属性。实例计算属性在访问时执行代码，不为每个实例新增一个保存结果的字段；实例存储属性需要每个实例分别保存对应的值。

新增实例存储属性会影响实例的内存布局和初始化规则。内存布局是实例中各项数据的排列方式和占用空间。新增字段需要纳入这个布局；新增的普通存储属性还必须在实例成功初始化完成前获得初始值。

扩展可以位于其他代码库，也可以作用于拿不到源代码的类型。原类型已经编译的代码不会因为另一个代码库声明扩展，就自动改变实例布局和初始化过程。Swift 的扩展没有提供修改这些规则的机制，所以禁止添加实例存储属性。这是语言设计限制；即使扩展与原类型位于同一文件，也不能直接添加实例存储属性，不能把限制解释为编译器先读取了哪个文件。

上面的 `Person` 示例没有手写 `init`，Swift 会为这个结构体自动生成接收 `age` 的构造器，因此可以通过 `Person(age: 18)` 创建已初始化的实例。扩展中的 `isAdult` 读取已有的 `age` 并计算结果；`adultAge` 属于类型层面的共享存储。这两项都不要求给每个 `Person` 实例增加存储字段。

## 协议存在类型与泛型约束

本节使用的 `Speaker` 协议，以及 `Dog`、`Cat` 类型，定义见[协议见证表示例](swift-objc-runtime.md#协议见证表与遵循关系)。

```swift
func makeSound(_ value: any Speaker) {
    value.speak()
}

func makeGenericSound<T: Speaker>(_ value: T) {
    value.speak()
}
```

只传一个值并调用 speak 时，两者可以表现出相同结果。any Speaker 表示可以承载不同遵循类型的存在类型值；泛型 T 保留函数签名中具体类型之间的关系，函数对每个满足约束的 T 都适用。

更容易看出区别的例子：

```swift
func sameType<T: Speaker>(_ first: T, _ second: T) {}
func possiblyDifferent(_ first: any Speaker, _ second: any Speaker) {}
```

前者要求两个参数能统一为同一个 T；对前面的两个独立 struct Dog 和 Cat，不能把它们直接作为同一个 T 传入。后者允许分别传入 Dog 和 Cat。泛型并不保证每次都静态派发，any 也不保证每次都发生堆分配，具体表示与优化有关。

能否把 `any Speaker` 简写为 `Speaker`，取决于 Swift 版本、语言模式和 ExistentialAny 等设置，不能仅凭 Swift 6 版本号作判断。示例显式写 `any`，便于与泛型约束区分。

## 逃逸闭包与执行时机

**`@escaping` 标记函数参数中的闭包可以逃出当前函数调用的生命周期。** 例如，函数把闭包保存到属性中，闭包就可以在函数返回之后被调用。它不会自动创建线程、安排任务或执行闭包。

```swift
final class CallbackHolder {
    var pending: (() -> Void)?

    func register(_ callback: @escaping () -> Void) {
        pending = callback
    }
}

let holder = CallbackHolder()
holder.register {
    print("回调被执行")
}
// register 已经返回，闭包保存在 pending 中。
holder.pending?()
```

执行过程是：`register` 保存闭包并返回，之后调用 `pending?()` 才执行闭包。保存闭包使它有机会在函数返回后使用，因此参数需要 `@escaping`。

如果函数只在自己的调用期间直接执行参数闭包，就不需要因为参数名叫 `callback` 而加 `@escaping`。被标记的闭包也可以同步执行；是否异步、在哪个线程执行，由实际调度代码决定。

**省略 `@escaping` 时的编译限制。** 对于上例中的 `() -> Void` 闭包参数，不加 `@escaping` 时，参数默认是非逃逸的。非逃逸表示接收函数不能让这个闭包保留到函数返回之后。

如果把上例的 `register` 改为下面的写法，编译器会在 `pending = callback` 处报错：

```swift
func register(_ callback: () -> Void) {
    pending = callback // 编译错误：不能把非逃逸闭包参数保存到这里
}
```

`pending` 是 `CallbackHolder` 的实例属性。`register` 返回后，调用方仍可以通过 `holder.pending` 访问闭包。把 `callback` 存入这个属性会让闭包逃逸，因此参数必须声明为 `@escaping`。

去掉 `@escaping` 后，这段代码无法通过编译，因此不会执行到 `holder.pending?()`。编译器在保存闭包的位置检查是否允许逃逸，不会等到后续调用闭包时才检查。

## 闭包参数、捕获列表与可选回调

闭包可以根据上下文推断参数类型。省略显式参数名称时，`$0` 表示第一个参数，`$1` 表示第二个参数。使用这种简写时，也省略参数列表与闭包体之间的 `in`。

```swift
let actionHandler: (String, Bool) -> Void = {
    print($0, $1)
}
// 等价写法
let namedHandler: (String, Bool) -> Void = { item, selected in
    print(item, selected)
}
```

上例中，变量类型已经规定闭包接收 `String` 和 `Bool`。编译器据此推断 `$0`、`$1` 的类型。省略名称没有减少参数数量，调用时仍需提供两个参数。

捕获列表用于指定闭包如何保存来自外部作用域的值。`[weak self]` 是捕获列表，不是参数列表。弱捕获不会让闭包强持有 `self`；当对象释放后，闭包中的 `self` 为 `nil`。

```swift
// 上下文已规定 actionHandler 的两个参数类型
// 简写
actionHandler = { [weak self] in
    self?.handle(item: $0, selected: $1)
}
// 显式参数名
actionHandler = { [weak self] item, selected in
    self?.handle(item: item, selected: selected)
}
```

`[weak self]` 后面的 `in` 用于结束闭包头部。即使省略了显式参数名，只要写了捕获列表，这个 `in` 仍需保留。闭包体使用的外部变量属于捕获值，不会自动占用 `$0` 或 `$1` 的位置。

## 多行字符串与脚本执行

**普通字符串用一对双引号包围；多行字符串用开头三个和结尾三个双引号包围。** 三引号属于 Swift 的字符串语法，不是 JavaScript 的语法。

```swift
let script = """
window.receiveProduct("123");
"""
```

开头是 `"""`，结尾也是 `"""`。内部的 `"123"` 是字符串内容的一部分；它表示之后交给 JS 执行的代码中的字符串参数。

上面的内容也可以用普通字符串表示：

```swift
let script = "window.receiveProduct(\"123\");"
```

普通字符串中，`\"` 表示内容里的双引号，避免被当作 Swift 字符串的结束标记。两种写法在此示例中得到相同的字符串内容。创建这个字符串不会执行 JS，调用 `evaluateJavaScript` 后才由 WebKit 执行，见[Native 调用前端](../cross-platform/webview-and-lynx-bridge.md#wkwebviewnative-执行-javascript)。

## map、KeyPath、Sequence、Collection 与 Set

沿用[策略示例](../engineering/design-patterns.md#策略模式与评论排序)的 result（[Comment]），这是 map 配合 KeyPath 字面量的写法，等价于：

```swift
result.map(\.text)

result.map { $0.text }

result.map { comment in
    comment.text
}
```

- map 对每个元素执行转换，生成新的数组。
- \$0 是闭包的第一个参数，这里是当前 Comment。
- \\.text 表示读取元素的 text 属性，根类型由上下文推断。

result 是 [Comment]，text 是 String，因此结果是 [String]。完整键路径也可写成 `result.map(\Comment.text)`。这里编译器可以将键路径字面量用于需要 `(Comment) -> String` 的位置；不应无条件推广为任意已保存的 KeyPath 变量都可直接当闭包传入。

**键路径：**

KeyPath 是描述“怎样从某个根值访问属性”的值，可以保存在变量里或作为参数传递。下面沿用[策略示例](../engineering/design-patterns.md#策略模式与评论排序)中的 Comment 类型：

```swift
let textPath: KeyPath<Comment, String> = \Comment.text
let comment = Comment(text: "你好", likeCount: 2, publishedAt: 100)

let firstText = comment[keyPath: textPath]
let secondText = comment.text
// 都读到 "你好"
```

KeyPath<Comment, String> 的第一个类型参数是根类型 Comment，第二个是沿路径读出的值类型 String。

路径本身没有保存某条评论的文字；应用到不同 Comment，可以读到不同的 text。它也不只适用于 class，前面的 Comment 就是 struct。上下文已知根类型时，\Comment.text 可以简写为 \\.text。

**直接属性访问与键路径的用途。** 只需要读取某个实例的固定属性时，直接使用 `comment.text` 即可。键路径的用途是把“访问哪个属性”表示成一个可以保存、传递和复用的值。

```swift
let text = comment.text  // 保存当前读到的字符串："你好"
let path = \Comment.text // 保存访问路径：读取 Comment 的 text 属性
```

`text` 保存本次读取的字符串。`path` 描述从一个 `Comment` 实例读取 `text` 属性的路径，不绑定某个具体的 `comment`，也不保存本次读取的结果。使用 `comment[keyPath: path]` 时，程序才从传入的实例读取对应属性。

当函数需要让调用方决定读取哪个属性时，可以把键路径作为参数传入。下面沿用前面的 `Comment` 类型和 `comment` 实例，用同一个函数读取评论文字或点赞数：

```swift
func collect<Value>(
    _ comments: [Comment],
    by path: KeyPath<Comment, Value>
) -> [Value] {
    comments.map { comment in
        comment[keyPath: path]
    }
}

let comments = [comment]

let texts = collect(comments, by: \Comment.text)           // ["你好"]
let likeCounts = collect(comments, by: \Comment.likeCount) // [2]
```

`Value` 是泛型类型参数，表示所选属性的值类型。传入 `\Comment.text` 时，`Value` 是 `String`，函数返回 `[String]`；传入 `\Comment.likeCount` 时，`Value` 是 `Int`，函数返回 `[Int]`。

调用方先传入评论数组和键路径。`collect` 通过 `map` 遍历每个 `Comment`，按照传入的路径读取属性，再把结果组成数组。函数实现保持不变，调用方通过不同键路径选择不同属性。

如果函数内部固定写 `comment.text`，函数就固定读取 `text`，不能通过上述参数选择 `likeCount`。键路径适用于需要传递或复用属性访问方式的场景；单次读取固定属性不需要引入键路径。

**容器与协议：**

Array、Set、Dictionary 都遵循 Sequence，也遵循 Collection。

| 类型 | 遍历得到的元素 | 顺序 |
|-|-|-|
| Array<Int> | 一个 Int | 按数组索引顺序 |
| Set<Int> | 一个 Int | 不保证插入顺序或排序顺序 |
| Dictionary<String, Int> | 一个 (key: String, value: Int) 元组 | 不保证插入顺序或排序顺序 |

```swift
let ages = ["小明": 18, "小红": 20]
for element in ages {
    print(element.key)
    print(element.value)
}

let numbers: Set<Int> = [1, 2, 3]
let doubled = numbers.map { $0 * 2 }
// doubled 是 [Int]，不保证按 2、4、6 排列

let names = ages.map { $0.key }
// names 是 [String]，不保证按插入顺序排列
```

上面 map 的结果都是数组，不会自动维持输入的 Set 或 Dictionary 类型。Sequence 强调能够逐个遍历，不要求按插入顺序排列；“顺序不保证”也不等于同一容器每次遍历必然随机变化。

`Sequence` 规定逐个遍历的能力；`Collection` 在此基础上增加可多次遍历和按索引访问等约定。`Set` 是保存不重复元素的具体类型。一般意义上的集合、Collection 协议与 Set 类型需要分开。

### 数组值语义、写时复制与返回值

对 [Int] 而言，值语义保证数组赋值后，修改一个数组的元素不会改变另一个数组的元素。值语义描述程序能观察到的行为，不要求每次赋值或返回数组时立即复制全部元素。元素存储空间是保存数组元素的内存区域，需要与数组值本身区分。

Array 使用写时复制：多个数组值可以先共享元素存储空间；当某个数组需要修改元素且存储空间仍被共享时，修改方先取得独立的元素存储空间，再完成修改。编译器可以在保持行为不变的前提下减少实际复制。

```swift
let original = [1, 2, 3]
var copy = original
copy[0] = 99

print(original) // [1, 2, 3]
print(copy)     // [99, 2, 3]
```

函数返回局部数组后，如果局部变量结束生存期，而且只有接收结果的变量继续持有元素存储空间，修改已有元素可以直接复用原来的存储空间。是否需要复制，取决于修改时是否仍有其他持有者共享存储，不取决于之前是否发生过返回或赋值。

```swift
func makeNumbers() -> [Int] {
    let numbers = Array(0..<1_000_000)
    return numbers
}

var preview = makeNumbers()
preview[0] = -1 // 只有 preview 持有存储时，可以原地修改

let backup = preview
preview[0] = -2
print(backup[0]) // -1：backup 仍需保留修改前的值
```

当存储仍被共享时，修改方需要能够保存全部现有元素的独立存储；实际分配容量不保证与旧存储完全相同。容量是无需重新分配就能容纳的元素数量。即使数组独占存储，追加元素超过容量时，也可能需要重新分配。

### ArraySlice 的存储保留与索引

ArraySlice 表示数组中一段连续元素。创建切片时，切片通常复用原数组的元素存储空间，并记录切片范围；切片本身也是值类型。切片值独立，不代表切片立即拥有独立的元素存储空间。

```swift
func makePreview() -> ArraySlice<Int> {
    let numbers = Array(0..<1_000_000)
    return numbers.suffix(10)
}

let preview = makePreview()
print(preview.count)      // 10
print(preview.startIndex) // 999990
print(preview.endIndex)   // 1000000
```

suffix(10) 取出最后 10 个元素。上述 preview 仍保留原数组的整块元素存储空间，即使 numbers 已结束生存期。返回切片或给切片赋值，不会自动把这 10 个元素复制成独立数组。只需要长期保存少量元素时，长期持有切片可能延长大量无关元素及存储空间的生存期。

切片保留元素在原数组中的索引。上述有效索引为 999990 到 999999，preview[0] 会越界。startIndex 是起始位置；endIndex 是末尾之后的位置，不能用于读取元素。切片非空时，可以用 preview[preview.startIndex] 读取第一个元素；可能为空时，可以使用返回可选值的 preview.first。

需要长期保存切片中的元素时，可以通过 Array(...) 构造数组。在下面的示例中，只把末尾 10 个整数复制到新数组，新数组的索引从 0 开始。

```swift
func makePreviewArray() -> [Int] {
    let numbers = Array(0..<1_000_000)
    return Array(numbers.suffix(10))
}

let preview = makePreviewArray()
print(preview.count) // 10
print(preview[0])    // 999990
```

新数组不再保留原数组的大块存储空间。原存储在没有其他持有者后可以释放；释放一块存储不等于进程的物理内存统计一定立即下降，因为内存分配器可能保留可复用的空间。

## 元组、字典与空元组 Void

元组把固定数量的值组合成一个值。每个位置都有确定的类型，位置之间可以使用不同类型，且元组无法添加新元素。字典保存键到值的映射；同一个字典的键遵循统一的 `Key` 类型，值遵循统一的 `Value` 类型，键必须满足 `Hashable` 约束。

| 比较项 | 元组 | 字典 |
|-|-|-|
| 结构 | 元素数量和各位置类型由元组类型确定 | 可变字典可添加或删除键值对 |
| 访问方式 | 按位置或元素标签访问，如 `.0`、`.name` | 按键访问，如 `["小明"]` |
| 顺序 | 元素位置有明确顺序 | 不保证按插入顺序或排序顺序遍历 |
| 查询结果 | 已声明的元素具有对应位置的类型 | 普通按键下标返回 `Value?`，因为键可能不存在 |

```swift
let person: (name: String, age: Int) = ("小明", 20)
print(person.name)  // 与 person.0 访问同一个元素
print(person.age)

let ages: [String: Int] = ["小明": 20, "小红": 22]
let age: Int? = ages["小明"]
let missing: Int? = ages["小王"] // nil
```

元组标签是元组结构的一部分，不是字典中的可动态查询的键。上例的元组适合组合某个人的不同字段，字典适合按姓名查找同一种年龄数据。

`Void` 是空元组类型 `()` 的别名。空元组有且只有一个值，也写作 `()`；这个值不包含任何元素。`Void` 不是 `nil`，也不是“没有可构造的值”。例如 `() -> Void` 的左侧 `()` 表示零个参数，右侧 `Void` 表示返回空元组。

## 相等规则与对象身份

| 运算符 | 判断内容 |
|-|-|
| === | 两个对象引用是否指向同一个实例，即身份相同 |
| == | 是否按照类型提供的规则相等 |

对[单例示例](../engineering/design-patterns.md#单例共享状态与依赖传入)中的纯 Swift PostHistoryManager：

```swift
let first = PostHistoryManager.shared
let second = PostHistoryManager.shared

first === second // true
// first == second // 当前定义不支持，编译错误
```

它没有提供 == 比较能力。类型通常通过遵循 Equatable 并实现相等规则来支持 ==。类不会仅因为两个引用实际相同，就自动获得 == 实现。

例如两个不同的用户对象，可以按用户 ID 相同被定义为 == true，但它们仍是两个实例，所以 === false。判断单例是否为同一对象，应使用 ===。这里的编译错误结论限定当前示例类型，不应推广为所有类都不能用 ==。

=== 用于比较类实例的身份，也支持可选对象引用。struct 和 enum 是值类型，不能直接使用它比较。

```swift
let a = [1, 2]
let b = [1, 2]

a == b // true，元素相等
// a === b // 编译错误，Array 是值类型
```

不能用 === 判断两个数组是否内容相等，也不能据此检查其底层写时复制缓冲区是否共享。后者不是本次语言级对象身份比较所表达的含义。

## defer 与作用域退出

defer 是 Swift 的延后执行语句，用于在离开当前作用域之前执行一段代码。这里的作用域指包围 defer 的代码块，例如函数体或 do 代码块。程序执行到 defer 语句时，不会立即执行其中的代码。

只有程序已经执行到某条 defer 语句，这条 defer 才会在退出对应作用域时执行。退出方式可以是正常执行完代码块、通过 return 提前返回，或通过抛出 Swift 错误离开作用域。如果程序在执行到 defer 之前就退出，defer 中的代码不会执行。

defer 的执行位置由所在作用域决定。写在函数体内的 defer 在函数退出前执行；写在内层 do 代码块里的 defer 在离开该代码块前执行，不必等到整个函数结束。同一作用域内，如果执行到了多条 defer，退出时按遇到它们的相反顺序执行。

下面的例子使用 defer 统一解锁。NSLock 是互斥锁，同一时刻只允许一个线程持有。lock.lock() 获取锁，lock.unlock() 释放锁。

```swift
import Foundation

func performWork(lock: NSLock, shouldSkip: Bool) {
    lock.lock()
    defer { lock.unlock() }

    if shouldSkip {
        return
    }

    print("执行操作")
}
```

函数先获取锁，再执行到 defer 语句，并继续运行后续代码。shouldSkip 为 true 时，函数在返回调用方之前解锁；shouldSkip 为 false 时，函数输出文字，并在函数结束之前解锁。多个退出位置可以共用同一条 defer，避免分别编写解锁代码。

defer 不会主动结束函数，也不会超时解锁。如果函数一直等待另一把锁、重复获取自己已经持有的 NSLock，或者进入无限循环，函数就无法退出，defer 中的解锁代码也无法执行。因此，使用 defer 不能代替对加锁顺序和重复加锁的检查。
