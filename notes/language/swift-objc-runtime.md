# Swift / Objective-C 运行时与方法派发

[返回笔记目录](../README.md)

## 本文目录

- [Objective-C 分类与类扩展](#objective-c-分类与类扩展)
- [Swift Runtime 与 Objective-C Runtime](#swift-runtime-与-objective-c-runtime)
- [Objective-C +load：调用时机与自动注册](#objective-c-load调用时机与自动注册)
- [关联对象、稳定 key 与持有关系](#关联对象稳定-key-与持有关系)
- [实例布局、静态存储与类型元数据](#实例布局静态存储与类型元数据)
- [对象引用与 UnsafePointer](#对象引用与-unsafepointer)
- [isa 与类型元数据指针](#isa-与类型元数据指针)
- [静态派发、动态派发与虚方法表](#静态派发动态派发与虚方法表)
- [编译阶段、机器码与内联](#编译阶段机器码与内联)
- [协议见证表与遵循关系](#协议见证表与遵循关系)
- [Objective-C 互操作、Selector 与 IMP](#objective-c-互操作selector-与-imp)
- [Selector 字符串与方法映射](#selector-字符串与方法映射)
- [@objcMembers 与 Objective-C 兼容限制](#objcmembers-与-objective-c-兼容限制)
- [KVO 观察关系与属性访问](#kvo-观察关系与属性访问)
- [自动 KVO 的动态子类机制](#自动-kvo-的动态子类机制)
- [KVO 通知选项与自动通知](#kvo-通知选项与自动通知)
- [dynamic 对 KVO 访问路径的影响](#dynamic-对-kvo-访问路径的影响)

## Objective-C 分类与类扩展

Category 可以扩展已有类或拆分实现，但不能直接增加实例变量。Category 声明属性不会自动合成新的实例变量与访问器。

Class Extension 随原类编译，可补充内部声明，并支持相应的实例变量与属性合成。源码中的内部声明不意味着运行时绝对不可调用。

## Swift Runtime 与 Objective-C Runtime

Runtime 是程序运行期间使用的一组支持代码和机制。Swift 编译器生成代码，Swift Runtime 为运行中的程序提供类型元数据管理、动态类型转换、泛型相关支持和内存管理等能力；部分工作由编译器生成的代码直接完成，并非每一步都调用某个统一的 Runtime 函数。

“运行时”作为时间阶段，是程序正在执行；“Runtime”作为组件，是支撑执行的运行时库。Swift Runtime 和 Objective-C Runtime 不是同一个组件，Apple 平台会让两者互操作。动态派发是在选择已经编译好的实现，不是临时把方法源码重新编译一遍。

Objective-C Runtime 支撑 OC 对象系统，可查询、添加、替换方法和管理关联对象。消息派发不会自动变成异步任务，也不是所有 Swift 方法调用都经过 OC Runtime。

## Objective-C +load：调用时机与自动注册

+load 是 Objective-C 提供的类方法。当类或分类被加载到 Objective-C Runtime 时，Runtime 会自动调用该类或分类自己实现的 +load。开发者可以在方法中编写需要在加载时执行的准备代码，无须先创建对象，也无须在业务代码中手动调用 +load。

对于随 App 启动加载的类和分类，+load 在 main 函数执行前调用。如果程序在启动后才动态加载某个库，该库中相关的 +load 会在后续加载时调用。因此，不能把所有 +load 都理解为一定在 main 之前执行。

### 示例：图片解析组件自动注册

假设开发者编写了 WebPDecoder，负责把 WebP 图片数据解码为可显示的图像。图片管理器保存一张“图片格式 → 解析器类”的映射表。注册是把解析器类及其支持的格式加入这张表，供图片管理器后续查找。

1. 开发者在 WebPDecoder 的 +load 中写入注册代码：向图片管理器登记“WebP 格式由 WebPDecoder 解析”。图片管理器及其注册接口需要支持在这个加载阶段使用。
2. WebPDecoder 被加载到 Objective-C Runtime。Runtime 自动调用 WebPDecoder 的 +load。注册代码随之执行，映射表中增加“WebP → WebPDecoder”这一项。这个阶段只登记解析能力，还没有解码具体图片。
3. 业务之后请求加载一张 WebP 图片。图片管理器识别图片格式，查询映射表，找到 WebPDecoder，再调用 WebPDecoder 的解码逻辑。

**+load 在这个例子中的作用，是提供自动执行注册代码的时机。真正完成注册的是开发者写在 +load 中的代码。**

仅仅定义 WebPDecoder 类，不会让自定义图片管理器自动知道“WebP → WebPDecoder”的对应关系。即使实现了一个空的 +load，也不会完成注册。如果把注册代码放到其他初始化入口，就需要保证该入口在第一次查找解析器之前执行。

### 与 App 启动耗时的关系

启动阶段执行的 +load 会占用启动时间。+load 中应只放确实需要在加载阶段完成的少量工作，避免同步读取大文件、同步等待网络结果或执行大量计算。耗时工作会推迟 main 函数及后续界面初始化的执行。

## 关联对象、稳定 key 与持有关系

class、struct、enum 都支持 extension，但扩展不能增加实例存储字段。给已有 UIView 附加 businessID，可用计算属性的 getter／setter 调用关联对象 API，由 Objective-C Runtime 在对象外保存附加值；原实例布局不变。

```text
view.businessID = 新值 → 计算属性 setter → objc_setAssociatedObject
读取 view.businessID   → 计算属性 getter → objc_getAssociatedObject
查找关系：宿主对象身份 + key → 附加值
```

**key 是指针标识。** 常用地址稳定的文件级变量：

```swift
// 类型和函数之外；private 在此限制在当前文件
private var businessIDKey: UInt8 = 0
// 读写都使用同一个 &businessIDKey
```

标识是变量地址，不是数字 0。同一 key 可配合不同 view 使用；同一宿主的不同附加属性应使用不同 key。不能每次创建临时局部变量再用其地址。实际声明还需满足工程 Swift 并发隔离规则。

**关联策略管理附加值。** retain 强持有附加对象；copy 保存复制结果；assign 不强持有，也不是自动置 nil 的 weak。NONATOMIC 表示关联访问策略，不能由此判断整个业务操作线程安全。方向是“宿主的关联关系 → 附加对象”；只有附加对象／闭包再强持有宿主才可能形成环。关联移除、替换或宿主销毁会解除这条持有，附加值能否释放仍取决于其他引用。

```swift
objc_setAssociatedObject(
    self,                              // 宿主
    &businessIDKey,                     // 稳定 key
    newValue,                          // 附加值
    .OBJC_ASSOCIATION_RETAIN_NONATOMIC  // 保存策略
)
```

newValue 只是 setter 参数名，不决定类型。class 值本身是对象引用；String 等值到 OC 边界可桥接为 NSString 或其他对象表示，retain 管理该对象表示，不依赖具体桥接盒布局。传 nil 移除该 key 的关联。

**附加值可桥接，不表示值类型能可靠充当宿主。** class 有稳定对象身份；struct／enum 赋值和传参表现为值复制，临时地址不是永久身份。第三个参数的包装对象会被关联持有；第一个参数若每次临时包装，设置时可能是对象 A，读取时却是对象 B，无法可靠找回关联。显式长期保存同一个 class 包装器可以提供身份，但关联属于包装器，不是扩展给任意值类型增加了实例字段。

## 实例布局、静态存储与类型元数据

实例布局是一个实例在内存中的组织方式：哪些字段存在，各自占多少空间，位于什么偏移，以及对齐和填充。class 实例除了业务存储属性，还包含运行时需要的对象头信息；具体布局随平台、类来源和实现而异。

普通原生 Swift 类实例可简化为：

```text
对象头：类型元数据指针、引用计数等管理信息
业务字段：age、name 等实例存储
必要的对齐／填充
```

对象头不是完整地存了一份类型说明或全部方法代码，而是通过类型指针找到共享的类元数据。OC 对象的 isa 及其他管理信息的编码方式，与原生 Swift 对象不应机械画成完全相同的逐字节布局。

关联对象在实例外管理附加值，计算属性执行访问逻辑，static 存储独立于实例；这三者都不要求给每个现有实例增加一个新业务字段。

**static 的存储：**

普通非泛型类型的 static 存储属性通常有编译器安排的独立全局／静态存储位置，通过相应访问逻辑取得。它属于类型层面的状态，不意味着这个值必须放在类元数据结构内部。

```text
person1 实例：自己的 age
person2 实例：自己的 age

Person.adultAge：独立的类型属性存储或优化后的常量表示
Person 类元数据：描述这个类型的信息
```

具体可能涉及程序映像的数据区和运行时初始化。常量还可能被折叠进使用它的代码，不能断言每个 static let 都永远占一块可见的独立内存。独立存储位置、类型元数据和每个实例的字段，是三个不同层次。

**类型元数据与方法：**

类型元数据是 Runtime 能使用的类型描述。通过它及相关描述符，可以获得类型类别、大小／布局、父类、泛型实参、相关方法入口等信息，支持类型转换、泛型运行、反射等。不同种类的类型，元数据结构并不相同。

方法实现是编译后的代码，通常位于代码区域。类元数据可以包含 Swift 虚方法表中的函数入口，或关联到 OC 方法信息；这不等于把所有方法的机器码都塞进元数据，也不表示所有函数都必须在某张虚方法表里。

```text
对象 → 类元数据 → 某个方法的实现入口 → 执行代码
```

类型元数据是被指向的数据，类型元数据指针是找到它的入口。两者不能画等号。

## 对象引用与 UnsafePointer

普通 class 引用在底层通常以对象地址相关的表示实现，概念上可以理解为“指向对象的值”。但 Swift 的对象引用是受类型系统和 ARC 管理的语言级值，不等于可随意做地址运算的裸指针。

```swift
let person: Person = Person()
```

`: Person` 告诉编译器 person 能被怎样使用。即使底层保存的主要是地址，编译器仍然知道这个地址代表 Person 引用，能检查允许访问哪些成员；对象的运行时类型信息则支持实际类型识别与动态派发。类型约束不要求把“Person”文字附在每个地址旁边。

`UnsafePointer<T>` 是 Swift 的显式、只读访问的类型化指针，可以通过 pointee 读取其指向的 T。`UnsafeMutablePointer<T>` 支持写入。它们也有类型参数，但不替调用者保证地址始终有效、内存已正确初始化或访问不越界，也不会仅因保存这个指针就自动延长所指对象／存储的生命周期。通过 withUnsafePointer 获得的临时指针应遵守其作用域和生命周期约束。

## isa 与类型元数据指针

isa 是对象用来找到所属类的入口，不是类型元数据本身，也不是成员调用方式的开关。

普通原生 Swift 类实例即使没有继承 NSObject，对象头里也有 metadata 指针。Apple 平台上，Swift 类元数据兼容 OC Class 对象，因此这个类型指针也承担 isa 的角色：

```text
同一个 Swift 对象
       │
       └─ 对象头中的类型入口
                    │
                    ▼
             Swift 类元数据
             同时兼容 OC Class
```

这不是“Swift 完全没有 isa，传给 OC 时才临时新增一个”；兼容关系在正常对象表示中已经建立。也不是“isa 先指向另一个 metadata 指针，后者再指向元数据”的两级关系。

限定范围很重要：不支持 OC 互操作的平台仍有 Swift 元数据指针，但不需要承担 OC isa 的作用；纯 OC 类的 Class 也不能一概视为完整 Swift 类元数据。实际 isa 还可能有编码等处理，逻辑关系不能替代对原始比特的合法解码。

不论某个属性有没有 dynamic，这个对象的类型入口都不会因此消失。有类型入口不代表每次调用都使用 OC 消息派发，也不代表所有 Swift 成员自动暴露成 selector 或自动支持 KVO。

## 静态派发、动态派发与虚方法表

运行时根据对象实际类型选择方法实现，是动态能力的一种，叫动态派发。更准确的是选择“这次调用哪个实现”，不一定是到运行时才创建对象或才知道接收对象是哪一个。

```swift
class Animal {
    func speak() { print("声音") }
}

class Dog: Animal {
    override func speak() { print("汪") }
}

func makeSound(_ animal: Animal) {
    animal.speak()
}
```

参数的声明类型是 Animal，传入的实际对象可以是 Dog。多态语义要求调用 Dog 的重写实现。如果编译器在某个调用位置已经能证明目标就是 Dog.speak，可以优化成直接调用甚至内联；改变的是实现路径，不能改变应有行为。动态能力还包括运行时类型转换等，不等同于动态派发这一项。

**静态派发：**

静态派发指调用目标在编译阶段已经可以确定，生成的调用直接指向相应实现，不必在运行时依据对象的实际类型再选择重写版本。它确定的是“调用哪个实现”，不是提前执行方法，也不是提前知道所有参数和返回值。

具体类型上的普通 struct 方法、不可重写的 final 成员等，常有条件直接调用。若调用经过协议要求、泛型或其他抽象，不能只看实现属于 struct 就断言这次调用一定静态派发。

类成员即使没有写 final，编译器也可能在某个调用位置证明目标唯一，进行去虚拟化优化。`static` 声明类型成员和“静态派发”是不同概念，不能靠有没有 static 关键字直接分类调用路径。

**虚方法表：**

可以把 Swift 类的虚方法表理解为按约定槽位保存可动态派发方法实现入口的表。父类和子类的相应槽位可以指向不同实现：

```text
Animal 的表：speak 槽位 → Animal.speak
Dog 的表：   speak 槽位 → Dog.speak
```

当调用需要保留类多态时，程序根据对象实际类型取得对应槽位的入口并调用。表里保存的是入口信息，不是全部方法源码。优化器在目标可证明时可以把这次表查找改成直接调用。Swift 虚方法表派发与 OC 按 selector 查找方法不是同一套机制。

## 编译阶段、机器码与内联

Swift 编译大致经过源码解析、类型检查、SIL 等中间表示、优化和目标代码生成，最后通过链接形成程序。机器码是 CPU 能执行的二进制指令，与 CPU 架构有关；十六进制和汇编是帮助人阅读的表示。

例如 ARM64 的普通 `ret` 指令编码可写成 `0xD65F03C0`；在小端内存中的四个字节是 `C0 03 5F D6`。这个例子只展示表示方式，不代表所有架构使用相同编码，也不代表所有实际返回序列只有这一条指令。

内联是编译器把被调用函数的逻辑放到调用位置附近，消除这次独立函数调用的开销，并为常量折叠等进一步优化创造条件。不是源文件里真的出现一次手工复制粘贴；过多内联也可能增大代码体积。静态派发更便于内联，但“能直接调用”不等于“必定内联”。

## 协议见证表与遵循关系

核心关系是“某个类型遵循某个协议”，而不是简单的一类型一表或一协议一表。见证表记录该遵循关系下，协议要求由什么实现满足。

```swift
protocol Speaker {
    func speak()
}

struct Dog: Speaker {
    func speak() { print("汪") }
}

struct Cat: Speaker {
    func speak() { print("喵") }
}
```

```text
Dog 遵循 Speaker：speak 要求 → 对应 Dog 实现的入口
Cat 遵循 Speaker：speak 要求 → 对应 Cat 实现的入口
```

同一类型遵循不同协议，对应不同的遵循信息；不同类型遵循同一协议也各有对应信息。实际还涉及泛型、继承和共享等实现细节，不必机械估算最终二进制中表的数量。

在泛型或协议存在类型上调用协议要求时，可以通过见证表找到实现；入口也可能是转接代码。泛型特化等优化可消除部分间接调用。这里只讨论声明在协议中的要求，不能把它直接推广为所有“仅写在协议 extension 里的方法”的派发规则。

## Objective-C 互操作、Selector 与 IMP

`@objc` 让符合 OC 表示要求的 Swift 声明暴露给 Objective-C。Selector 是 OC 方法的标识；`#selector(...)` 用编译器可检查的写法得到该标识，不会在这一步执行方法，也不是取得方法体。

UIKit 的 target-action API 保存目标和 selector，事件发生时经 OC 机制调用。例如按钮的回调方法标记 @objc 后即可提供对应入口；UIKit 这次 selector 调用本来就走 OC 消息派发，通常不需要再加 dynamic。

但仅有 @objc 不保证 Swift 自己直接访问该成员时也经过 OC 消息派发。在此处讨论的 @objc 成员上，dynamic 用来保证相应访问采用 OC 动态派发，供 KVO 等依赖运行时替换实现的机制使用。不能把所有普通回调都归因于 dynamic。

**selector 与方法实现：**

selector 是方法标识，不包含方法体，也不是 setter 的函数地址。IMP 是具体实现的函数入口。类的运行时方法信息建立两者的对应关系；同一个 selector 可以在不同类里对应不同实现。

```text
接收对象：person
方法标识：setAge:
参数：20
      ↓
isa 确定从哪个类开始找
      ↓
selector 确定找哪个方法
      ↓
找到对应 IMP，调用其实现
      ↓
setter 修改 age
```

“查找 setAge: → 执行 setter”是查找与执行两个阶段，不是在 setter 前又执行一个独立的 setAge 方法。Runtime 会用缓存加速查找，必要时沿父类查找。

| 查找起点 | selector | 对应实现 |
|-|-|-|
| Person | setAge: | 原 setter |
| KVO 动态子类 | setAge: | 带通知逻辑的 setter |

KVO 改变查找起点及对应实现，selector 始终可以是同一个 setAge:，不存在“往 selector 里面放入新的 setter”。

对于暴露给 OC、没有自定义访问器名称的普通 name 属性，getter 的 selector 是 `name`，setter 是 **`setName:`**。最后的冒号是名称的一部分，表示有一个显式参数，即要设置的新值；`setName` 与 `setName:` 是两个不同 selector。

## Selector 字符串与方法映射

`NSStringFromSelector` 取得 Selector 对应的 Objective-C 方法名字符串，不执行该方法，也不把方法体转换为字符串。以前文的[ProductModule](../cross-platform/webview-and-lynx-bridge.md#lynx-原生模块注册与回调)为例：

```swift
let selector = #selector(ProductModule.getProductName(_:callback:))
let methodName = NSStringFromSelector(selector)
// methodName 为 "getProductName:callback:"
```

Selector 名中的两个冒号对应两个参数。这里没有通过 `@objc(...)` 自定义方法名；若自定义了名称，应以暴露给 OC 的名称为准。

Lynx 的 `methodLookup` 使用 `[String: String]`，键是 JS 方法名，值是 OC 方法名，因此需要把 Selector 转成字符串。例如：`"getProductName" → "getProductName:callback:"`。`#selector` 让编译器检查目标方法是否存在且可用于 OC；相比手写整段字符串，方法变更时更容易得到编译期诊断。

## @objcMembers 与 Objective-C 兼容限制

**`@objcMembers` 是作用于类的 Swift 特性标注，让编译器为其中符合 Objective-C 表示要求的成员推断 `@objc`。** 它也会影响相关扩展和子类中的成员；不能把不兼容的声明自动变成 OC 可用声明，也不负责把类注册为 Lynx 模块。

在常见的 `NSObject` 子类中，属性类型、方法参数和返回类型需要能够用 OC 表示，声明形式也必须满足互操作要求。常见情况：

| 类型或声明 | 能否用于暴露给 OC 的普通成员 |
|-|-|
| `Int`、`Bool`、`Double` 等常用标量 | 可以，对应 OC 可表示的基础类型 |
| `String`、`String?` | 可以，通过与 `NSString` 的桥接表示 |
| `NSObject`、`UIView` 等 OC 可见的类及其可选引用 | 可以 |
| 元素、键和值均满足桥接要求的数组或字典 | 可以通过相应 Foundation 集合表示；不能推广到任意泛型集合 |
| 标记为 `@objc`、以受支持整数类型为原始值的枚举 | 可以；不能带 Swift 枚举关联值 |
| 满足 OC 限制并标记为 `@objc` 的协议类型 | 可以；纯 Swift 协议不能直接按此处理 |
| 参数和返回类型兼容的闭包 | 可以通过 Objective-C Block 表示；Block 是 OC 的可调用代码块对象 |
| 自定义 Swift `struct`、元组、带关联值的枚举 | 通常不能直接作为这类成员的类型 |
| 泛型方法，如 `func echo<T>(_ value: T)` | 不能直接导出为 OC 方法 |

示例：

```swift
import Foundation

struct ProductInfo {
    let name: String
}

@objcMembers
final class ProductBox: NSObject {
    var title: String = "运动鞋"       // 可以暴露给 OC
    var count: Int = 1                // 可以暴露给 OC

    func show(_ productId: String) {} // 可以暴露给 OC

    var info = ProductInfo(name: "运动鞋") // 自定义 struct，不自动暴露
    func echo<T>(_ value: T) {}           // 泛型方法，不自动暴露
}
```

`info`、`echo` 仍可被 Swift 使用。`@objcMembers` 会跳过不适用的成员；若在不兼容声明上显式添加 `@objc`，编译器会报错。

“Swift 的值类型一律不能暴露给 OC”不成立，`String` 等类型有既定桥接支持。另一方面，某个类型可桥接，不代表其任意组合形式都能导出；例如普通 `Int?` 属性不能像 `String?` 一样直接暴露为 OC 属性。应以实际声明是否满足编译器的互操作规则为准。

`@objcMembers` 也不等于 `dynamic`。对派发的影响见[@objc、selector 与 IMP](#objective-c-互操作selector-与-imp)。

## KVO 观察关系与属性访问

```swift
import Foundation

class Person: NSObject {
    @objc dynamic var age: Int = 18
}

let person = Person()
let observation = person.observe(\.age) { _, _ in
    print("收到了 age 的变化通知")
}

person.age = 20
observation.invalidate()
```

- NSObject 提供传统 Foundation KVO 所依赖的支持。
- @objc 暴露属性访问器，dynamic 让相关 Swift 访问走 OC 动态派发。
- observe 注册针对这个 person 实例的 age 的观察关系。
- setter 真正修改属性值，KVO 在相关路径安排通知处理。
- 返回的 observation 要在需要监听期间保持有效，实际页面中通常存为属性；invalidate 主动停止观察。

只写 @objc dynamic 不等于注册 KVO，也不会仅凭这两个标记就自动产生一次观察回调。这里讲的是传统 KVO，不是 Swift 的 Observation 宏机制。

## 自动 KVO 的动态子类机制

在支持自动通知的标准场景中，KVO 创建或复用运行时子类，安排被观察属性的 setter，并调整被观察对象的 isa，使消息查找从这个子类开始。KVO_Person 只是示意名称，不依赖具体私有类名。

```text
未注册观察：
person → isa → Person → 原 setter

注册观察后：
person → isa → KVO_Person → 带通知逻辑的 setter → Person 原 setter
```

还是同一个 person 对象，没有把原对象复制成一个新 person；变量的 Swift 声明类型仍然是 Person。变化的是运行时方法查找起点。注册观察针对具体对象，不是把所有 Person 实例统一换成 KVO 子类。

并非每调用一次 observe 都新建一个类，系统可以复用动态子类。也不能依赖注销观察后 isa 一定按某个私有时序恢复原值。

## KVO 通知选项与自动通知

下面是自动通知效果的伪代码，不是系统实际源码，也不是需要手写的 KVO 子类：

```text
KVO setter(newValue):
    willChangeValue(forKey: "age")
    调用父类原 setter(newValue)
    didChangeValue(forKey: "age")
```

前后处理用于组织变化通知，默认观察闭包在变化后收到通知，不因为内部有 will／did 两步就默认调用两次。注册时指定 .prior 才会额外要求变化前通知；.initial 则要求注册时立即提供初始通知。是否提供 old／new 值也由 options 控制。

上述说明以默认自动通知开启且属性符合 KVO 条件为前提。手动 KVO 可以由实现方自行在修改前后配对通知，不应在自动通知仍开启时不加区分地重复手动发送。

## dynamic 对 KVO 访问路径的影响

**有 @objc dynamic，但这个对象尚未注册任何 KVO 观察：** 仍走 OC 消息派发，isa 指向原类，查找 setAge: 后执行原 setter。dynamic 决定查找方式，不决定查到的一定是 KVO 的实现。对于本例的存储属性，原 setter 是编译器生成的；如果自己写了 setter，则执行自己写的原逻辑。

**去掉 dynamic、只保留 @objc：** Swift 访问不再被强制走 OC 消息派发，可以经 Swift 虚方法表调用，也可能在目标可确定时直接调用或内联。最终仍能读写属性存储；不经 OC Runtime 查找不等于无法访问属性。

```text
有 @objc dynamic 且注册观察：
Swift 赋值 → OC 消息派发 → KVO setter → 原 setter + 通知

只有 @objc 的一种可能路径：
Swift 赋值 → 直接调用／Swift 虚方法表 → 原 Swift setter
```

后一条路径可能绕过 KVO 在 OC 方法入口安排的拦截，出现“值改变但没有观察通知”。如果调用来自 OC 消息发送一侧，即使声明没有显式 dynamic，这次调用仍可走 OC Runtime。因此不能把“无 dynamic”与“所有场景均不经 OC”画等号。

还要区分“age 没观察”和“整个对象从未注册观察”：同一个对象的其他属性已被观察时，它可能已经使用 KVO 子类，不能仅凭 age 没观察就推断 isa 必然还是原类。
