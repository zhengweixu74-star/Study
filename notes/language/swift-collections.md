# Swift 集合、KeyPath 与写时复制

[返回笔记目录](../README.md)

本文说明 Swift 的 map 与 KeyPath、Sequence、Collection 与 Set、集合遍历、stride、数组的常用操作、数组值语义与写时复制、ArraySlice，以及集合与泛型协议约束。

## 本文目录

- [map、KeyPath、Sequence、Collection 与 Set](#mapkeypathsequencecollection-与-set)
- [集合类型与 for-in 遍历](#集合类型与-for-in-遍历)
- [stride 的步长、方向与边界](#stride-的步长方向与边界)
- [数组的 last 属性](#数组的-last-属性)
- [数组反转：reverse() 与 reversed()](#数组反转reverse-与-reversed)
- [数组值语义、写时复制与返回值](#数组值语义写时复制与返回值)
  - [自定义 CoW 与引用唯一性检查](#自定义-cow-与引用唯一性检查)
- [ArraySlice 的存储保留与索引](#arrayslice-的存储保留与索引)
- [集合、泛型与协议约束](#集合泛型与协议约束)

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

## 集合类型与 for-in 遍历

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

字典每次提供一个键值元组。`(name, score)` 把元组的两个成员分别绑定到名称上；`entry.key` 和 `entry.value` 则通过元组标签访问同一组数据。元组与字典的结构区别见[元组、字典与空元组 Void](swift-basics.md#元组字典与空元组-void)。

“顺序不保证”是指程序不能依赖遍历顺序，并不是每次遍历的顺序都会变。需要固定顺序时，先对元素或键排序，再遍历排序结果。

```swift
let doubled = uniqueNums.map { $0 * 2 }
// doubled 是 [Int]，元素为 2 和 4，顺序不保证

let names = scores.map { $0.key }
// names 是 [String]，顺序不保证
```

对 `Set` 或 `Dictionary` 调用 `map`，结果都是数组。注意三个容易混淆的名称：“集合类型”泛指 `Array`、`Set`、`Dictionary` 这类保存多个元素的类型；`Collection` 是一个协议；`Set` 是一个具体类型。“集合类型”说的是用途，“值类型”说的是赋值语义，所以 `Array` 既是集合类型，也是值类型。

## stride 的步长、方向与边界

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

## 数组的 last 属性

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

用 `result[result.count - 1]` 读取末尾元素前，必须先确认数组非空，否则下标越界会使程序崩溃。`result.last` 在数组为空时返回 `nil`，但使用其中的值前需要解包，即从可选值中取出内部值。可选类型的定义见[可选类型、状态表达与并发支持](swift-basics.md#可选类型状态表达与并发支持)，读取后继续判断的写法见[可选绑定、条件逗号与逻辑与](swift-basics.md#可选绑定条件逗号与逻辑与)。

## 数组反转：reverse() 与 reversed()

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

## 数组值语义、写时复制与返回值

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

检查和复制由下标赋值、追加元素等修改方法完成。数组根据存储对象的强引用计数判断是否被独占，强引用计数是 ARC 记录的、当前有多少条强引用指向这个对象的数量；检查的对象是存储对象，不是数组结构体，也不是数组中的元素。

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

即使 copied 已经有了独立的存储，复制到新存储中的仍是原来那些 Box 的引用，不会变成新 Box。存储独立，不等于元素对象独立。需要对象副本时，要显式创建新对象；值内部持有对象引用的规则见[结构体、枚举与值的生命周期](swift-memory-and-references.md#结构体枚举与值的生命周期)。

### 自定义 CoW 与引用唯一性检查

自己声明的 struct 不会自动具备 CoW。成员只有整数这类直接保存的值时，复制本来就很便宜，不需要 CoW。结构体内部持有类实例时，复制结构体只复制对象引用，多个结构体副本共享同一个对象（见[结构体、枚举与值的生命周期](swift-memory-and-references.md#结构体枚举与值的生命周期)）。如果希望这些副本能独立修改，就要自己实现 CoW：修改前确认内部对象只被当前副本引用。

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

storage 声明为 private，外部只能通过 value 写入，无法绕过 CoW 检查直接修改内部对象。&storage 把变量作为 inout 参数传入，访问规则见[inout 与内存独占访问](swift-memory-and-references.md#inout-与内存独占访问)。

isKnownUniquelyReferenced 不是锁。多个线程同时读写同一个 Counter 变量仍然是数据竞争，需要另外同步。

## ArraySlice 的存储保留与索引

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

## 集合、泛型与协议约束

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

泛型用类型参数编写适用于多种类型的代码，同时保留输入与输出之间的类型关系。上例 T 表示本次调用的元素类型；输入 [String]，返回 String?，不需要先返回 Any 再手动转换。协议定义类型必须满足的能力要求；T: Equatable 要求 T 支持相等比较，让函数内部可以使用 ==。更完整的协议约束与存在类型比较见[协议存在类型与泛型约束](swift-basics.md#协议存在类型与泛型约束)。
