# Swift 基础语法与类型系统

[返回笔记目录](../README.md)

本文说明 Swift 的类型安全与类型转换、可选类型与可选绑定、函数类型与错误处理、闭包、协议与 some／any、声明与引用、C 接口互操作，以及字符串、元组和 defer 等语法。内存管理、并发、属性与初始化、集合、Codable 分别见各自的笔记，入口见[笔记目录](../README.md#语言与运行时)。

## 本文目录

- [类型安全与动态能力](#类型安全与动态能力)
  - [静态类型、类型推断与数值转换](#静态类型类型推断与数值转换)
  - [运行时类型转换与安全边界](#运行时类型转换与安全边界)
- [可选类型、状态表达与并发支持](#可选类型状态表达与并发支持)
- [可选绑定、条件逗号与逻辑与](#可选绑定条件逗号与逻辑与)
- [函数签名与回调类型](#函数签名与回调类型)
  - [throws、try 与错误处理](#throwstry-与错误处理)
- [逃逸闭包与执行时机](#逃逸闭包与执行时机)
- [闭包参数、捕获列表与可选回调](#闭包参数捕获列表与可选回调)
  - [默认捕获与捕获列表](#默认捕获与捕获列表)
  - [捕获对象引用与对象内容](#捕获对象引用与对象内容)
  - [捕获存储与函数返回](#捕获存储与函数返回)
- [协议继承与类继承的区别](#协议继承与类继承的区别)
- [协议存在类型与泛型约束](#协议存在类型与泛型约束)
  - [泛型保留类型关系，any 隐藏具体静态类型](#泛型保留类型关系any-隐藏具体静态类型)
  - [some 用于返回值和存储属性](#some-用于返回值和存储属性)
  - [some 和 any 用于参数](#some-和-any-用于参数)
- [符号引用与对象引用](#符号引用与对象引用)
- [声明、定义、实现与引用](#声明定义实现与引用)
- [Swift 与 C 接口互操作](#swift-与-c-接口互操作)
- [多行字符串与脚本执行](#多行字符串与脚本执行)
- [Data、字符串解码与打印](#data字符串解码与打印)
  - [UTF-8 解码与 JSON 模型解码](#utf-8-解码与-json-模型解码)
  - [可选字符串的打印与换行](#可选字符串的打印与换行)
- [元组、字典与空元组 Void](#元组字典与空元组-void)
- [defer 与作用域退出](#defer-与作用域退出)

## 类型安全与动态能力

类型安全是值只能按照类型规则允许的方式使用。Swift 编译器检查赋值、操作、参数、返回值和类型约束；无法在编译期确定的部分需要运行时检查。类型匹配不要求类型名称完全相同，例如子类对象可以传给接受父类的参数。

动态能力包括根据实际类型调用类方法、调用协议要求、执行动态类型转换，以及 Apple 平台上的 Objective-C 消息派发。这些能力与编译期类型检查可以同时存在，具体派发机制见[Swift / Objective-C 运行时与方法派发](swift-objc-runtime.md)。

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

## 可选类型、状态表达与并发支持

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

async/await 让异步流程按书写顺序表达，不必再嵌套多层回调。await 标记一个可能挂起的位置；await 不会新建线程，也不会把工作移到后台。actor 让外部只能通过 actor 访问它的可变状态，同一时刻最多一个任务修改状态；但在 await 挂起期间，其他任务可能改变状态。细节见[actor 的默认隔离与重入](swift-concurrency.md#actor-的默认隔离与重入)。

## 可选绑定、条件逗号与逻辑与

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

本例的每个区间固定包含两个整数，因此 `last[1]` 可以读取第二个整数。`result.last` 的类型是 `[Int]?`，属性的返回规则见[数组的 last 属性](swift-collections.md#数组的-last-属性)。程序先检查可选值：有值时，程序声明类型为 `[Int]` 的常量 `last`，再计算 `interval[0] <= last[1]`；没有值时，绑定失败，程序跳过后续条件并进入 `else`。比较结果为 `false` 时也会进入 `else`。`last` 不能在对应的 `else` 中或整个 `if` 语句之后使用。

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

## 函数签名与回调类型

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

将内部函数改写成闭包表达式时，`makeReader` 的函数体可以写为 `return { value }`，返回类型仍是 `() -> Int`。`@Sendable () -> Int` 在函数类型上增加并发传递要求；具名函数和闭包表达式都必须满足该要求，具体见[Sendable 与 @Sendable](swift-concurrency.md#sendable-与-sendable)。

### throws、try 与错误处理

`throws` 表示函数可能抛错；`try` 本身不捕获错误，应使用 `do-catch` 捕获或向外传播。`try?` 在抛错时得到 nil 并丢弃具体错误；`try!` 在抛错时直接触发运行时错误，使程序崩溃。

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

`[object]` 捕获的是常量引用：闭包不能让它指向别的对象，但可以修改对象的可写属性。默认捕获和 `[object]` 都是强引用。强引用是会让 ARC 把对象的引用计数加 1、从而让对象保持存活的引用，ARC 的规则见[ARC 与 Android GC](swift-memory-and-references.md#arc-与-android-gc引用计数可达性与引用环)。`[weak object]` 是弱引用，不会让对象保持存活。`weak` 与 `unowned` 的访问条件见[weak 与 unowned 的使用条件](swift-memory-and-references.md#weak-与-unowned-的使用条件)。

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

## 协议继承与类继承的区别

协议定义能力要求，class、struct、enum 可遵循多个协议。类采用单继承，并通过继承复用实现。协议本身只声明要求；协议的 extension 可以为要求提供默认实现，遵循类型没有自己实现时使用默认实现。


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

这里沿用[按钮委托示例](swift-memory-and-references.md#闭包与-delegate)中的类型声明。`ButtonDelegate: AnyObject` 把协议限制为只能由类遵循，因此普通结构体和枚举不能遵循 `ButtonDelegate`。这个写法没有为遵循者指定父类。

`weak` 只能用于类实例的引用。`AnyObject` 约束使编译器能够确定 `any ButtonDelegate` 中保存的是类实例引用，因此允许声明 `weak var delegate: (any ButtonDelegate)?`。如果没有其他类类型限制，去掉 `AnyObject` 后，编译器就不能接受这条 `weak` 声明。

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

调用示例见[RunLoop、Timer 与 CADisplayLink 中的 Source0 示例](../ios/runloop-timer-and-displaylink.md#source0-示例后台压缩完成后刷新预览)。

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
