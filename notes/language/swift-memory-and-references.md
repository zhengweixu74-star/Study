# Swift 内存管理、值语义与对象引用

[返回笔记目录](../README.md)

本文说明 Swift 中类型、实例与对象引用的关系，值类型的复制、参数传递与生命周期，ARC 与 Android GC 的区别，weak 与 unowned，闭包和 Delegate 的持有关系，以及对象复制和相等规则。

ARC（自动引用计数）按强引用的数量管理类实例的生命周期。编译器在代码中插入增减计数的操作：新增一条强引用时计数加 1，去掉一条强引用时计数减 1，计数归零时实例被释放。weak 引用不计入强引用计数，所以不会让实例继续存活。闭包捕获 self 时，要看 self 是否也直接或间接强引用这个闭包：两者互相强引用才形成引用环；没有形成引用环时，捕获只是让 self 活到闭包释放为止。ARC 与垃圾回收（GC）的区别见 [ARC 与 Android GC](#arc-与-android-gc引用计数可达性与引用环)。

## 本文目录

- [类型、实例与对象引用](#类型实例与对象引用)
- [值语义与值类型](#值语义与值类型)
  - [值类型与引用类型的选择](#值类型与引用类型的选择)
  - [值类型方法中的 mutating](#值类型方法中的-mutating)
  - [普通函数参数与局部变量](#普通函数参数与局部变量)
  - [inout 与内存独占访问](#inout-与内存独占访问)
  - [结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)
- [ARC 与 Android GC：引用计数、可达性与引用环](#arc-与-android-gc引用计数可达性与引用环)
- [weak 与 unowned 的使用条件](#weak-与-unowned-的使用条件)
- [闭包与 Delegate](#闭包与-delegate)
  - [概念与职责](#概念与职责)
  - [选型依据](#选型依据)
  - [使用场景](#使用场景)
  - [适用边界](#适用边界)
  - [生命周期与持有关系](#生命周期与持有关系)
- [引用赋值、浅拷贝与深拷贝](#引用赋值浅拷贝与深拷贝)
  - [Swift 中的深拷贝实现方法](#swift-中的深拷贝实现方法)
- [相等规则与对象身份](#相等规则与对象身份)

## 类型、实例与对象引用

Person 是一个具体的类型名；class 是声明类类型的关键字，struct、enum 是声明其他类型的关键字。“引用类型”是对类型的分类，class 声明的类型都是引用类型；“引用”则是运行时指向某个对象的值。

```swift
class Person {}
let person = Person()
```

Person 是类型，Person() 创建 Person 类的一个实例，person 常量保存指向这个实例的引用。struct 和 enum 同样有实例，例如 `Point(x: 1, y: 2)` 是 Point 结构体的实例，`Direction.left` 是 Direction 枚举的实例。

Swift 语境中的“对象”通常指 class 实例；“实例”也包括 struct 和 enum 的具体值。

## 值语义与值类型

值语义表示 struct、enum 以及 Array、String 等常见值类型在赋值和传参时表现为值的复制。底层可以采用写时复制共享存储，不要求每次赋值都立即深拷贝；值中包含 class 成员时，多份值仍可能引用同一个对象。Swift 同时提供 class 的引用语义和运行时动态能力，并非所有类型都是值类型。

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

## ARC 与 Android GC：引用计数、可达性与引用环

Swift 类实例通常由 ARC（自动引用计数）管理。ARC 跟踪对象的强引用计数；当强引用计数归零时，对象可以释放。Android 上的 Kotlin／Java 对象主要由追踪式垃圾回收（GC）管理。GC 从一组起点出发，沿对象引用找出仍然可达的对象，再回收不可达的对象。

ARC 的计数操作由编译器生成。retain 让对象的引用计数加 1，release 让引用计数减 1，计数减到 0 时对象被释放，定义见[自动释放池与 AutoreleasePoolPage](swift-objc-runtime.md#自动释放池与-autoreleasepoolpage)。早期 Objective-C 使用 MRC（手动引用计数），程序员在代码中自己调用 retain 和 release。使用 ARC 后，编译器在编译时分析每条强引用从哪里开始、到哪里结束，并在生成的机器码中写入 `swift_retain`、`swift_release` 调用；Objective-C 对象对应的是 `objc_retain`、`objc_release`。App 运行时逐个执行这些调用。

```swift
func show() {
    let user = User(name: "Tom")
    let other = user
    print(other.name)
}
```

不经优化时，编译器为 `show()` 生成的代码在逻辑上相当于：

```text
user = 创建 User 实例        // 引用计数 = 1
swift_retain(user)           // other 也指向这个实例，引用计数 = 2
other = user
print(other.name)
swift_release(other)         // other 的作用域结束，引用计数 = 1
swift_release(user)          // user 的作用域结束，引用计数 = 0，调用 deinit 并释放内存
```

编译器优化时会删除多余的成对调用，例如 `other` 的那一对 retain 和 release；`show()` 结束时，`User` 实例同样被释放。

所以 ARC 在编译时确定计数变化的位置，在运行时执行计数操作，对象在最后一次 release 让计数归零时立即释放。GC 在运行时查找不可达对象，不可达对象要等到下一次 GC 运行时才被回收，GC 的运行时机见[ART 管理对象内存](../android/system-architecture-and-app-launch.md#art-管理对象内存)。weak 引用指向实例时，编译器不为这条引用插入 retain，所以 weak 引用不增加强引用计数。

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

## weak 与 unowned 的使用条件

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

## 闭包与 Delegate

### 概念与职责

**概念区别。** 闭包是可以作为值传递的代码块，闭包可以捕获定义位置的变量。在回调场景中，使用组件的一方提供具体行为，组件在约定的时机调用闭包。Delegate 是一种委托模式：一个对象把约定的职责交给另一个对象。Swift 通常通过协议定义这些职责，由遵循协议的对象实现对应方法。

### 选型依据

**选型原则。** 先判断使用组件的一方要提供的是“一段回调行为”，还是“一个承担明确职责的对象”。只需要传入一段具体的回调行为时，选择闭包；需要由一个对象统一承担一组相关职责时，选择 Delegate。事件的数量和频率不决定选型。

| 判断维度 | 更适合闭包 | 更适合 Delegate |
| --- | --- | --- |
| 回调内容 | 一个或少量独立的结果、动作 | 一组共同描述某种职责的方法 |
| 代码组织 | 希望在发起操作的位置直接写回调 | 希望由接收对象集中实现相关方法 |
| 对象关系 | 只需要提供具体行为，无须指定接收对象的角色 | 需要明确某个对象担任事件接收者或决策者 |
| 状态管理 | 回调只使用少量局部变量 | 多个方法需要共享接收对象维护的状态 |

### 使用场景

**闭包示例：图片导出。** 使用组件的一方发起一次导出，并传入处理导出结果的闭包。导出完成后，导出组件把成功结果或错误传给闭包。使用组件的一方在发起导出的位置直接编写后续行为，代码比较集中。

**Delegate 示例：编辑器交互。** 编辑器需要通知“开始编辑”“内容变化”“结束编辑”，还可能询问“是否允许结束编辑”。这些方法共同构成编辑器委托对象的职责。由同一个对象集中实现这些方法，职责更清楚。

### 适用边界

**选型边界。** 调用次数和返回值都不决定选型。闭包可以反复调用，例如持续触发的进度回调；Delegate 方法也可以只调用一次。两种方式都能通知事件，也都能返回决策结果。

### 生命周期与持有关系

**持有关系。** 组件保存闭包时，闭包可能强引用捕获的对象，因此需要检查循环引用。Delegate 属性通常声明为 weak，但 weak 不会保持委托对象存活，必须有其他对象负责持有委托对象。

下面用按钮通知页面说明委托调用。`ButtonDelegate` 定义通知方法，`Page` 实现这个方法，`Button` 在点击时调用委托对象。`AnyObject` 的类类型限制见[协议继承与类继承的区别](swift-basics.md#协议继承与类继承的区别)。

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

## 引用赋值、浅拷贝与深拷贝

引用赋值复制对象引用，不创建对象副本。浅拷贝创建新的外层对象，但继续共享嵌套对象。深拷贝除了创建新的外层对象，还为需要独立修改的嵌套对象创建副本，修改副本不会影响原对象。判断是哪种复制，要检查嵌套的可修改对象是否仍被共享，只看最外层对象是不是新地址不够。

下面的 Address 和 Person 是独立示例使用的模型。Codable 表示模型同时支持编码和解码，具体机制见[Swift Codable 与序列化复制](swift-codable-and-serialization.md)。

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

复制 struct 不一定是深拷贝：struct 内部的类引用只会复制引用。class 赋值也不是浅拷贝：赋值只复制引用，不创建任何新对象。结构体内部的类引用如何复制，见[结构体、枚举与值的生命周期](#结构体枚举与值的生命周期)；数组中的类实例见[数组值语义、写时复制与返回值](swift-collections.md#数组值语义写时复制与返回值)。

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

编码后再解码可以重建模型，例如使用 JSONEncoder 和 JSONDecoder，或使用 PropertyListEncoder 和 PropertyListDecoder。副本是否完整、是否独立，取决于编码时保存了哪些属性，以及解码时是否创建了新对象，详见[序列化复制的边界](swift-codable-and-serialization.md#序列化复制的边界)。

NSKeyedArchiver 和 NSKeyedUnarchiver 是 Foundation 的对象归档与解档工具。归档是按编码协议保存对象及相关状态，解档是读取归档数据重建对象。参与归档的对象通常遵循 NSCoding；使用安全编码时还需满足 NSSecureCoding。归档会记录对象之间的引用关系；解档得到的副本是否满足深拷贝要求，取决于归档时保存了哪些属性、解档时如何重建对象。

递归复制时，如果多个属性指向同一个对象，或者对象之间互相指向，复制代码需要用一张表记录“原对象 → 副本”。再次遇到同一个原对象时，直接使用表中的副本。这样副本之间也保持原来的共享关系，复制也不会因循环指向而无限递归。

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
