# iOS 数据持久化

[返回笔记目录](../README.md)

## 持久化与常用方式

数据持久化是将数据保存到应用进程结束后仍能保留的存储介质中，使应用下次启动时可以重新读取。只把数据放在变量、数组或单例中，不属于持久化。持久化也不等于永久保留；数据能保留多久，还取决于存储位置和清理规则。

iOS 中使用 Swift 开发时，常见方式包括 UserDefaults、文件存储、Keychain、SQLite、Core Data 和 SwiftData。

| 方式 | 保存机制 | 适用场景 |
| --- | --- | --- |
| UserDefaults | 按键值保存设置，通过字符串键定位对应的值 | 少量、非敏感的偏好设置，例如主题和是否显示过引导；不适合保存密码或大量业务记录 |
| 文件存储 | 将数据写入应用可访问的文件；通常由 FileManager 管理目录和文件，由 Data 等类型读写内容 | 图片、视频、文档，以及保存为 JSON、plist 等格式的数据 |
| Keychain | 使用系统提供的加密存储，并由系统控制访问权限 | 密码、登录凭证、密钥等少量敏感数据 |
| SQLite | 将数据组织成表，使用 SQL 定义、查询和修改数据；SQL 是用于操作数据库的语言 | 需要按条件查询、排序、统计或频繁修改的记录，例如消息记录 |
| Core Data | 管理对象、对象之间的关系及对象的变化，并将对象数据保存到持久化存储 | 具有多个关联数据类型的应用，例如笔记与笔记分类；通常直接操作对象，无须自己编写 SQL |
| SwiftData | 使用 Swift 类型定义数据模型，通过框架保存和查询模型对象 | 使用 Swift 模型管理数据，并与 SwiftUI 集成；框架从 iOS 17 起可用 |

数据模型是描述应用保存哪些数据以及这些数据如何关联的类型和规则。例如，笔记有标题、正文和所属分类，分类可以关联多篇笔记。

## UserDefaults 的最小示例

下面通过键 hasSeenGuide 保存“用户是否看过引导”。键是用于定位数据的名称，true 是保存的值。

```swift
import Foundation

// 保存
UserDefaults.standard.set(true, forKey: "hasSeenGuide")

// 读取
let hasSeenGuide = UserDefaults.standard.bool(forKey: "hasSeenGuide")

// 删除
UserDefaults.standard.removeObject(forKey: "hasSeenGuide")
```

调用 set 后，UserDefaults 立即更新内存中的值，再异步写入磁盘。调用 bool(forKey:) 时，如果对应键不存在，方法返回 false。因此，这个读取方法不能区分“未保存过”和“明确保存了 false”。

SwiftUI 的 @AppStorage 使用 UserDefaults 读写设置，并在对应值变化时更新相关视图。@AppStorage 没有提供独立于 UserDefaults 的另一套存储。

## 编码、文件格式与持久化的区别

Codable 表达类型支持编码和解码的能力，本身不负责把数据写入磁盘。编码与解码的具体机制见[Swift 语言基础](../language/swift-basics.md)。

以 JSON 文件为例，保存流程是：Swift 对象 → JSONEncoder 转换为 Data → 将 Data 写入文件。读取流程是：从文件读取 Data → JSONDecoder 根据指定类型恢复 Swift 对象。Data 表示字节数据；只完成编码时，编码结果仍可能只存在于内存中。

JSON 和 plist 是数据格式，不是独立的存储服务。选择 JSON 或 plist，决定数据如何表示；选择文件、数据库等存储，决定数据保存在哪里以及如何读写。

文件存储还需要选择合适的目录。目录用途、系统清理和备份规则见[iOS 文件存储](ios-file-storage.md)。

## Core Data 与 SQLite 的关系

Core Data 负责管理对象及其保存过程。SQLite 负责数据库存储和查询。Core Data 可以使用 SQLite 作为保存数据的底层存储，但 Core Data 不等同于 SQLite。

例如，使用 SQLite 时，应用可以通过 SQL 查询某个分类下的笔记。使用 Core Data 时，应用通常通过框架查询笔记对象，并访问笔记与分类的关系。Core Data 负责将这些对象数据映射到配置的持久化存储。

使用 Core Data 或 SwiftData 管理了对象，不代表所有配置都会写入磁盘。需要跨应用启动保留数据时，应使用持久化存储，并确保保存成功；只保存在内存中的配置不能满足这个要求。
