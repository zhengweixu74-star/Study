# Swift 语言基础

[返回笔记目录](../README.md)

## 本文目录

- [基础语义与内存管理](#基础语义与内存管理)
  - [值类型方法中的 mutating](#值类型方法中的-mutating)
  - [普通函数参数与局部变量](#普通函数参数与局部变量)
  - [inout 与内存独占访问](#inout-与内存独占访问)
  - [ARC 与 Android GC：引用计数、可达性与引用环](#arc-与-android-gc引用计数可达性与引用环)
  - [结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)
  - [weak 与 unowned 的使用条件](#weak-与-unowned-的使用条件)
  - [闭包与 Delegate](#闭包与-delegate)
  - [协议继承与类继承的区别](#协议继承与类继承的区别)
- [类型安全、值语义与动态能力](#类型安全值语义与动态能力)
  - [值类型与引用类型的选择](#值类型与引用类型的选择)
  - [静态类型、类型推断与数值转换](#静态类型类型推断与数值转换)
  - [可选类型、状态表达与并发支持](#可选类型状态表达与并发支持)
  - [可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)
  - [集合、泛型与协议约束](#集合泛型与协议约束)
  - [函数签名与回调类型](#函数签名与回调类型)
  - [运行时类型转换与安全边界](#运行时类型转换与安全边界)
  - [actor 的默认隔离与重入](#actor-的默认隔离与重入)
  - [MainActor、主队列与异步调用](#mainactor主队列与异步调用)
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
- [类型、实例与对象引用](#类型实例与对象引用)
- [引用赋值、浅拷贝与深拷贝](#引用赋值浅拷贝与深拷贝)
  - [Swift 中的深拷贝实现方法](#swift-中的深拷贝实现方法)
- [Codable 与序列化](#codable-与序列化)
  - [CodingKeys 的作用与枚举语法](#codingkeys-的作用与枚举语法)
- [序列化复制的边界](#序列化复制的边界)
  - [未编码的属性不会保留原值](#未编码的属性不会保留原值)
  - [自定义解码可能重新使用共享对象](#自定义解码可能重新使用共享对象)
  - [原来共享的对象可能被拆成两个对象](#原来共享的对象可能被拆成两个对象)
  - [循环引用需要显式记录对象关系](#循环引用需要显式记录对象关系)
- [符号引用与对象引用](#符号引用与对象引用)
- [声明、定义、实现与引用](#声明定义实现与引用)
- [Swift 与 C 接口互操作](#swift-与-c-接口互操作)
- [存储属性、计算属性与访问器](#存储属性计算属性与访问器)
  - [实例存储属性与局部变量的初始化](#实例存储属性与局部变量的初始化)
  - [指定初始化方法与便利初始化方法](#指定初始化方法与便利初始化方法)
  - [required、override 与初始化方法继承](#requiredoverride-与初始化方法继承)
  - [UIViewController 的 init?(coder:) 与纯代码初始化](#uiviewcontroller-的-initcoder-与纯代码初始化)
  - [类的两阶段初始化与 self 使用时机](#类的两阶段初始化与-self-使用时机)
- [扩展中的类型属性与实例存储限制](#扩展中的类型属性与实例存储限制)
  - [为什么扩展不能添加实例存储属性](#为什么扩展不能添加实例存储属性)
- [协议存在类型与泛型约束](#协议存在类型与泛型约束)
  - [泛型保留类型关系，any 隐藏具体静态类型](#泛型保留类型关系any-隐藏具体静态类型)
  - [some 用于返回值和存储属性](#some-用于返回值和存储属性)
  - [some 和 any 用于参数](#some-和-any-用于参数)
- [逃逸闭包与执行时机](#逃逸闭包与执行时机)
- [闭包参数、捕获列表与可选回调](#闭包参数捕获列表与可选回调)
  - [默认捕获与捕获列表](#默认捕获与捕获列表)
  - [捕获对象引用与对象内容](#捕获对象引用与对象内容)
  - [捕获存储与函数返回](#捕获存储与函数返回)
- [多行字符串与脚本执行](#多行字符串与脚本执行)
- [Data、字符串解码与打印](#data字符串解码与打印)
  - [UTF-8 解码与 JSON 模型解码](#utf-8-解码与-json-模型解码)
  - [可选字符串的打印与换行](#可选字符串的打印与换行)
- [map、KeyPath、Sequence、Collection 与 Set](#mapkeypathsequencecollection-与-set)
  - [集合类型与 for-in 遍历](#集合类型与-for-in-遍历)
  - [stride 的步长、方向与边界](#stride-的步长方向与边界)
  - [数组的 last 属性](#数组的-last-属性)
  - [数组反转：reverse() 与 reversed()](#数组反转reverse-与-reversed)
  - [数组值语义、写时复制与返回值](#数组值语义写时复制与返回值)
  - [ArraySlice 的存储保留与索引](#arrayslice-的存储保留与索引)
- [元组、字典与空元组 Void](#元组字典与空元组-void)
- [相等规则与对象身份](#相等规则与对象身份)
- [defer 与作用域退出](#defer-与作用域退出)

## 基础语义与内存管理

**ARC 与引用关系。** ARC（自动引用计数）按强引用的数量管理类实例的生命周期。编译器在代码中插入增减计数的操作：新增一条强引用时计数加 1，去掉一条强引用时计数减 1，计数归零时实例被释放。weak 引用不计入强引用计数，所以不会让实例继续存活。闭包捕获 self 时，要看 self 是否也直接或间接强引用这个闭包：两者互相强引用才形成引用环；没有形成引用环时，捕获只是让 self 活到闭包释放为止。ARC 与垃圾回收（GC）的区别见 [ARC 与 Android GC](#arc-与-android-gc引用计数可达性与引用环)。

**协议与继承。** 协议定义能力要求，class、struct、enum 可遵循多个协议。类采用单继承，并通过继承复用实现。协议本身只声明要求；协议的 extension 可以为要求提供默认实现，遵循类型没有自己实现时使用默认实现。

**可选绑定。** `if let` 与 `guard let` 的条件检查、取值和作用域见[可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)。

**实例 lazy。** 普通实例的 lazy 存储属性在首次读取时执行初始化并保存结果，后续读取已有值。lazy 适合初始化开销大、又不一定会用到的属性。多个线程同时首次读取实例 lazy 属性时，初始化代码可能执行多次；`static let` 则保证只初始化一次，并且线程安全。

**错误处理。**`throws` 表示函数可能抛错；`try` 本身不捕获错误，应使用 `do-catch` 捕获或向外传播。`try?` 在抛错时得到 nil 并丢弃具体错误；`try!` 在抛错时直接触发运行时错误，使程序崩溃。

### 值类型方法中的 mutating

`mutating` 表示值类型的实例方法可以修改当前实例的值，包括给存储属性赋值或替换整个 `self`。普通结构体和枚举的方法要执行这些修改时，需要声明为 `mutating`。类的方法修改对象属性时，不需要这个修饰符。

```swift
struct Counter {
    var count = 0

    mutating func increment() {
        count += 1
    }
}

var counter = Counter()
counter.increment()
counter.count = 10

let fixed = Counter()
// fixed.increment() // 编译错误：不能修改 let 保存的结构体值
```

`var count` 使属性可修改，`mutating` 允许方法修改当前结构体值，调用位置的 `var counter` 允许保存修改后的值。这三个条件分别约束属性、方法和调用位置。直接执行 `counter.count = 10` 不需要在赋值语句前写 `mutating`。

```swift
final class CounterObject {
    var count = 0

    func increment() {
        count += 1
    }
}

let object = CounterObject()
object.increment() // 允许：对象中的 count 是可写属性
// object = CounterObject() // 编译错误：let 不允许更换对象引用
```

用 `let` 保存类引用时，`let` 只禁止变量改为指向另一个对象，对象的 `var` 属性仍然可以修改。同理，如果结构体的属性保存类引用，非 `mutating` 方法也可以修改被引用对象的可写属性：修改发生在对象上，结构体保存的引用没有变，结构体自身的值也没有变。

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

参数类型是 class 时，“不能重新赋值”只禁止参数改为指向另一个对象。对象的属性可写时，函数仍然可以修改这些属性。用参数初始化一个 `var` 局部变量，只是多了一条指向同一对象的引用，对象不会被复制。

这里的普通参数不包括 inout 参数。inout 参数允许函数修改传入变量的值，并在函数返回时将修改写回调用方。

### inout 与内存独占访问

inout 参数允许函数修改调用方传入的变量，调用时在变量前加 &。对普通变量的一次修改访问尚未结束时，其他独立访问不能读取或修改同一块内存；这个要求称为内存独占访问。函数内部通过这个 inout 参数读取和修改值，属于同一次访问，不构成冲突。

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

两条加法语句虽然先后执行，但 first 和 second 的修改访问都从调用开始、到函数返回才结束，两段访问时间仍然重叠。即使函数体完全不使用 second，只要 second 声明为 inout，`update(&value, &value)` 仍然不合法。这类冲突在单线程中也会发生；Swift 在编译时或运行时检查，本例在编译时就能发现。

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

Swift 类实例通常由 ARC（自动引用计数）管理。ARC 跟踪对象的强引用计数；当强引用计数归零时，对象可以释放。Android 上的 Kotlin／Java 对象主要由追踪式垃圾回收（GC）管理。GC 从一组起点出发，沿对象引用找出仍然可达的对象，再回收不可达的对象。

两种机制都只管理各自语言的对象，不管理手动申请的内存：

- Android 应用可以同时包含 Kotlin／Java 代码和 C/C++ 原生代码。原生代码是编译成机器指令直接执行的代码。原生代码用 `malloc` 申请的内存，必须由代码或负责这块内存的库调用 `free` 释放，GC 不会回收。
- Swift 用 `UnsafeMutablePointer.allocate` 手动申请的内存，必须调用 `deallocate()` 释放，ARC 不会释放。

强引用是会保持目标对象存活的对象引用。可达性表示能否从垃圾回收起点沿引用链到达某个对象。以下示意中的箭头全部表示强引用。

GC Roots 是垃圾回收器追踪对象时使用的一组起点，例如活动线程栈中仍然有效的对象引用。GC 从这些起点逐层访问其他对象。对象是否可达只取决于引用链，与对象当前是否在执行方法无关。

`GC Root → A → B`

在上面的关系中，GC 能从起点找到 A，再从 A 找到 B，因此 A 和 B 都不能作为不可达对象回收。

引用环是沿对象之间的引用可以回到起点的关系，例如 A 强引用 B，同时 B 强引用 A。假设 A 和 B 都是普通对象，最初只有一条外部强引用连接这个引用环：

`最初：外部强引用 → A ⇄ B`

`断开唯一外部强引用后：A ⇄ B`

| 机制 | 断开唯一外部强引用后的结果 |
| --- | --- |
| Swift ARC | A 和 B 各自仍被对方强引用，强引用计数没有归零，因此不能自动释放。 |
| Android GC | 如果从所有 GC Roots 出发都无法到达 A 和 B，整个引用环可以在后续垃圾回收中回收。回收不保证立即发生。 |

Swift 中可以按对象所有权关系把合适的一条引用改为 weak，或者在业务结束时明确断开强引用。weak 不保持目标对象存活，所以只能用在不负责持有目标的那条引用上。负责持有对象的引用改成 weak 后，对象会被提前释放。

内存泄漏是对象已经不再需要，却仍然无法释放而持续占用内存。GC 能回收不可达的引用环，但不能回收仍通过强引用链可达的普通对象。例如：

`GC Root → 长期存活的单例 → 已关闭的 Activity`

只要这条强引用链存在，GC 就不能回收旧 Activity。页面关闭只改变页面的生命周期状态，不会删除其他对象指向页面的强引用；对象能否回收只取决于引用关系。因此，在 iOS 和 Android 上排查泄漏，都要检查谁持有页面、闭包或订阅，以及这些持有关系在什么时候解除。

### 结构体、枚举与值的生命周期

结构体和枚举的普通值也由 Swift 自动管理，不需要为值本身手动释放内存。与类实例不同，普通结构体和枚举值不依靠自身的强引用计数决定生命周期。编译器根据值的使用关系和存储方式安排必要的清理。以下示例均使用普通、可复制的结构体和枚举。

值本身占用的存储，与值内部持有的资源要分开看。只包含整数等简单成员的值，销毁时没有需要释放的对象引用；值占用的存储会被复用，或随所在的存储区域一起释放。结构体和枚举不一定存放在栈上，实际存储位置取决于使用方式和编译器实现。`Array` 虽然是结构体，其内部共享存储仍会涉及引用计数。

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

**作用域与生命周期。** 作用域是源代码中可以使用某个变量名的范围。生命周期是程序运行时，某一份值从创建到销毁的时间段。变量名离开作用域时，如果这份值已经被返回或保存到其他位置，值仍然存活。

```swift
func example() {
    let photo = Photo(image: Image())
    print(photo.image)  // 最后一次使用这份局部值
    // 后续代码不再使用 photo
}                      // photo 的作用域结束
```

**局部值的清理。** 如果 `photo` 只在当前作用域中使用，没有被返回，也没有被外部闭包捕获，编译器会在最后一次使用之后、作用域结束之前的某个位置插入清理代码。清理代码执行时，`photo` 的生命周期结束，`photo.image` 持有的强引用被释放。清理的具体位置由编译器决定，可能在 `}` 处，也可能更早。

```swift
func replacePhoto() {
    var photo: Photo? = Photo(image: Image())
    print(photo?.image as Any)
    photo = nil
    // 当前作用域还没有结束
}
```

**旧值被替换。** `photo = nil` 让变量不再保存原来的 `Photo` 值，原值持有的 `Image` 强引用随之释放，不必等到作用域结束。`Image` 实例是否在这时销毁，还要看是否存在其他强引用。即使没有其他强引用，编译器在不同构建配置下也可能调整释放时机，所以不能把这条赋值语句当作 `deinit` 的固定执行位置。

```swift
func makePhoto() -> Photo {
    let photo = Photo(image: Image())
    return photo
}

let result = makePhoto()
```

**值被返回或保存在外部。** `makePhoto()` 返回后，局部变量名 `photo` 已不可用，但返回的 `Photo` 值仍由 `result` 保存。`result` 中的 `image` 强引用仍可保持 `Image` 实例存活。如果值保存在对象的存储属性中，属性被替换或所属对象被销毁时会清理对应的旧值。如果闭包捕获了这份值或变量，并且闭包在作用域结束后仍然存活，值也会继续存活。创建值的函数返回，不会清除这些外部保存关系。

下面的 `Box` 保存一个值类型属性。把属性读出并赋给 `savedSize` 后，`savedSize` 保存独立的结构体值。

```swift
struct Size {
    var width = 10
}

final class Box {
    var size = Size()
}

var box: Box? = Box()
let savedSize = box!.size
box = nil // 示例中没有其他强引用，Box 可以销毁
print(savedSize.width) // 10
```

`Box` 销毁时，`Box.size` 中那份值的生命周期结束，已经复制到 `savedSize` 的值仍然有效。值内部的类引用或数组共享存储是否释放，仍取决于这些资源是否被其他值或对象持有。

### weak 与 unowned 的使用条件

`weak` 和 `unowned` 都不通过强引用保持目标类实例存活。区别在于目标销毁后如何处理这条引用。

| 声明 | 目标销毁后的行为 | 使用条件 |
| --- | --- | --- |
| `weak var target: T?` | 自动变成 `nil` | 允许目标先销毁，访问方需要处理目标不存在的情况 |
| `unowned let target: T` | 不会自动变成 `nil`；访问已销毁目标会触发运行时错误 | 每次访问时都能保证目标仍然存在 |
| `unowned var target: T?` | 可以主动赋为 `nil`，但不会在目标销毁时自动置空 | 程序必须保证每次读取时保存的是有效引用或 `nil` |

`weak` 适合可能先于调用方销毁的委托对象，以及页面销毁后应跳过操作的闭包捕获。`unowned` 适合生命周期关系已经得到保证的反向引用。例如，辅助对象只在所属对象存活期间使用，且程序能保证辅助对象不会在所属对象销毁后访问该反向引用。所属对象持有辅助对象，并不足以满足这个条件：辅助对象如果还被其他对象持有，就可能在所属对象销毁后继续被调用，并访问已失效的反向引用。

```swift
// 在页面的实例方法中设置回调；loader 保存该闭包。
loader.onFinish = { [weak self] in
    self?.handleResult()
}
```

页面已销毁时，闭包中的 `self` 为 `nil`，因此跳过 `handleResult()`。如果改成 `[unowned self]`，则必须保证闭包每次执行时页面都还活着。异步回调不必一律写成 weak。如果任务完成前需要对象一直存活，可以用强引用捕获；这时要确认任务结束后闭包会被释放，强引用环不会一直存在。

### 闭包与 Delegate

#### 概念与职责

**概念区别。** 闭包是可以作为值传递的代码块，闭包可以捕获定义位置的变量。在回调场景中，使用组件的一方提供具体行为，组件在约定的时机调用闭包。Delegate 是一种委托模式：一个对象把约定的职责交给另一个对象。Swift 通常通过协议定义这些职责，由遵循协议的对象实现对应方法。

#### 选型依据

**选型原则。** 先判断使用组件的一方要提供的是“一段回调行为”，还是“一个承担明确职责的对象”。只需要传入一段具体的回调行为时，选择闭包；需要由一个对象统一承担一组相关职责时，选择 Delegate。事件的数量和频率不决定选型。

| 判断维度 | 更适合闭包 | 更适合 Delegate |
| --- | --- | --- |
| 回调内容 | 一个或少量独立的结果、动作 | 一组共同描述某种职责的方法 |
| 代码组织 | 希望在发起操作的位置直接写回调 | 希望由接收对象集中实现相关方法 |
| 对象关系 | 只需要提供具体行为，无须指定接收对象的角色 | 需要明确某个对象担任事件接收者或决策者 |
| 状态管理 | 回调只使用少量局部变量 | 多个方法需要共享接收对象维护的状态 |

#### 使用场景

**闭包示例：图片导出。** 使用组件的一方发起一次导出，并传入处理导出结果的闭包。导出完成后，导出组件把成功结果或错误传给闭包。使用组件的一方在发起导出的位置直接编写后续行为，代码比较集中。

**Delegate 示例：编辑器交互。** 编辑器需要通知“开始编辑”“内容变化”“结束编辑”，还可能询问“是否允许结束编辑”。这些方法共同构成编辑器委托对象的职责。由同一个对象集中实现这些方法，职责更清楚。

#### 适用边界

**选型边界。** 调用次数和返回值都不决定选型。闭包可以反复调用，例如持续触发的进度回调；Delegate 方法也可以只调用一次。两种方式都能通知事件，也都能返回决策结果。

#### 生命周期与持有关系

**持有关系。** 组件保存闭包时，闭包可能强引用捕获的对象，因此需要检查循环引用。Delegate 属性通常声明为 weak，但 weak 不会保持委托对象存活，必须有其他对象负责持有委托对象。

下面用按钮通知页面说明委托调用。`ButtonDelegate` 定义通知方法，`Page` 实现这个方法，`Button` 在点击时调用委托对象。`AnyObject` 的类类型限制见[协议继承与类继承的区别](#协议继承与类继承的区别)。

```swift
protocol ButtonDelegate: AnyObject {
    func buttonDidTap(_ button: Button)
}

final class Button {
    weak var delegate: (any ButtonDelegate)?

    func tap() {
        delegate?.buttonDidTap(self)
    }
}

final class Page: ButtonDelegate {
    let button = Button()

    init() {
        button.delegate = self
    }

    func buttonDidTap(_ button: Button) {
        print("页面收到按钮点击事件")
    }
}
```

`Page.init()` 把页面设置为按钮的委托对象。调用 `button.tap()` 后，按钮读取 `delegate`；若委托对象存在，按钮调用页面的 `buttonDidTap`，并把按钮自身作为参数传入。页面收到通知后执行对应方法。`ButtonDelegate` 是协议，实际承担委托职责的对象是 `Page` 实例。

页面强引用按钮，按钮通过 `weak delegate` 引用页面。如果去掉 `weak`，这两个对象就会相互强引用。外部释放页面后，这个强引用环仍会让两个对象保持存活。

```swift
var page: Page? = Page()
let button = page!.button // 外部也持有这个按钮
button.tap()             // 页面收到按钮点击事件

page = nil               // 示例中没有其他强引用指向页面
button.tap()             // 按钮还在，但 delegate 已为 nil
```

页面持有按钮，只能保证页面存活期间按钮也存活；按钮存活时，页面不一定存活。上例中，页面销毁后，外部变量仍持有按钮。

如果把属性改成 `unowned var delegate: (any ButtonDelegate)? = nil`，页面销毁后 `delegate` 不会自动清空。随后调用 `button.tap()` 会在读取已失效的 `unowned` 引用时触发运行时错误。`delegate?.buttonDidTap(self)` 只能跳过值为 `nil` 的情况，不能保护已经失效的 `unowned` 引用。选择条件见[weak 与 unowned 的使用条件](#weak-与-unowned-的使用条件)。

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

协议继承只继承要求，不继承存储属性和方法实现。方法的默认实现来自协议扩展，与协议继承无关。类继承则会继承父类的存储属性和方法实现。

协议要求还需要区分必需和可选。普通 Swift 协议中的方法要求默认是必需的；适用 Objective-C 互操作的协议可以声明 optional 要求。UIKit 的 UICollectionViewDelegate、UIScrollViewDelegate 等协议包含可选回调，遵循这些协议的类可以只实现需要的回调。

`AnyObject` 是 Swift 的特殊协议，不是类。所有类类型都自动符合 `AnyObject`，不需要显式声明遵循关系。

```swift
protocol ButtonDelegate: AnyObject {
    func buttonDidTap(_ button: Button)
}
```

这里沿用[按钮委托示例](#闭包与-delegate)中的类型声明。`ButtonDelegate: AnyObject` 把协议限制为只能由类遵循，因此普通结构体和枚举不能遵循 `ButtonDelegate`。这个写法没有为遵循者指定父类。

`weak` 只能用于类实例的引用。`AnyObject` 约束使编译器能够确定 `any ButtonDelegate` 中保存的是类实例引用，因此允许声明 `weak var delegate: (any ButtonDelegate)?`。如果没有其他类类型限制，去掉 `AnyObject` 后，编译器就不能接受这条 `weak` 声明。

## 类型安全、值语义与动态能力

类型安全是值只能按照类型规则允许的方式使用。Swift 编译器检查赋值、操作、参数、返回值和类型约束；无法在编译期确定的部分需要运行时检查。类型匹配不要求类型名称完全相同，例如子类对象可以传给接受父类的参数。

值语义表示 struct、enum 以及 Array、String 等常见值类型在赋值和传参时表现为值的复制。底层可以采用写时复制共享存储，不要求每次赋值都立即深拷贝；值中包含 class 成员时，多份值仍可能引用同一个对象。Swift 同时提供 class 的引用语义和运行时动态能力，并非所有类型都是值类型。

动态能力包括根据实际类型调用类方法、调用协议要求、执行动态类型转换，以及 Apple 平台上的 Objective-C 消息派发。这些能力与编译期类型检查可以同时存在，具体派发机制见[Swift / Objective-C 运行时与方法派发](swift-objc-runtime.md)。

### 值类型与引用类型的选择

引用语义表示类实例的赋值和传参复制对象引用，多个引用可以指向同一个实例。修改这个实例的属性后，其他引用也能读到修改。值类型适合表达需要独立修改的数据；引用类型适合表达需要共享状态或身份的对象。

```swift
struct Draft {
    var title: String
}

var original = Draft(title: "原文")
var editing = original
editing.title = "草稿"
print(original.title) // 原文
print(editing.title)  // 草稿

final class Session {
    var userName = "Alice"
}

let first = Session()
let second = first
second.userName = "Bob"
print(first.userName)     // Bob
print(first === second)   // true
```

`Draft` 的两个变量保存独立的值。`Session` 的两个变量指向同一个实例。`===` 判断两个类实例引用是否指向同一个实例；属性内容相同不表示实例身份相同，具体比较规则见[相等规则与对象身份](#相等规则与对象身份)。

结构体包含类实例属性时，复制结构体只复制该属性保存的引用，不会自动复制内部类实例。相关生命周期和引用关系见[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)。如果需要继承已有类，或者需要遵循只允许类遵循的协议、对接 Objective-C 接口，就必须使用 `class`。选择 `struct` 还是 `class`，看的是数据需要独立复制还是共享身份，与数据大小无关。

### 静态类型、类型推断与数值转换

类型推断是编译器根据初始值和上下文确定变量的类型。类型一旦确定就不能再改变，之后的赋值与运算都要符合这个类型的规则。

```swift
var count = 10       // 推断为 Int
count = 20
// count = "20"     // 编译错误：String 不能赋给 Int

let quantity: Int = 3
let price: Double = 2.5
// let total = quantity * price // 编译错误
let total = Double(quantity) * price
```

Swift 不会在 Int 和 Double 之间自动转换，两者运算前必须显式转换，例如用 `Double(quantity)` 构造一个对应的浮点数。类型推断只决定变量最初的类型，不会把已确定类型的变量转换成另一种数值类型。

### 可选类型、状态表达与并发支持

Optional<T>，简写 T?，表示存在一个 T 类型的值，或者没有值。nil 表示没有值。String? 与 String 是不同类型；普通 String 不能赋值为 nil。Optional 把“可能没有值”写进类型，编译器因此能要求调用方先处理 nil。Objective-C 同样有静态类型检查，也可以用 `nullable`、`nonnull` 标注空值，但编译器对 Objective-C 空值标注的检查比 Swift 宽松。

```swift
let input: String? = "Alice"
if let name = input {
    print(name.count) // name 的类型为 String
}
```

可选绑定的作用域及多个条件的组合方式见[可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)。强制解包（`!`）能通过编译，但值为 nil 时会在运行时触发错误。

枚举用若干命名情况表达合法状态。关联值是某个情况携带的、具有指定类型的数据。下面的 LoadState 在同一时刻只能属于一种情况，可以避免多个独立布尔值和可选属性形成互相矛盾的状态组合。

```swift
enum LoadState {
    case loading
    case success(String)
    case failure(String)
}

let state = LoadState.success("加载结果")
// LoadState.success(123) // 编译错误

switch state {
case .loading:
    print("加载中")
case .success(let content):
    print(content)
case .failure(let message):
    print(message)
}
```

switch 必须覆盖所有可能情况。对于上面的自定义枚举，遗漏情况且没有 default 时，编译器会报告错误。以后新增情况时，没有 default 的相关 switch 也能提示需要补充处理。在 `case .success(let content)` 中，Swift 取出 `.success` 已经携带的字符串，赋给局部常量 `content`。这一步只是取值，不会重新创建结果或解码数据。`content` 这个名称由调用方自己起，改名不影响取值。

Swift 标准库的 `Result<Success, Failure>` 用 `.success(Success)` 和 `.failure(Failure)` 表示成功或失败，其中 `Failure` 必须遵循 `Error` 协议。`Result` 值本身和成功时携带的值是两层：`Result<String, SomeError>` 是结果类型，匹配 `.success(let text)` 取出值后，`text` 的类型才是 `String`。这里的 `SomeError` 表示调用方定义的、遵循 `Error` 的错误类型。网络响应中的应用见 [Alamofire：请求与响应模型](../network-and-storage/network-and-proxy.md#alamofire请求与响应模型)。

async/await 让异步流程按书写顺序表达，不必再嵌套多层回调。await 标记一个可能挂起的位置；await 不会新建线程，也不会把工作移到后台。actor 让外部只能通过 actor 访问它的可变状态，同一时刻最多一个任务修改状态；但在 await 挂起期间，其他任务可能改变状态。细节见[actor 的默认隔离与重入](#actor-的默认隔离与重入)。

### 可选绑定、条件逗号与逻辑与

可选绑定是在可选值有值时取出内部值，并声明一个常量或变量的条件语法。`if let` 声明常量；绑定成功后，后续条件和成功分支可以使用该常量。

```swift
let result: [[Int]] = [[1, 4]]
let interval = [4, 5]

if let last = result.last, interval[0] <= last[1] {
    print(last) // [1, 4]，last 的类型是 [Int]
} else {
    print("至少一个条件不成立")
}
```

本例的每个区间固定包含两个整数，因此 `last[1]` 可以读取第二个整数。`result.last` 的类型是 `[Int]?`，属性的返回规则见[数组的 last 属性](#数组的-last-属性)。程序先检查可选值：有值时，程序声明类型为 `[Int]` 的常量 `last`，再计算 `interval[0] <= last[1]`；没有值时，绑定失败，程序跳过后续条件并进入 `else`。比较结果为 `false` 时也会进入 `else`。`last` 不能在对应的 `else` 中或整个 `if` 语句之后使用。

`guard let` 的失败分支必须退出当前作用域；绑定成功后，`guard` 之后的代码都可以使用取出的值。退出方式取决于所在作用域：函数中用 `return` 或 `throw`，循环中还可以用 `break` 或 `continue`。

`Bool` 表达式是结果为 `true` 或 `false` 的表达式。`&&` 是逻辑与运算符，用两个 `Bool` 表达式组成一个新的 `Bool` 表达式。条件中的逗号用于分隔多个条件，不是生成布尔值的运算符。

| 比较项 | 条件中的逗号 `,` | 逻辑与 `&&` |
| --- | --- | --- |
| 可以连接的内容 | `Bool` 表达式、可选绑定等条件 | 两个 `Bool` 表达式 |
| 检查顺序 | 从左到右，某个条件不成立就停止 | 先计算左侧；左侧为 `false` 时不计算右侧 |
| 是否能声明绑定名称 | 可以，后续条件可以使用前面的绑定名称 | 运算符本身不能声明可选绑定 |
| 是否产生可赋值的布尔结果 | 不能把条件列表直接赋给变量 | 可以，例如 `let passed = x > 0 && x < 10` |

前面的条件已经决定最终结果时，程序跳过后续求值，这种行为称为短路求值。下面两个条件都是 `Bool` 表达式，两种写法的判断结果和求值顺序相同：

```swift
let x = 4

if x > 0, x < 10 {
    print(x)
}

if x > 0 && x < 10 {
    print(x)
}
```

`let last = result.last` 是可选绑定条件，不是返回 `Bool` 的表达式。因此，不能把原例中的逗号直接替换为 `&&`：

```swift
// 错误写法，不能编译：
// if let last = result.last && interval[0] <= last[1] { }
```

原例的两个条件可按相同顺序写成嵌套判断。下面只展示成功分支；原例的 `else` 会在任一条件不成立时执行。

```swift
if let last = result.last {
    if interval[0] <= last[1] {
        print(last)
    }
}
```

与 `||` 混用时，改写需要保留分组。`||` 表示逻辑或，即至少一个布尔条件为真时结果为真。对于 `Bool` 值 `a`、`b`、`c`，`if a || b, c` 对应 `if (a || b) && c`；直接写成 `if a || b && c` 会按 `a || (b && c)` 分组，因为 `&&` 的运算优先级高于 `||`。

### 集合、泛型与协议约束

集合明确元素类型；字典明确键和值的类型。编译器据此阻止不符合类型要求的数据写入。

```swift
var names: [String] = ["Alice"]
names.append("Bob")
// names.append(123) // 编译错误

var scores: [String: Int] = ["Alice": 90]
// scores["Bob"] = "优秀" // 编译错误

func firstElement<T>(of values: [T]) -> T? {
    values.first
}
let firstName = firstElement(of: names) // String?
```

泛型用类型参数编写适用于多种类型的代码，同时保留输入与输出之间的类型关系。上例 T 表示本次调用的元素类型；输入 [String]，返回 String?，不需要先返回 Any 再手动转换。协议定义类型必须满足的能力要求；T: Equatable 要求 T 支持相等比较，让函数内部可以使用 ==。更完整的协议约束与存在类型比较见[协议存在类型与泛型约束](#协议存在类型与泛型约束)。

### 函数签名与回调类型

**函数、方法与函数值。** 函数是可以被调用、执行特定任务的代码单元，可以接收参数并返回结果。方法是作为类型成员声明的函数，包括实例方法和类型方法；在类型扩展中声明的成员函数也属于方法。同样写成 `func speak() { ... }`，声明在类型内部或类型扩展中时是方法，声明在类型之外时是普通函数。

函数值是具有函数类型的可调用值，可以保存到变量或常量、作为参数传递，或作为结果返回。“函数值”指的是被当作数据使用的函数，并不是另一种函数。函数类型描述参数类型、返回类型及相关属性；例如 `() -> Void` 表示不接收参数、不返回有用的结果值，`(Int) -> String` 表示接收一个 `Int` 并返回 `String`。

```swift
func speak() {
    print("Hello")
}

let action: () -> Void = speak // 保存函数值，不执行函数体
speak()                       // 打印 Hello
action()                     // 同样执行 speak，打印 Hello
```

| 写法 | 含义 |
| --- | --- |
| `func speak() { ... }` | 声明并定义函数 |
| 表达式中的 `speak` | 表示函数值，可以保存和传递 |
| `speak()` | 调用函数，执行函数体 |
| `action` | 保存函数值的常量 |

`let action = speak` 不会重新定义函数，也不会执行 `speak()`。函数值与调用结果不同：`speak` 的类型为 `() -> Void`，`speak()` 的结果为 `Void`。声明、定义与函数体的边界见[声明、定义、实现与引用](#声明定义实现与引用)。

**方法引用与方法调用。** 以下非 `mutating` 实例方法可以绑定到具体的 `person`，形成可保存和调用的函数值：

```swift
struct Person {
    func speak() {
        print("Hello")
    }
}

let person = Person()
let action: () -> Void = person.speak // 绑定 person，不调用 speak()
action()                             // 调用 person 的 speak()，打印 Hello
```

`Person` 中的 `speak()` 是实例方法，`person.speak` 表达式得到绑定接收者的函数值，`person.speak()` 才执行方法。方法值保留方法原有的类型和隔离要求。例如，方法受 `MainActor` 隔离时，从隔离范围外调用这个方法值仍然需要 `await`。

**闭包与闭包表达式。** Swift 将全局函数、嵌套函数和闭包表达式视为闭包的不同形式。全局函数定义在类型和其他函数之外，有名称且不捕获局部值；嵌套函数定义在另一个函数内部，有名称并可以捕获外层值。闭包表达式用 `{ ... }` 创建可调用值，没有通过 `func` 声明函数名称。捕获表示闭包保留对定义位置外部变量或值的访问，具体规则见[默认捕获与捕获列表](#默认捕获与捕获列表)。

```swift
let action: () -> Void = {
    print("Hello")
}

action() // 打印 Hello
```

`action` 是保存闭包的常量名称，`{ print("Hello") }` 是闭包表达式，产生的函数值具有 `() -> Void` 类型。闭包表达式没有捕获外部值时，也仍然是闭包表达式。前面的 `person.speak` 作为值使用时保存了接收者和要调用的方法，通常也称为闭包。这几个名称描述的层次不同：函数声明、方法声明和闭包表达式是源码中的写法；“函数值”是运行时可以保存、传递和调用的值。同一个值可以既是函数值，又是闭包。

**参数和返回类型检查。** 函数声明中的参数类型和返回类型约束调用方与实现方。编译器检查传入参数，也检查函数返回的值。

```swift
func double(_ value: Int) -> Int {
    value * 2
}
let result = double(3)
// double("3") // 编译错误
```

函数作为值传递时，类型检查仍然有效。要求 `(Int) -> String` 时，传入的函数或闭包不能返回不兼容的值，调用方也不能传入错误类型。闭包参数的推断与最小示例见[闭包参数、捕获列表与可选回调](#闭包参数捕获列表与可选回调)。

**返回函数值。** 下面的 `makeReader` 返回一个不接收参数、返回 `Int` 的函数值：

```swift
func makeReader(value: Int) -> () -> Int {
    func read() -> Int {
        value
    }
    return read
}

let reader = makeReader(value: 10)
let number = reader() // 10
```

`return read` 把内部函数作为值返回；`return read()` 会先执行内部函数并得到 `Int`，不符合 `makeReader` 要求的返回类型。`read` 的函数体只有一个表达式，因此可以省略 `return`。内部函数捕获 `value`，`makeReader` 返回后，返回的函数值仍能使用 `value`。

将内部函数改写成闭包表达式时，`makeReader` 的函数体可以写为 `return { value }`，返回类型仍是 `() -> Int`。`@Sendable () -> Int` 在函数类型上增加并发传递要求；具名函数和闭包表达式都必须满足该要求，具体见[Sendable 与 @Sendable](#sendable-与-sendable)。

### 运行时类型转换与安全边界

Any 可以保存任意类型的值，但不能直接通过 Any 使用某个具体类型特有的成员。需要具体能力时，应确认实际类型。

```swift
let value: Any = "Hello"
if let text = value as? String {
    print(text.count)
}
```

is 检查值能否作为指定类型使用并返回 Bool；as? 尝试转换，失败时返回 nil；as! 强制转换，失败时触发运行时错误。值保存为 Any 后，编译器只知道“这是某个值”，不知道具体类型，因此要先用 `as?` 等转换取回具体类型才能使用其成员；转换本身仍受类型检查。

类型安全只保证值按类型规则使用。数组越界、强制解包失败、强制转换失败和计算逻辑错误仍然会发生。例如年龄为 -10 符合 Int 的类型要求，却不符合“年龄不能为负数”的业务要求，这类条件需要程序自己检查。使用不安全指针时，开发者还要自己保证指针指向有效、已初始化的内存。

### actor 的默认隔离与重入

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

### MainActor、主队列与异步调用

MainActor 是 Swift 内置的全局 actor。全局 actor 是由程序各处共同使用的 actor，其隔离规则可以应用于不同类型中的方法和属性。@MainActor 声明代码和数据归 MainActor 管理；actor 隔离的基本含义见[actor 的默认隔离与重入](#actor-的默认隔离与重入)。

在 iOS、macOS 应用中，MainActor 使用与主队列关联的串行执行器，安排受隔离代码在主线程执行。所有归属于 MainActor 的方法和属性共享这一隔离范围，不会为每个标有 @MainActor 的类型或实例分别创建执行器。与普通 actor 一样，MainActor 上的异步方法挂起期间，其他调用也可以进入。

三者是不同的概念：主线程是实际执行代码的线程；`DispatchQueue.main` 是 GCD 提供的串行队列，提交到主队列的任务在主线程执行；MainActor 是 Swift 的隔离规则，编译器据此检查访问是否合法，运行时把受隔离代码安排到主线程执行。

#### 跨隔离调用与同步调用

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

#### 任务挂起与线程释放

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

#### 单线程并发与挂起恢复的触发者

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

#### 同步函数发起主 actor 任务

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

`[count]` 在创建闭包时复制当时的整数值，闭包不再与外部共享 `count` 变量，因此外部之后把 `count` 改成 10，`read()` 仍返回 0。捕获类实例时，`[object]` 只复制对象引用，不会复制对象，对象的类型仍需符合 `Sendable`。捕获列表的基础机制见[默认捕获与捕获列表](#默认捕获与捕获列表)。

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

`read` 捕获的 `value` 是不可变的 `Int`，符合传递要求。`return read` 返回函数值，不执行函数体；函数类型与调用写法见[函数签名与回调类型](#函数签名与回调类型)。闭包表达式写法 `return { value }` 也可以由返回类型推断为 `@Sendable`。

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

Task 表示 Swift 管理的一段异步工作。Task 是结构体类型；创建出的值称为 Task 实例（结构体的值也称为实例）。Task 实例代表一个已经创建的任务，可以用来等待结果、查询状态和发出取消请求。类型与实例的基本区别见[类型、实例与对象引用](#类型实例与对象引用)。

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

`Void` 是空元组 `()` 的类型，表示没有需要交给调用方的结果数据，具体定义见[元组、字典与空元组 Void](#元组字典与空元组-void)。`Never` 是没有任何值的类型；用于 `Failure` 时，表示任务闭包不会向外抛出错误。`Task<Void, Never>` 因此表示“不返回有意义的数据，也不向外抛出错误”的任务。

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

普通局部变量与任务局部值不同，需要通过闭包捕获才能在任务中使用。Task 和 Task.detached 都可以捕获普通值，但捕获的值必须满足并发安全要求；捕获共享的可变变量时，Swift 不会自动添加同步保护。闭包捕获的具体规则见[默认捕获与捕获列表](#默认捕获与捕获列表)。

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

## 类型、实例与对象引用

Person 是一个具体的类型名；class 是声明类类型的关键字，struct、enum 是声明其他类型的关键字。“引用类型”是对类型的分类，class 声明的类型都是引用类型；“引用”则是运行时指向某个对象的值。

```swift
class Person {}
let person = Person()
```

Person 是类型，Person() 创建 Person 类的一个实例，person 常量保存指向这个实例的引用。struct 和 enum 同样有实例，例如 `Point(x: 1, y: 2)` 是 Point 结构体的实例，`Direction.left` 是 Direction 枚举的实例。

Swift 语境中的“对象”通常指 class 实例；“实例”也包括 struct 和 enum 的具体值。

## 引用赋值、浅拷贝与深拷贝

引用赋值复制对象引用，不创建对象副本。浅拷贝创建新的外层对象，但继续共享嵌套对象。深拷贝除了创建新的外层对象，还为需要独立修改的嵌套对象创建副本，修改副本不会影响原对象。判断是哪种复制，要检查嵌套的可修改对象是否仍被共享，只看最外层对象是不是新地址不够。

下面的 Address 和 Person 是独立示例使用的模型。Codable 表示模型同时支持编码和解码，具体机制见后文“Codable 与序列化”。

```swift
import Foundation

final class Address: Codable {
    var city: String

    init(city: String) {
        self.city = city
    }
}

final class Person: Codable {
    var name: String
    var address: Address

    init(name: String, address: Address) {
        self.name = name
        self.address = address
    }
}
```

以下三段代码依次执行。=== 判断两个引用是否指向同一个类实例，规则见[相等规则与对象身份](#相等规则与对象身份)。

```swift
let original = Person(name: "小明", address: Address(city: "北京"))
let other = original

print(original === other) // true
other.name = "小红"
print(original.name)      // 小红
```

other 和 original 指向同一个 Person。上面的赋值只复制引用，没有创建浅拷贝对象。

```swift
let shallow = Person(name: original.name, address: original.address)

print(original === shallow)                 // false
print(original.address === shallow.address) // true

shallow.name = "小李"
print(original.name) // 小红

shallow.address.city = "上海"
print(original.address.city) // 上海
```

shallow 是新的 Person，姓名可以独立修改。但初始化时传入的是原 Address 的引用，所以两个 Person 仍共享同一个地址对象。

```swift
let deep = Person(
    name: original.name,
    address: Address(city: original.address.city)
)

print(original === deep)                 // false
print(original.address === deep.address) // false

deep.address.city = "广州"
print(original.address.city) // 上海
```

deep 同时拥有新的 Person 和新的 Address。对于这个模型，姓名和城市都能独立修改。如果 Address 内部还引用了需要独立修改的对象，也要继续复制这些对象。

| 操作 | Person 是否为同一个实例 | Address 是否为同一个实例 |
| --- | --- | --- |
| 引用赋值 | 是 | 是 |
| 上例的浅拷贝 | 否 | 是 |
| 上例的深拷贝 | 否 | 否 |

复制 struct 不一定是深拷贝：struct 内部的类引用只会复制引用。class 赋值也不是浅拷贝：赋值只复制引用，不创建任何新对象。结构体内部的类引用如何复制，见[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)；数组中的类实例见[数组值语义、写时复制与返回值](#数组值语义写时复制与返回值)。

### Swift 中的深拷贝实现方法

手动复制通过初始化方法创建新对象，并逐层复制需要独立的嵌套对象。可以把这个过程封装为 deepCopy() 方法。deepCopy 只是普通方法名，能否得到深拷贝取决于方法里的实现。

```swift
extension Person {
    func deepCopy() -> Person {
        Person(name: name, address: Address(city: address.city))
    }
}
```

NSCopying 是 Foundation 提供的复制协议，要求类型实现 copy(with:)。协议只规定复制方法的接口，不会自动复制属性。若 copy(with:) 复用了原 address，结果仍然是浅拷贝；要获得深拷贝，需要在实现中创建新的嵌套对象。

编码后再解码可以重建模型，例如使用 JSONEncoder 和 JSONDecoder，或使用 PropertyListEncoder 和 PropertyListDecoder。副本是否完整、是否独立，取决于编码时保存了哪些属性，以及解码时是否创建了新对象，详见[序列化复制的边界](#序列化复制的边界)。

NSKeyedArchiver 和 NSKeyedUnarchiver 是 Foundation 的对象归档与解档工具。归档是按编码协议保存对象及相关状态，解档是读取归档数据重建对象。参与归档的对象通常遵循 NSCoding；使用安全编码时还需满足 NSSecureCoding。归档会记录对象之间的引用关系；解档得到的副本是否满足深拷贝要求，取决于归档时保存了哪些属性、解档时如何重建对象。

递归复制时，如果多个属性指向同一个对象，或者对象之间互相指向，复制代码需要用一张表记录“原对象 → 副本”。再次遇到同一个原对象时，直接使用表中的副本。这样副本之间也保持原来的共享关系，复制也不会因循环指向而无限递归。

## Codable 与序列化

序列化把模型中需要保存的状态转换成可存储、可传输的数据。反序列化读取这些数据，并据此构建模型。JSON 是表达数据的文本格式；Foundation 的 Data 用于保存字节数据。JSONEncoder 将模型编码为包含 JSON 的 Data，JSONDecoder 从这份 Data 解码出指定类型的模型。

| 协议 | 表达的能力 |
| --- | --- |
| Encodable | 类型能通过 encode(to:) 把状态交给编码器 |
| Decodable | 类型能通过 init(from:) 从解码器读取数据并初始化实例 |
| Codable | 同时遵循 Encodable 和 Decodable |

Codable 是类型别名，等价于协议组合 Encodable & Decodable。只需要反序列化时，遵循 Decodable 即可。如果类型的所有存储属性都遵循 Codable，编译器会自动生成 encode(to:) 和 init(from:)；有属性不遵循 Codable，或需要自定义字段规则时，要自己实现这两个方法。Codable 只是协议，真正执行编码的是 JSONEncoder 等编码器。

下面复用前文的 Person 和 Address。两个类的所有存储属性都参与编码和解码，使用编译器自动生成的实现。

```swift
let source = Person(name: "小明", address: Address(city: "北京"))

do {
    let data = try JSONEncoder().encode(source)
    let restored = try JSONDecoder().decode(Person.self, from: data)

    print(source === restored)                 // false
    print(source.address === restored.address) // false

    restored.address.city = "上海"
    print(source.address.city) // 北京
} catch {
    print("编码或解码失败：\(error)")
}
```

Person.self 把 Person 这个类型传给 decode，指定需要构建的模型类型。try 表示调用可能抛出错误，do-catch 在示例中捕获编码或解码错误。JSONDecoder 按目标类型解码。外部数据的内容在编译时无法检查，字段类型不匹配或缺少必需字段时，decode 会在运行时抛错。

编码结果相当于下面的 JSON。JSON 保存姓名和城市，不保存原 Person 或 Address 的内存地址。

```json
{"name":"小明","address":{"city":"北京"}}
```

解码时，自动生成的实现创建新的 Person，并为 address 创建新的 Address，所以副本不再共享原地址，可以独立修改。副本独立的原因是解码实现创建了新对象；如果自定义解码代码复用已有对象，副本仍会共享这个对象。

### CodingKeys 的作用与枚举语法

CodingKeys 指定哪些属性参与自动生成的编码和解码，以及每个属性在数据中对应的字段名称。字段名称是 JSON 中用于标识某个值的名称，例如 {"name":"小明"} 中的 name。下面的 SelectionModel 是一个独立模型。

```swift
struct SelectionModel: Codable {
    var name: String
    var isSelected = false

    enum CodingKeys: String, CodingKey {
        case name
    }
}
```

这段代码声明了一个名为 CodingKeys 的枚举类型。嵌套类型是声明在另一个类型内部的类型，CodingKeys 是 SelectionModel 的嵌套类型。嵌套类型不是存储属性，SelectionModel 实例中不会多保存一个 CodingKeys 值。

| 写法 | 含义 |
| --- | --- |
| enum CodingKeys | 声明名为 CodingKeys 的枚举类型 |
| String | 指定枚举原始值的类型为字符串；原始值是枚举成员对应的固定值 |
| CodingKey | 遵循 CodingKey 协议，使成员可作为编码和解码时的字段标识 |
| case name | 声明一个名为 name 的枚举成员 |

冒号后同时写了原始值类型 String 和遵循的协议 CodingKey；String 在这里不是父类。对于这个字符串枚举，case name 未显式指定原始值时，Swift 使用成员名称作为原始值，因此相当于 case name = "name"。

```swift
let key: SelectionModel.CodingKeys = .name
print(key.rawValue)    // name
print(key.stringValue) // name
// let invalid: SelectionModel.CodingKeys = .nmae // 编译错误：没有这个枚举成员
```

rawValue 来自原始值枚举。stringValue 是 CodingKey 协议要求的属性，表示字段名称；对于原始值是 String 的枚举，编译器自动让 stringValue 返回 rawValue。从类型外访问嵌套枚举时，使用 SelectionModel.CodingKeys；在 SelectionModel 内部的方法中，可以直接使用 CodingKeys。

枚举适合表示一组明确的字段标识，每个 case 对应一个标识，拼错成员名能在编译期被发现。编译器按约定在模型内部查找名为 CodingKeys 的类型，用其中的成员生成编码和解码代码；这一步发生在编译时，名字不是 CodingKeys 的枚举不会被使用。手动实现编码和解码时，可以使用任意遵循 CodingKey 的类型，不必是枚举。

此例只列出了 name，所以自动生成的实现只转换 name。若加入 case isSelected，两个属性都会参与转换。对于这个简单模型，也可以完全省略 CodingKeys，让编译器自动为两个属性生成对应的键。

## 序列化复制的边界

用“编码再解码”做深拷贝，要分别检查三件事：副本是否保留了全部状态，新旧对象是否独立，副本内部的对象共享关系是否与原对象一致。以下示例使用 JSONEncoder、JSONDecoder 和编译器自动生成的实现；自定义实现会单独说明。

### 未编码的属性不会保留原值

继续使用 SelectionModel。isSelected 没有列在 CodingKeys 中，因此 JSON 不包含该属性，解码得到的实例使用声明时的默认值 false。

```swift
func copySelection() throws {
    var original = SelectionModel(name: "小明")
    original.isSelected = true

    let data = try JSONEncoder().encode(original)
    let copied = try JSONDecoder().decode(SelectionModel.self, from: data)

    print(original.isSelected) // true
    print(copied.isSelected)   // false
}
```

isSelected 能从 CodingKeys 中省略，是因为它有默认值；没有默认值的属性被省略时，编译器无法自动生成解码实现。这个例子也说明，编码和解码都成功，副本仍可能丢失部分状态。

### 自定义解码可能重新使用共享对象

自定义解码是自己实现 init(from:)，决定怎样读取数据并初始化属性。假设程序提供 AddressStore.sharedAddress，表示一个已经存在的共享 Address。若在自定义 init(from:) 中执行下面的代码，就会复用已有对象，而不是创建新地址。

```swift
// 自定义 init(from:) 中的示意语句。
// 前提：AddressStore.sharedAddress 已经定义为共享 Address。
self.address = AddressStore.sharedAddress
```

如果原 Person 也引用 AddressStore.sharedAddress，新旧 Person 就会共享地址。修改新 Person 的 address.city，原 Person 读取到的城市也会改变。共享来自自定义代码的选择，JSONDecoder 默认不会复用已有对象。

### 原来共享的对象可能被拆成两个对象

下面的 AddressPair 使用前文的 Address。home 和 mailing 原来指向同一个实例。=== 比较的是实例身份，不是城市内容是否相同。

```swift
struct AddressPair: Codable {
    var home: Address
    var mailing: Address
}

func copyAddresses() throws {
    let address = Address(city: "北京")
    let original = AddressPair(home: address, mailing: address)
    print(original.home === original.mailing) // true

    let data = try JSONEncoder().encode(original)
    let copied = try JSONDecoder().decode(AddressPair.self, from: data)
    print(copied.home === copied.mailing) // false

    original.home.city = "上海"
    print(original.mailing.city) // 上海

    copied.home.city = "广州"
    print(copied.mailing.city)   // 北京
}
```

JSON 相当于 {"home":{"city":"北京"},"mailing":{"city":"北京"}}。JSON 保存了两份城市内容，但没有记录 home 和 mailing 原来指向同一个对象，所以自动解码分别创建了两个 Address。结果是：副本与原模型相互独立，但副本内部的共享关系与原模型不同。

### 循环引用需要显式记录对象关系

这里的循环引用指沿着对象属性访问其他对象后，又回到先前的对象。例如 A.friend 指向 B，B.friend 又指向 A。编码代码如果不断展开 friend，就会重复执行“编码 A → 编码 B → 编码 A”，无法得到有限的嵌套结果。这里只关心属性之间的指向关系，与这些引用是强引用还是 weak 无关。

普通嵌套 JSON 不能直接表达这种循环。自动生成的 Codable 实现也不会自动把重复对象替换成编号。可以自行定义 id 和 friendID，分别表示对象编号和朋友的对象编号。

```json
[
    {"id": 1, "friendID": 2},
    {"id": 2, "friendID": 1}
]
```

解码时先创建编号为 1 和 2 的对象，再按 friendID 查找对应对象并建立引用。保存编号、查找对象和恢复关系都需要自己实现。如果某个反向关系不需要保留，也可以不编码这个属性，但这样副本就不包含这层关系。

## 符号引用与对象引用

**在代码搜索语境中，符号（symbol）是编译器或 IDE 用来表示代码实体的标识。** 类型、变量、属性、方法和类型别名都可以对应符号。符号除了名称，还记录所属作用域、类型、声明位置等信息。两个类分别声明同名的 `actionHandler`，对应两个不同的属性符号。

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

值类型变量同样可以被源码引用。例如 `let number = 10; print(number)` 中，`print(number)` 使用了变量符号 `number`，但 `Int` 仍是值类型。符号引用只是源码中的使用位置，与运行时的 ARC 强引用、弱引用无关。对象引用与指针的区别另见[引用与指针](swift-objc-runtime.md#对象引用与-unsafepointer)。

## 声明、定义、实现与引用

声明、定义和实现是看同一段代码的三个角度，不是按大括号内外划分的三个位置。在 Swift 语法中，函数和类型都通过声明引入；带函数体的完整函数声明通常也称为函数定义；“实现”指其中完成具体行为的代码，也就是函数体。

| 概念 | 含义 |
| --- | --- |
| 声明 | 引入类型、函数等名称，并说明相关信息；函数声明包含名称、参数和返回类型等信息 |
| 定义 | 给出类型或函数的完整内容；函数定义包含名称、参数、返回类型和函数体，不是仅指大括号部分 |
| 实现 | 完成具体行为的执行代码；函数体承载函数的实现逻辑 |
| 符号引用 | 源码中使用已声明符号的位置，包括调用函数或把函数保存为函数值 |

**协议声明与方法要求。** `protocol Speaker { ... }` 整体是一条协议声明，也可以说定义了 `Speaker` 协议；`protocol Speaker` 只是声明的开头，`{ ... }` 是协议体。协议体中的 `func speak()` 声明方法要求，没有提供执行代码。

```swift
protocol Speaker {
    func speak() // 协议要求，没有函数体
}

extension Speaker {
    func speak() {
        print("Hello")
    }
}
```

扩展中的完整 `func speak() { ... }` 是方法声明，也提供了方法定义和默认实现。`{ print("Hello") }` 是函数体，其中的 `print("Hello")` 是具体实现逻辑。所以一条声明可以同时包含实现；只写 `func speak()` 而没有函数体，才是不含实现的声明。

协议体中的 `func speak()` 只是要求，协议本身不提供实现。扩展中的 `speak()` 是否属于协议要求，要看协议体中有没有声明这个方法：协议体中声明了，扩展中的方法就是这项要求的默认实现；没有声明，扩展中的方法就只是普通扩展方法。分发规则见[扩展方法与协议要求的派发](swift-objc-runtime.md#扩展方法与协议要求的派发)。

**C++ 的声明与定义分离。** C++ 允许先声明普通函数，再提供函数定义。常见组织方式是头文件保存声明、源文件保存定义，但语言不要求每个函数都拆开编写。

```cpp
// Speaker.h：声明函数，不提供实现
void speak();

// Speaker.cpp：包含声明，再提供完整定义
#include "Speaker.h"
#include <iostream>

void speak() {
    std::cout << "Hello\n";
}
```

`void speak();` 是只有声明、没有定义的函数声明；完整的 `void speak() { ... }` 是函数定义，同时也是声明；大括号包围的部分是函数体。如果调用位置之前已经能看到完整定义，就不需要再写一次单独的声明。“声明与定义分离”中的“声明”，指的就是 `void speak();` 这种不带函数体的声明。

**Objective-C 的声明与定义分离。** Objective-C 的头文件与实现文件也可以分别包含同一个方法的声明和定义：

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

在 Xcode 中搜索 `reload` 的声明和定义，索引完整时会得到两条结果，分别在 Demo.h 和 Demo.m；如果搜索范围还包含引用，调用位置也会出现在结果中。这个例子是**同一个方法符号对应两个源码位置**。协议中的方法要求和遵循类型中的实现则是两个不同的声明，不属于这种情况。

这与[属性、getter 和 setter](../engineering/xcode-and-testing.md#属性访问器与索引记录)的例子不同：属性与两个访问器是三个不同符号，可以关联到同一源码位置。**搜索结果条数不等于不同符号的数量。**

## Swift 与 C 接口互操作

C 接口是指库公开的函数、类型和调用约定采用 C 兼容的形式。例如 CFRunLoopSourceSignal 是一个 C 风格函数，Swift 导入 CoreFoundation 框架后就可以直接调用。接口形式说明调用者怎样使用库，实现语言说明库内部用什么语言编写，两者互不决定。

Core Foundation 最初用 C 实现，但接口是 C 风格，并不能说明当前系统版本的内部实现只用 C。使用系统框架时，应依据公开接口文档描述的行为，不依赖未公开的内部实现。

调用示例见[iOS 线程与任务调度中的 Source0 示例](../ios/threads-and-scheduling.md#source0-示例后台压缩完成后刷新预览)。

## 存储属性、计算属性与访问器

getter 负责读取属性，setter 负责写入属性。`let x = person.age` 是读取，`person.age = 20` 是写入；`newValue` 是 setter 接收的新值的默认参数名。

```swift
class Person {
    var age = 18
}
```

age 是存储属性，值保存在实例内部。读写 age 时，可以理解为 getter 读取 age 的存储，setter 把新值写入存储。下面的代码只用来说明这层含义，不是编译器生成的源码，实例中也只有一个 age 字段：

```swift
// 以下成员位于 Person 类内部
private var ageStorage = 18

var age: Int {
    get { ageStorage }
    set { ageStorage = newValue }
}
```

计算属性不在实例中占用存储，读写时运行 get／set 中的代码，这些代码可以读写其他存储属性。编译器优化后，简单的属性访问可能直接变成一次内存读写，机器码中不一定有独立的 getter／setter 函数调用。

Xcode 索引可以将实例属性的隐式 getter／setter 分类为 `InstanceMethod`，子类别分别为 `AccessorGetter`／`AccessorSetter`，因此显示 M 图标；属性本身显示 P。声明处与赋值处的具体记录见[P／M 图标与源码位置](../engineering/xcode-and-testing.md#属性访问器与索引记录)。索引记录的是源码层面的符号，与优化后的机器码中是否保留 getter／setter 调用无关。

### 实例存储属性与局部变量的初始化

初始化是第一次为属性或变量设置值。实例存储属性由每个实例分别保存；局部变量是在函数或方法内部声明、在对应代码范围内使用的变量。声明时没有写初值的属性和变量，仍然必须在使用前初始化。

非 `lazy` 的实例存储属性必须在实例成功初始化完成前获得初始值。初值可以写在属性声明处，也可以由 `init` 设置。`init` 是创建实例时执行初始化工作的构造器。非可选的 `Int` 属性不会自动获得 `0`，必须显式赋值。

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

这里的 `age` 在声明处没有初值，`Person(age: 18)` 在 `init` 中完成赋值。初始化必须在 `init` 结束前完成，实例创建之后再给属性赋值，属于普通修改，不能替代初始化。

局部变量可以先声明、后赋值，但首次读取前必须已经初始化。编译器必须能够确认：所有能执行到这次读取的路径，都已经给变量赋值。局部变量如果始终没有被读取，可以一直不赋值。

```swift
func showAge() {
    var age: Int          // 允许先声明

    // print(age)         // 编译错误：变量尚未初始化

    age = 18              // 第一次赋值，完成初始化
    print(age)            // 允许读取
}
```

可选存储属性的值为 `nil`，与“尚未初始化”是两种状态。`var nickname: String?` 没有写初值时，Swift 自动用 `nil` 初始化；`nil` 表示当前没有 `String` 值，但属性已经初始化。`let nickname: String?` 不会自动初始化为 `nil`，必须在声明处或 `init` 中赋值。

`lazy` 存储属性是例外：它不需要在实例初始化完成前获得值。声明中的初始化表达式在第一次读取属性时才执行，结果保存下来供之后读取；如果第一次读取前已经给属性赋值，就使用赋的值，初始化表达式不再执行。

### 指定初始化方法与便利初始化方法

初始化方法是通过 `init` 声明、用于建立实例初始状态的入口。对于 Swift 类，初始化方法分为指定初始化方法和便利初始化方法。

指定初始化方法负责初始化本类引入的存储属性。类具有父类时，指定初始化方法再调用直接父类的指定初始化方法。没有 `convenience` 修饰的 `init` 都是指定初始化方法，是否带 `required`、`override` 或 `?` 不影响分类。

一个类可以有多个指定初始化方法。每个能成功返回实例的初始化方法，都必须初始化全部存储属性，并完成父类初始化。

```swift
class Item {
    let title: String

    init(title: String) {
        self.title = title
    }

    init(number: Int) {
        self.title = String(number)
    }

    convenience init() {
        self.init(title: "未命名")
    }
}
```

`init(title:)` 和 `init(number:)` 是两个指定初始化方法。`convenience init()` 是便利初始化方法；便利初始化方法通过 `self.init(...)` 调用同一个类的其他初始化方法，最终必须到达指定初始化方法。便利初始化方法不能直接用 `super.init(...)` 完成父类初始化。

### required、override 与初始化方法继承

`required` 表示所有子类都必须提供这个初始化方法。子类可以通过自动继承满足要求，也可以自己实现，不必每个子类都重写一遍。

`override` 表示子类为父类已有的指定初始化方法提供新实现。子类重写父类的 `required` 初始化方法时，只写 `required`，不写 `override`。子类新写一个参数不同的指定初始化方法，属于新方法，不是重写。

Swift 类的初始化方法不会像普通实例方法那样无条件继承。以下自动继承规则以“子类为新增的全部存储属性提供默认值”为前提：

1. 子类没有定义任何指定初始化方法时，自动继承父类全部指定初始化方法。
2. 子类通过继承或自行实现，具备父类全部指定初始化方法时，自动继承父类全部便利初始化方法。

在这两条规则下，子类仍然可以自己添加便利初始化方法。规则中的“默认值”指属性声明处写的初值，在其他方法里赋值不算。

```swift
class Parent {
    let name: String

    required init(name: String) {
        self.name = name
    }
}

class Child: Parent {
    var count = 0
    // 没有定义指定初始化方法，自动继承 init(name:)。
}

let child = Child(name: "示例")
```

这里 `Child` 不需要重写 `required init(name:)`。如果 `Child` 定义了自己的指定初始化方法，就不再满足第一条自动继承规则；`Child` 必须自行提供父类要求的 `init(name:)`。

### UIViewController 的 init?(coder:) 与纯代码初始化

`UIViewController` 已经声明 `required init?(coder: NSCoder)`。该方法是指定初始化方法，也是可失败初始化方法；`init?` 表示初始化可以通过 `return nil` 失败，调用方得到可选实例。

`NSCoder` 是 Foundation 提供的编码和解码基类。这里的 `coder` 提供从归档数据恢复对象所需的信息。UIKit 从 Storyboard 或 nib 还原控制器时，会使用对应的解码初始化路径。

子类定义了自己的指定初始化方法后，就不再自动继承父类的指定初始化方法，其中包括 `init?(coder:)`。而 `init?(coder:)` 是 `required`，所以子类必须自己实现它。

```swift
import UIKit

final class DetailViewController: UIViewController {
    let titleText: String

    init(titleText: String) {
        self.titleText = titleText
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("此控制器仅支持 init(titleText:)")
    }
}
```

`init(titleText:)` 先设置子类属性，再通过 `super.init(nibName:bundle:)` 初始化父类部分；该方法参数与父类方法不同，因此不写 `override`。初始化顺序见[类的两阶段初始化与 self 使用时机](#类的两阶段初始化与-self-使用时机)。

示例明确约定只通过代码创建控制器。`required init?(coder:)` 保留父类要求的入口，但调用 `fatalError` 拒绝这条路径。`fatalError` 报告不可恢复的错误并直接终止程序，返回类型是 `Never`，不会返回，所以这条路径不需要初始化 `titleText`。`fatalError` 既不返回 `nil`，也不能被 `do-catch` 捕获。

如果控制器需要支持 Storyboard 或 nib，就要在 `init?(coder:)` 中初始化子类属性，并调用 `super.init(coder: coder)`。程序只通过代码创建控制器时，`init?(coder:)` 不会被调用，其中的 `fatalError` 也不会执行。

### 类的两阶段初始化与 self 使用时机

Swift 类的两阶段初始化，是先让继承链中各层引入的存储属性取得初始值，再执行进一步的定制操作。初始化入口的分类见[指定初始化方法与便利初始化方法](#指定初始化方法与便利初始化方法)。这里讨论普通、非 lazy 的存储属性。

子类的指定初始化方法必须先初始化子类自己引入的存储属性，再调用 super.init；父类引入的属性由父类的指定初始化方法初始化。顺序不能颠倒，子类属性的第一次赋值必须在 super.init 之前。

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

第一阶段完成前，初始化方法不能调用实例方法、读取实例属性，或把 self 作为一个完整实例传给其他代码。`self.age = age` 是第一阶段允许的写入，只用于初始化属性。子类属性初始化完成、super.init 返回后，第二阶段开始，初始化方法才可以调用实例方法和传递 self。

这个顺序保证：任何实例方法开始执行前，实例的所有存储属性都已经有值。对于继承得到且允许修改的属性，子类应在 super.init 之后再定制其值，避免先写入的值被父类初始化覆盖。

## 扩展中的类型属性与实例存储限制

对于普通的非泛型类型，extension 可以添加 static 存储属性，也可以添加实例计算属性和类型计算属性。static 存储属性属于类型本身，整个程序只有一份，不需要每个实例新增字段，所以不违反“扩展不能添加实例存储属性”的限制。泛型类型不支持 static 存储属性，这条规则不适用于泛型类型。

```swift
struct Person {
    var age: Int
}

extension Person {
    static let adultAge = 18
    var isAdult: Bool { age >= Self.adultAge }
}
```

Person 本来的 age 是实例存储；adultAge 是类型属性；isAdult 是计算属性。这条限制只针对 extension：struct 在原始定义中照样可以声明实例存储属性，enum 也可以通过关联值携带数据。

### 为什么扩展不能添加实例存储属性

Swift 的 `extension` 可以添加实例计算属性，但不能添加实例存储属性。实例计算属性在访问时执行代码，不为每个实例新增一个保存结果的字段；实例存储属性需要每个实例分别保存对应的值。

新增实例存储属性会影响实例的内存布局和初始化规则。内存布局是实例中各项数据的排列方式和占用空间。新增字段需要纳入这个布局；新增的普通存储属性还必须在实例成功初始化完成前获得初始值。

扩展可以写在其他代码库中，也可以扩展拿不到源代码的类型。原类型已经编译好，实例的布局和初始化过程已经确定，另一个代码库中的扩展无法再改变这些规则，所以 Swift 禁止扩展添加实例存储属性。这是语言层面的统一规则，即使扩展与原类型写在同一个文件中，也不能添加实例存储属性。

上面的 `Person` 示例没有手写 `init`，Swift 会为这个结构体自动生成接收 `age` 的构造器，因此可以通过 `Person(age: 18)` 创建已初始化的实例。扩展中的 `isAdult` 读取已有的 `age` 并计算结果；`adultAge` 是整个类型共享的一份存储。这两项都不需要给每个 `Person` 实例增加存储字段。

## 协议存在类型与泛型约束

协议规定类型必须满足的要求。泛型用类型参数编写能处理多种类型的代码，协议可以作为类型参数的约束，所以两者经常一起使用。

本节使用的 `Speaker` 协议，以及 `Dog`、`Cat` 类型，定义见[协议见证表示例](swift-objc-runtime.md#协议见证表与遵循关系)。`Speaker` 要求实现 `speak()`，`Dog` 和 `Cat` 是两个独立的结构体类型。

### 泛型保留类型关系，any 隐藏具体静态类型

`any Speaker` 是协议存在类型：类型为 `any Speaker` 的变量可以保存任意遵循 `Speaker` 的值，但通过这个变量只能使用 `Speaker` 协议声明的成员。这种做法称为类型擦除：编译时不再知道具体类型，但运行时值仍然保留实际类型。

```swift
func makeGenericSound<T: Speaker>(_ value: T) -> T {
    value.speak()
    return value
}

func makeSound(_ value: any Speaker) -> any Speaker {
    value.speak()
    return value
}

let dog: Dog = makeGenericSound(Dog()) // 输入 Dog，返回类型仍是 Dog
let result = makeSound(Dog())         // 静态类型为 any Speaker
// let otherDog: Dog = result         // 编译错误：不能直接当作 Dog
result.speak()                       // 仍然执行 Dog 的实现
```

泛型签名中的输入和输出使用同一个 `T`，因此调用方知道返回值与输入值具有相同类型。`makeSound` 的签名只承诺返回某个遵循 `Speaker` 的值，没有承诺与输入是同一个具体类型。运行时仍保存实际类型信息，需要时可以通过类型转换检查实际类型。

```swift
func sameType<T: Speaker>(_ first: T, _ second: T) {}
func possiblyDifferent(_ first: any Speaker, _ second: any Speaker) {}

sameType(Dog(), Dog())
// sameType(Dog(), Cat()) // 编译错误：两个独立结构体不能统一为同一个 T
possiblyDifferent(Dog(), Cat()) // 允许
```

`sameType` 要求两个参数的类型能确定为同一个 `T`。如果两个参数是不同的类，编译器可以把 `T` 确定为它们的共同父类；上例中的 `Dog`、`Cat` 是两个独立的结构体，没有共同类型，所以调用失败。另外，泛型函数体只能使用约束声明的成员：即使某次调用传入 `Dog`，函数体也不能访问 `Speaker` 没有要求的 `Dog` 专属成员。

### some 用于返回值和存储属性

`some Speaker` 称为不透明类型：它代表某一个固定的、遵循 `Speaker` 的具体类型，调用方看不到类型名称，但编译器知道它是哪个类型。用作返回值时，具体类型由函数实现决定，所有返回语句必须返回同一个具体类型（称为底层类型）。

```swift
func makeDog() -> some Speaker {
    Dog()
}

func chooseSpeaker(_ useDog: Bool) -> any Speaker {
    if useDog { return Dog() }
    return Cat()
}

var fixedSpeaker: some Speaker = Dog()
// fixedSpeaker = Cat() // 编译错误：不能换成另一种遵循类型

var flexibleSpeaker: any Speaker = Dog()
flexibleSpeaker = Cat() // 允许

let speakers: [any Speaker] = [Dog(), Cat()]
```

如果把 `chooseSpeaker` 的返回类型改成 `some Speaker`，两个分支分别返回 `Dog` 和 `Cat`，底层类型不一致，无法编译。在泛型函数中，不透明返回值的底层类型可以随泛型参数变化，但对同一组泛型参数，所有返回语句仍须返回同一个类型。

### some 和 any 用于参数

```swift
func show(_ value: some Speaker) {
    value.speak()
}

// 对这个函数，等价于显式泛型写法：
func showGeneric<T: Speaker>(_ value: T) {
    value.speak()
}

show(Dog())
show(Cat())
```

参数位置的 `some Speaker` 是匿名泛型参数的简写，具体类型由调用方在每次调用时决定，不同调用可以传入不同类型。返回值位置的 `some Speaker` 正好相反，具体类型由函数实现决定。

```swift
func showSome(_ value: some Speaker) {
    var current = value
    // current = Cat() // 编译错误：current 的类型是本次的泛型类型
    current.speak()
}

func showAny(_ value: any Speaker) {
    var current = value
    current = Cat() // 允许：current 的类型是 any Speaker
    current.speak()
}
```

`current` 的类型是本次调用传入的具体类型。函数体要对所有可能传入的类型都成立，如果传入的是 `Dog`，把 `Cat()` 赋给 `current` 就是类型错误，所以编译器拒绝这行代码；即使调用 `showSome(Cat())` 也一样。若确实需要更换具体类型，可以在泛型函数中显式声明 `var current: any Speaker = value`。

两个分别写成 `some Speaker` 的参数表示两个独立的匿名泛型参数，不自动要求类型相同。需要表达多个参数或返回值之间的同类型关系时，应使用显式的同一个 `T`。

泛型调用不一定是静态派发，`any` 值也不一定分配在堆上，实际方式由编译优化决定。能否把 `any Speaker` 简写为 `Speaker`，取决于语言模式和是否启用 ExistentialAny 特性，与编译器版本号无关。

## 逃逸闭包与执行时机

**`@escaping` 表示闭包参数可以在函数返回之后继续被使用，这种情况称为闭包逃逸。** 例如，函数把闭包保存到属性中，闭包就可以在函数返回之后被调用。`@escaping` 只是一项编译期标记，不会创建线程、安排任务或执行闭包。

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

如果函数只在返回前执行参数闭包，就不需要 `@escaping`，与参数叫什么名字无关。被标记为 `@escaping` 的闭包也可以同步执行；闭包是否异步执行、在哪个线程执行，由调用闭包的代码决定。

**省略 `@escaping` 时的编译限制。** 对于上例中的 `() -> Void` 闭包参数，不加 `@escaping` 时，参数默认是非逃逸的。非逃逸表示接收函数不能让这个闭包保留到函数返回之后。

如果把上例的 `register` 改为下面的写法，编译器会在 `pending = callback` 处报错：

```swift
func register(_ callback: () -> Void) {
    pending = callback // 编译错误：不能把非逃逸闭包参数保存到这里
}
```

`pending` 是 `CallbackHolder` 的实例属性。`register` 返回后，调用方仍可以通过 `holder.pending` 访问闭包。把 `callback` 存入这个属性会让闭包逃逸，因此参数必须声明为 `@escaping`。

编译器在保存闭包的那一行检查是否允许逃逸，所以去掉 `@escaping` 后，整段代码在编译阶段就会报错。

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

上例中，变量类型已经规定闭包接收 `String` 和 `Bool`，编译器据此推断 `$0`、`$1` 的类型。简写只省略了参数名称，参数数量不变，调用时仍需传入两个参数。

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

`[weak self]` 后面的 `in` 用于结束闭包头部。只要写了捕获列表，即使省略了参数名，也必须写 `in`。捕获的外部变量不是闭包参数，不会占用 `$0` 或 `$1`。

### 默认捕获与捕获列表

捕获使闭包能够在执行时访问定义位置的外部变量或值。没有写捕获列表时，闭包按默认方式捕获局部 `var`：闭包和外部代码共享同一个变量，而不是在创建闭包时复制一份值。

```swift
func captureVariable() {
    var count = 0
    let increment = { count += 1 }

    count = 10
    increment()
    print(count) // 11
}
```

`increment` 修改的就是外部的 `count`，所以先改成 10、再加 1，结果是 11。`Int` 虽然是值类型，默认捕获时仍然共享变量，不复制值。

显式写 `[count]` 时，Swift 在创建闭包的那一刻读取外部 `count`，用读到的值初始化闭包内部的一个常量。之后每次调用闭包，读到的都是这个常量，不再读取外部变量。

```swift
func captureValue() {
    var count = 0
    let printSaved = { [count] in
        print(count)
        // count += 1 // 编译错误：捕获列表中的 count 是常量
    }

    count = 10
    printSaved() // 0
}
```

### 捕获对象引用与对象内容

变量保存类实例引用时，`[object]` 在创建闭包时复制这条引用，对象本身不复制。默认捕获则共享外部变量，外部变量之后改为指向别的对象，闭包也会看到新对象。

```swift
final class CapturedCounter {
    var value = 0
}

func captureObject() {
    var object = CapturedCounter() // 对象 A
    let sharedVariable = { print(object.value) }
    let savedReference = { [object] in print(object.value) }

    object.value = 10
    savedReference() // 10：捕获的是 A 的引用，不是 A 的内容快照

    object = CapturedCounter() // 变量改为指向对象 B
    object.value = 20
    sharedVariable() // 20：读取外部变量当前指向的 B
    savedReference() // 10：仍指向创建闭包时的 A
}
```

`[object]` 捕获的是常量引用：闭包不能让它指向别的对象，但可以修改对象的可写属性。默认捕获和 `[object]` 都是强引用，会让对象保持存活；`[weak object]` 不会。`weak` 与 `unowned` 的访问条件见[weak 与 unowned 的使用条件](#weak-与-unowned-的使用条件)。

如果外部声明的是 `let object`，变量不会改为指向其他对象，默认捕获和 `[object]` 就指向同一个对象，都能看到对象属性之后的修改。

### 捕获存储与函数返回

Swift 闭包具有引用语义：把一个闭包赋给另一个变量，两个变量指向同一个闭包，共享同一份捕获状态。闭包不是 `class` 实例，但赋值行为与类实例相同。

捕获环境是闭包执行时访问外部变量或值所需的状态。创建闭包的函数返回后，闭包仍然可以使用捕获的变量。对于被闭包捕获、又会被修改的局部变量，编译器把变量放到一块由闭包共同持有的存储中，函数返回后这块存储仍然有效，闭包访问的不是已经失效的栈地址。

```swift
func makeIncrementer() -> () -> Int {
    var count = 0
    return {
        count += 1
        return count
    }
}

let first = makeIncrementer()
let second = first
print(first())  // 1
print(second()) // 2：访问同一份 count
```

再调用一次 `makeIncrementer()`，才会创建一个使用另一份 `count` 的新闭包。捕获的变量不一定都分配在堆上：只要程序的运行结果不变，编译器可以直接复制不会被修改的值，或者省去不必要的存储。

闭包的捕获环境同样由引用计数管理。实例强持有闭包、闭包又强捕获这个实例时，两者会形成强引用环。关联闭包的具体关系见[关联对象、稳定 key 与持有关系](swift-objc-runtime.md#关联对象稳定-key-与持有关系)。

## 多行字符串与脚本执行

**普通字符串用一对双引号包围；多行字符串用开头三个和结尾三个双引号包围。** 三引号属于 Swift 的字符串语法，不是 JavaScript 的语法。

```swift
let script = """
window.receiveProduct("123");
"""
```

开头是 `"""`，结尾也是 `"""`。内部的 `"123"` 是字符串内容的一部分，在之后交给 JS 执行的代码中充当字符串参数。

上面的内容也可以用普通字符串表示：

```swift
let script = "window.receiveProduct(\"123\");"
```

普通字符串中，`\"` 表示内容里的双引号，避免被当作 Swift 字符串的结束标记。两种写法在此示例中得到相同的字符串内容。创建这个字符串不会执行 JS，调用 `evaluateJavaScript` 后才由 WebKit 执行，见[Native 调用前端](../cross-platform/webview-and-lynx-bridge.md#wkwebviewnative-执行-javascript)。

## Data、字符串解码与打印

`Data` 是 Foundation 用来保存字节数据的类型。字符编码规定文字与字节序列之间的对应规则。编码把文字转换为字节；解码按指定规则把字节转换为文字。UTF-8 是一种字符编码，例如字符 `A` 对应字节 `0x41`。

### UTF-8 解码与 JSON 模型解码

`String(data:encoding:)` 按指定字符编码把 `Data` 解码为字符串。`encoding: .utf8` 指定按 UTF-8 规则把字节解释为文字。这个初始化方法返回 `String?`；数据无法按指定规则解码时，结果为 `nil`。

```swift
import Foundation

let data = Data([0x41])
if let text = String(data: data, encoding: .utf8) {
    print(text) // A
}
```

JSON 是一种文本数据格式。把包含 JSON 的 `Data` 转换为 `String`，只能得到 JSON 文本：既不会生成业务模型，也不会检查文本是否是合法 JSON。`JSONDecoder` 则按照 JSON 结构和目标类型构建模型；Alamofire 的具体调用过程见 [响应容器、Result 与模型值](../network-and-storage/network-and-proxy.md#响应容器result-与模型值)。

### 可选字符串的打印与换行

`String(data:encoding:)` 只把字节解码为文字，不会给 JSON 添加缩进或换行。JSON 文本可以写在一行，也可以在结构之间包含换行和空格。打印普通 `String` 时，`print` 原样输出字符串中的字符，遇到换行字符就换行。

直接打印 `String?` 时，Swift 使用可选值的调试表示：有值时显示 `Optional("…")`，没有值时显示 `nil`。字符串内部的实际换行字符会显示为可见的 `\n`，双引号会显示为 `\"`。

```swift
let value: String? = "A\nB"
print(value as Any) // Optional("A\nB")

if let text = value {
    print(text)
}
```

`as Any` 把可选值显式转换为 Any 再传给 `print`，用来消除编译器的警告；这个转换不解包，也不修改字符串。`if let` 取出字符串之后，第二次打印输出：

```text
A
B
```

两次打印的字符串内容相同。第一次打印的是可选值的调试表示，换行显示为 `\n`；第二次打印的是 `if let` 取出的普通字符串，换行字符直接换行。可选绑定的条件和作用域见 [可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)。

源码中的 `"A\nB"` 包含一个实际换行字符；`"A\\nB"` 包含反斜杠和字母 `n` 两个普通字符。后一种内容不会因为解包或打印而自动变成换行：

```swift
print("A\\nB") // A\nB，A 和 B 在同一行
```

JSON 字符串字段中的 `\n` 是两个字符：反斜杠和 `n`。`String(data:encoding:)` 得到的 JSON 文本保留这两个字符；用 JSON 解码器解析这个字段后，才得到包含实际换行字符的字段值。JSON 结构之间用于排版的换行则本来就是实际换行字符。

## map、KeyPath、Sequence、Collection 与 Set

下面沿用[策略示例](../engineering/design-patterns.md#策略模式与评论排序)中类型为 [Comment] 的 result。`result.map(\.text)` 是 map 配合键路径字面量的写法，下面三种写法等价：

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

result 是 [Comment]，text 是 String，因此结果是 [String]。完整键路径也可写成 `result.map(\Comment.text)`。map 需要一个 `(Comment) -> String` 函数，编译器允许直接写键路径字面量 `\.text` 代替这个函数。这种替代只适用于直接写出的字面量；已经保存在变量中的 KeyPath 不能直接传给 map。

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

`textPath` 不保存任何一条评论的文字；把 `textPath` 应用到不同的 Comment，会读到各自的 text。键路径也适用于 struct，前面的 Comment 就是 struct。上下文已知根类型时，\Comment.text 可以简写为 \\.text。

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

### 集合类型与 for-in 遍历

集合类型用于把多个数据元素组织成一个值，并提供保存、访问、遍历以及按类型规则修改元素的操作。`Array` 适合按位置保存一组元素，`Set` 适合去重和判断元素是否存在，`Dictionary` 适合通过键查找对应的值。

| 类型 | 保存规则 | 遍历得到的元素 | 遍历顺序 |
| --- | --- | --- | --- |
| `Array<Int>` | 允许重复元素，元素有明确位置 | 一个 `Int` | 按数组索引顺序 |
| `Set<Int>` | 不保存重复元素 | 一个 `Int` | 不保证插入顺序或排序顺序 |
| `Dictionary<String, Int>` | 键唯一，不同键可以对应相同的值 | 一个 `(key: String, value: Int)` 元组 | 不保证插入顺序或排序顺序 |

`Sequence` 协议规定逐个遍历元素的能力，`for-in` 使用这种能力。`Collection` 协议在此基础上增加可多次遍历和按索引访问等约定。`Array`、`Set`、`Dictionary` 都遵循这两个协议，因此都支持 `for-in`。

```swift
let nums = [1, 2, 2]
for num in nums {
    print(num) // 依次输出 1、2、2
}

let uniqueNums: Set<Int> = [1, 2, 2]
for num in uniqueNums {
    print(num) // 只包含 1 和 2，不保证输出顺序
}

let scores = ["小明": 90, "小红": 95]
for (name, score) in scores {
    print(name, score)
}

for entry in scores {
    print(entry.key, entry.value)
}
```

字典每次提供一个键值元组。`(name, score)` 把元组的两个成员分别绑定到名称上；`entry.key` 和 `entry.value` 则通过元组标签访问同一组数据。元组与字典的结构区别见[元组、字典与空元组 Void](#元组字典与空元组-void)。

“顺序不保证”是指程序不能依赖遍历顺序，并不是每次遍历的顺序都会变。需要固定顺序时，先对元素或键排序，再遍历排序结果。

```swift
let doubled = uniqueNums.map { $0 * 2 }
// doubled 是 [Int]，元素为 2 和 4，顺序不保证

let names = scores.map { $0.key }
// names 是 [String]，顺序不保证
```

对 `Set` 或 `Dictionary` 调用 `map`，结果都是数组。注意三个容易混淆的名称：“集合类型”泛指 `Array`、`Set`、`Dictionary` 这类保存多个元素的类型；`Collection` 是一个协议；`Set` 是一个具体类型。“集合类型”说的是用途，“值类型”说的是赋值语义，所以 `Array` 既是集合类型，也是值类型。

### stride 的步长、方向与边界

`stride` 是 Swift 标准库函数，用于生成按指定步长变化的序列。步长是相邻两次取值之间的差。`stride` 返回的序列可以用 `for-in` 遍历，不会预先创建保存全部结果的数组。以下示例使用 `Int`。

```swift
let right = 4
let left = 1

for i in stride(from: right, through: left, by: -1) {
    print(i)
}
// 依次输出：4、3、2、1
```

`from`、`through`、`to` 和 `by` 是参数标签，用于说明各个参数的含义。调用时使用 `through:` 或 `to:` 中的一种。

| 参数 | 含义 |
| --- | --- |
| `from:` | 起点；起点满足边界条件时，起点就是第一个值 |
| `through:` | 结束边界；允许包含终点 |
| `to:` | 结束边界；不包含终点 |
| `by:` | 步长；正数表示递增，负数表示递减，不能为 `0` |

`through:` 与 `to:` 的区别是是否允许取到终点。下面用 `Array(...)` 把序列中的值收集成数组，便于展示结果；普通 `for-in` 遍历不需要这一步。

```swift
Array(stride(from: 4, through: 1, by: -1)) // [4, 3, 2, 1]
Array(stride(from: 4, to: 1, by: -1))      // [4, 3, 2]

Array(stride(from: 1, through: 4, by: 1))  // [1, 2, 3, 4]
Array(stride(from: 1, to: 4, by: 1))       // [1, 2, 3]
```

生成的值必须满足以下边界条件，起点也不例外。判断方向由步长的正负决定，与变量叫 `right` 还是 `left` 无关。

| 步长 | `through: end` 的取值条件 | `to: end` 的取值条件 |
| --- | --- | --- |
| 正数 | 当前值 `<= end` | 当前值 `< end` |
| 负数 | 当前值 `>= end` | 当前值 `> end` |

例如，`stride(from: right, through: left, by: -1)` 要求当前值大于或等于 `left`。`right == left` 时，起点满足条件，序列只包含这一个值，循环执行一次。`right < left` 时，起点已经小于 `left`，序列为空，循环一次也不执行，程序不会报错。

```swift
Array(stride(from: 2, through: 2, by: -1)) // [2]
Array(stride(from: 1, through: 2, by: -1)) // []
Array(stride(from: 2, to: 2, by: -1))      // []
```

如果把步长改成 `1`，`through: left` 对应的条件就是当前值小于或等于 `left`。起点等于终点时仍取值一次；起点大于终点时序列为空。使用 `to:` 时，无论步长是正数还是负数，起点等于终点都会得到空序列。

```swift
Array(stride(from: 2, through: 4, by: 1)) // [2, 3, 4]
Array(stride(from: 2, through: 2, by: 1)) // [2]
Array(stride(from: 4, through: 2, by: 1)) // []
Array(stride(from: 2, to: 2, by: 1))      // []
```

`through:` 允许包含终点，但不会为了包含终点而改变步长。只有按步长递进恰好到达终点时，序列才会包含终点。

```swift
Array(stride(from: 1, through: 4, by: 2)) // [1, 3]，不会额外加入 4
```

### 数组的 last 属性

`Array.last` 是读取数组最后一个元素的只读属性，使用时写作 `array.last`，不加调用括号。设数组元素类型为 `Element`，`last` 的类型就是 `Element?`：数组非空时返回包含最后一个元素的可选值，数组为空时返回 `nil`。读取 `last` 不会删除元素，也不会改变数组。

```swift
let numbers = [1, 4, 5]
let lastNumber: Int? = numbers.last // 包含 5

let empty: [Int] = []
let missing: Int? = empty.last     // nil
```

二维数组的元素本身是数组。因此，`[[Int]]` 的 `last` 类型是 `[Int]?`，不是 `Int?`。

```swift
let result: [[Int]] = [[1, 4], [10, 11]]
let lastInterval: [Int]? = result.last // 包含 [10, 11]
```

用 `result[result.count - 1]` 读取末尾元素前，必须先确认数组非空，否则下标越界会使程序崩溃。`result.last` 在数组为空时返回 `nil`，但使用其中的值前需要解包，即从可选值中取出内部值。可选类型的定义见[可选类型、状态表达与并发支持](#可选类型状态表达与并发支持)，读取后继续判断的写法见[可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)。

### 数组反转：reverse() 与 reversed()

反转是把元素的先后顺序颠倒。对 `Array` 而言，`reverse()` 修改调用数组的元素顺序；`reversed()` 保持原数组不变，返回一个按相反顺序访问元素的集合。

`reverse()` 会修改数组值，因此调用数组必须可写，例如使用 `var` 声明。方法返回 `Void`，表示不返回数组结果。

```swift
var nums = [1, 2, 3]
nums.reverse()
print(nums) // [3, 2, 1]
```

`reversed()` 不修改原数组，因此可以对 `let` 声明的数组调用。对于 `[Int]`，返回值类型是 `ReversedCollection<[Int]>`，不是 `[Int]`。`ReversedCollection` 包装原集合，按相反顺序提供元素访问。需要新的数组时，使用 `Array(nums.reversed())`。

```swift
let nums = [1, 2, 3]
let reversedNumbers = nums.reversed()

for number in reversedNumbers {
    print(number) // 依次输出 3、2、1
}

let newNumbers: [Int] = Array(nums.reversed())
print(newNumbers) // [3, 2, 1]
print(nums)       // [1, 2, 3]
```

### 数组值语义、写时复制与返回值

对 [Int] 而言，值语义保证：数组赋值后，修改一个数组的元素不会改变另一个数组。值语义只规定程序能观察到的结果，不要求每次赋值或返回数组时立即复制全部元素。

Array 本身是结构体，元素保存在单独分配的缓冲区中。缓冲区是存放元素的一块内存，由一个存储对象管理。数组值只保存指向这个存储对象的引用，所以数组值和存储对象是两样东西。

Array 使用写时复制（Copy-on-Write，简称 CoW）：赋值时多个数组值共享同一个存储对象；修改前先检查存储对象是否只被当前数组引用。存储对象仍被共享时，先复制一份再修改；已经独占且容量足够时，直接修改。所以 CoW 只在“共享状态下写入”时才复制。

```swift
let original = [1, 2, 3]
var copy = original
copy[0] = 99

print(original) // [1, 2, 3]
print(copy)     // [99, 2, 3]
```

赋值之后、修改之前，original 和 copy 是两个数组值，共享同一个存储对象。下图的箭头表示数组内部持有的存储引用，只用来说明共享关系，不代表实际内存布局。

```text
original ──┐
           ├──→ 缓冲区 A：[1, 2, 3]
copy ──────┘
```

执行 copy[0] = 99 时，数组先检查存储对象是否被独占。此时存储对象被共享，修改过程是：

1. 分配新的元素缓冲区 B。
2. 将 A 中的元素复制到 B。
3. 将 copy 内部的存储引用改为指向 B。
4. 将 B 的第一个元素改为 99；original 继续使用 A。

```text
original ─────→ 缓冲区 A：[ 1, 2, 3]
copy ─────────→ 缓冲区 B：[99, 2, 3]
```

检查和复制由下标赋值、追加元素等修改方法完成。数组根据存储对象的强引用计数判断是否被独占；检查的对象是存储对象，不是数组结构体，也不是数组中的元素。

函数返回局部数组后，局部变量已经结束，只有接收结果的变量还引用存储对象，这时修改元素直接在原存储上进行。是否复制只看修改的那一刻存储对象是否被共享，与之前是否发生过返回或赋值无关。

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

存储被共享时，修改方会分配一块能容纳全部现有元素的新存储，新存储的容量不一定与旧存储相同。容量是不重新分配就能容纳的元素数量。即使数组独占存储，追加元素超过容量时，也要重新分配。

数组的写时复制不会自动复制元素引用的类实例。对 [Box] 而言，数组元素保存 Box 的引用；修改 Box.value 没有替换数组元素，所以两个数组仍能观察到同一个对象的变化。下面是独立示例。

```swift
final class Box {
    var value: Int

    init(_ value: Int) {
        self.value = value
    }
}

let original = [Box(1)]
var copied = original

copied[0].value = 2
print(original[0].value) // 2：仍共享同一个 Box

copied[0] = Box(3)
print(original[0].value) // 2：替换 copied 的元素不改变 original
print(copied[0].value)   // 3
```

即使 copied 已经有了独立的存储，复制到新存储中的仍是原来那些 Box 的引用，不会变成新 Box。存储独立，不等于元素对象独立。需要对象副本时，要显式创建新对象；值内部持有对象引用的规则见[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)。

#### 自定义 CoW 与引用唯一性检查

自己声明的 struct 不会自动具备 CoW。成员只有整数这类直接保存的值时，复制本来就很便宜，不需要 CoW。结构体内部持有类实例时，复制结构体只复制对象引用，多个结构体副本共享同一个对象（见[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)）。如果希望这些副本能独立修改，就要自己实现 CoW：修改前确认内部对象只被当前副本引用。

实现方式是定义一个 Storage 类保存数据，结构体只持有 Storage 的引用。Storage 是开发者自己定义的普通类。下例的 Storage 只保存一个整数，仅用于演示；实际只有一个整数时，直接存在结构体里更简单。

isKnownUniquelyReferenced(&storage) 检查 storage 指向的类实例是否只有一个强引用。这个函数适用于任何类实例，只做检查，不创建或复制对象。参数必须是类实例引用，不能直接传入结构体。

返回 true 表示对象只有一个强引用；返回 false 表示还有其他强引用，CoW 实现应先创建新的存储对象。函数只统计强引用，weak 和 unowned 引用不计入。

```swift
struct Counter {
    private final class Storage {
        var value: Int

        init(value: Int) {
            self.value = value
        }
    }

    private var storage: Storage

    init(value: Int) {
        storage = Storage(value: value)
    }

    var value: Int {
        get { storage.value }
        set {
            if !isKnownUniquelyReferenced(&storage) {
                storage = Storage(value: storage.value)
            }
            storage.value = newValue
        }
    }
}

let a = Counter(value: 1)
var b = a
b.value = 100

print(a.value) // 1
print(b.value) // 100
```

var b = a 之后，a.storage 和 b.storage 指向同一个 Storage 对象。设置 b.value 时，setter 先检查 b.storage：对象还被 a 引用，所以 setter 用原来的整数创建一个新 Storage，让 b.storage 指向新对象，再写入 100。a 仍读取旧对象中的 1。如果对象只被 b 引用，setter 直接修改原对象。

如果删除 setter 中的检查和复制，只保留 storage.value = newValue，a 和 b 会一直指向同一个 Storage，修改 b.value 后 a.value 也变成 100。复制 Counter 只复制了 Storage 的引用。

storage 声明为 private，外部只能通过 value 写入，无法绕过 CoW 检查直接修改内部对象。&storage 把变量作为 inout 参数传入，访问规则见[inout 与内存独占访问](#inout-与内存独占访问)。

isKnownUniquelyReferenced 不是锁。多个线程同时读写同一个 Counter 变量仍然是数据竞争，需要另外同步。

### ArraySlice 的存储保留与索引

ArraySlice 表示数组中的一段连续元素。创建切片时，切片不复制元素，而是引用原数组的存储对象，并记录切片的范围。切片本身是值类型，但切片与原数组共享存储。

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

suffix(10) 取出最后 10 个元素。numbers 在函数返回时已经结束，但 preview 仍引用原数组的整块存储，100 万个元素都不会释放。返回切片或给切片赋值，都不会把这 10 个元素复制成独立数组。所以只需要长期保存少量元素时，不要长期持有切片。

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

新数组不引用原数组的存储，原存储在没有其他引用后就会释放。释放后，进程的物理内存统计不一定立即下降，因为内存分配器可能把这块空间留着供之后复用。

## 元组、字典与空元组 Void

元组把固定数量的值组合成一个值。每个位置的类型在声明时确定，不同位置可以是不同类型，元组不能添加新元素。字典保存键到值的映射；同一个字典的键遵循统一的 `Key` 类型，值遵循统一的 `Value` 类型，键必须满足 `Hashable` 约束。

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

元组标签是元组类型的一部分，在编译时确定，不能像字典的键那样在运行时查询。上例的元组适合组合一个人的不同字段，字典适合按姓名查找同一类数据（年龄）。

`Void` 是空元组类型 `()` 的别名。空元组有且只有一个值，也写作 `()`；这个值不包含任何元素。`Void` 有一个值 `()`，这与 `nil`（可选值没有值）和 `Never`（没有任何值）都不同。例如 `() -> Void` 左侧的 `()` 表示零个参数，右侧的 `Void` 表示返回空元组。

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

PostHistoryManager 没有遵循 Equatable，所以不能用 ==。类型要支持 ==，需要遵循 Equatable 并实现相等规则；类不会自动获得 ==。

例如可以规定两个用户对象的 ID 相同时 == 为 true，但两个对象仍是两个实例，=== 为 false。判断单例是否为同一对象，使用 ===。

=== 用于比较类实例的身份，也支持可选对象引用。struct 和 enum 是值类型，不能直接使用它比较。

```swift
let a = [1, 2]
let b = [1, 2]

a == b // true，元素相等
// a === b // 编译错误，Array 是值类型
```

=== 不能用于数组，既不能比较数组内容，也不能检查两个数组是否共享写时复制的存储。比较数组内容用 ==。

## defer 与作用域退出

defer 是 Swift 的延后执行语句，用于在离开当前作用域之前执行一段代码。这里的作用域指包围 defer 的代码块，例如函数体或 do 代码块。程序执行到 defer 语句时，只是登记这段代码，不会立即执行。

只有已经执行到的 defer 语句，才会在退出作用域时执行。退出方式可以是正常执行完代码块、通过 return 提前返回，或通过抛出 Swift 错误离开作用域。如果程序在执行到 defer 之前就退出，defer 中的代码不会执行。

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

函数先获取锁，再执行到 defer 语句，并继续运行后续代码。shouldSkip 为 true 时，函数在返回调用方之前解锁；shouldSkip 为 false 时，函数输出文字，并在函数结束之前解锁。多个退出位置共用同一条 defer，不必在每个 return 前都写一次解锁。

defer 只在函数退出时执行，不会主动结束函数，也不会超时解锁。如果函数一直等待另一把锁、重复获取自己已经持有的 NSLock，或者进入无限循环，函数无法退出，defer 中的解锁代码也就不会执行。所以 defer 只能避免漏写解锁，加锁顺序和重复加锁仍需自己检查。
