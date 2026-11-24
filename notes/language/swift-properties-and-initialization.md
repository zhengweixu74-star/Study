# Swift 属性、初始化方法与扩展限制

[返回笔记目录](../README.md)

本文说明 Swift 的存储属性与计算属性、lazy 属性、实例属性的初始化、指定与便利初始化方法、初始化方法的继承、类的两阶段初始化，以及扩展中能声明和不能声明的属性。

## 本文目录

- [存储属性、计算属性与访问器](#存储属性计算属性与访问器)
  - [实例 lazy 存储属性](#实例-lazy-存储属性)
- [实例存储属性与局部变量的初始化](#实例存储属性与局部变量的初始化)
- [指定初始化方法与便利初始化方法](#指定初始化方法与便利初始化方法)
- [required、override 与初始化方法继承](#requiredoverride-与初始化方法继承)
- [UIViewController 的 init?(coder:) 与纯代码初始化](#uiviewcontroller-的-initcoder-与纯代码初始化)
- [类的两阶段初始化与 self 使用时机](#类的两阶段初始化与-self-使用时机)
- [扩展中的类型属性与实例存储限制](#扩展中的类型属性与实例存储限制)
  - [为什么扩展不能添加实例存储属性](#为什么扩展不能添加实例存储属性)

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

### 实例 lazy 存储属性

普通实例的 lazy 存储属性在首次读取时执行初始化并保存结果，后续读取已有值。lazy 适合初始化开销大、又不一定会用到的属性。多个线程同时首次读取实例 lazy 属性时，初始化代码可能执行多次；`static let` 则保证只初始化一次，并且线程安全。

## 实例存储属性与局部变量的初始化

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

## 指定初始化方法与便利初始化方法

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

## required、override 与初始化方法继承

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

## UIViewController 的 init?(coder:) 与纯代码初始化

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

## 类的两阶段初始化与 self 使用时机

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
