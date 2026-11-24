# Swift Codable 与序列化复制

[返回笔记目录](../README.md)

本文说明 Codable 编码与解码、CodingKeys 的作用，以及用编码再解码的方式复制对象时会出现的问题。

## 本文目录

- [Codable 与序列化](#codable-与序列化)
  - [CodingKeys 的作用与枚举语法](#codingkeys-的作用与枚举语法)
- [序列化复制的边界](#序列化复制的边界)
  - [未编码的属性不会保留原值](#未编码的属性不会保留原值)
  - [自定义解码可能重新使用共享对象](#自定义解码可能重新使用共享对象)
  - [原来共享的对象可能被拆成两个对象](#原来共享的对象可能被拆成两个对象)
  - [循环引用需要显式记录对象关系](#循环引用需要显式记录对象关系)

## Codable 与序列化

序列化把模型中需要保存的状态转换成可存储、可传输的数据。反序列化读取这些数据，并据此构建模型。JSON 是表达数据的文本格式；Foundation 的 Data 用于保存字节数据。JSONEncoder 将模型编码为包含 JSON 的 Data，JSONDecoder 从这份 Data 解码出指定类型的模型。

| 协议 | 表达的能力 |
| --- | --- |
| Encodable | 类型能通过 encode(to:) 把状态交给编码器 |
| Decodable | 类型能通过 init(from:) 从解码器读取数据并初始化实例 |
| Codable | 同时遵循 Encodable 和 Decodable |

Codable 是类型别名，等价于协议组合 Encodable & Decodable。只需要反序列化时，遵循 Decodable 即可。如果类型的所有存储属性都遵循 Codable，编译器会自动生成 encode(to:) 和 init(from:)；有属性不遵循 Codable，或需要自定义字段规则时，要自己实现这两个方法。Codable 只是协议，真正执行编码的是 JSONEncoder 等编码器。

下面的示例使用 Address 和 Person 两个类。两个类的所有存储属性都参与编码和解码，使用编译器自动生成的实现。

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

深拷贝是复制对象时，把对象引用的其他对象也一并复制，使副本与原对象互不共享可变对象；浅拷贝与深拷贝的区别见[引用赋值、浅拷贝与深拷贝](swift-memory-and-references.md#引用赋值浅拷贝与深拷贝)。

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

下面的 AddressPair 使用上文的 Address。home 和 mailing 原来指向同一个实例。=== 比较的是实例身份，不是城市内容是否相同。

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
