# Xcode 工程管理与自动化测试

[返回笔记目录](../README.md)

## 本文目录

- [源码编辑与搜索](#源码编辑与搜索)
- [工程与构建](#工程与构建)
- [自动化测试](#自动化测试)

[界面预览与 Android 布局预览](#界面预览与-android-布局预览)

- [编译、链接、嵌入与装载](#编译链接嵌入与装载)
- [静态库的目标文件提取与代码裁剪](#静态库的目标文件提取与代码裁剪)
- [动态库的打包、启动装载与延迟装载](#动态库的打包启动装载与延迟装载)

## 源码编辑与搜索

### 源码编辑器的列选择

当每行的 `*` 位于同一列时，可以使用**列选择**：按矩形范围选中多行中相同列的字符，再一次删除。

1. 按住 **Option（⌥）**，从第一行 `*` 的左侧拖动鼠标，到最后一行 `*` 的右侧，选中整列 `*`。
2. 松开鼠标，**继续按住 Option**，确认选区只覆盖每行的 `*`。
3. 保持 Option 按下，按 **Delete** 删除选中的字符。
4. 删除完成后，再松开 Option。

如果还要删除 `*` 后面的一个空格，拖动时将这一列空格一起选中；不要选中后续代码。

**操作注意：删除过程中不能松开 Option 键，否则会选中一整段。Option 必须从开始选择一直按到删除完成。**

### 文本搜索与符号搜索

**文本搜索匹配文件中的字符；符号搜索查询代码索引中的符号名称及其引用。** 符号的定义和引用概念见[符号与引用](../language/swift-basics.md#符号引用与对象引用)。

| 对比项 | 文本搜索 | 符号搜索 |
|-|-|-|
| 搜索对象 | 当前搜索范围内的文件文字 | 代码索引识别的符号及其出现位置 |
| 注释、普通字符串内容 | 可以命中 | 不作为对目标符号的引用匹配 |
| 声明、定义、引用 | 只按字符匹配，不能据此区分这些角色 | 索引具有这些角色信息，可按搜索选项筛选 |
| 搜索 `actionHandler =` 等代码片段 | 适合 | 不适合，片段不是符号名称 |
| 索引依赖 | 不依赖代码符号索引 | 未纳入索引或索引未更新时，结果可能不完整或过时 |

`Find Selected Symbol in Workspace` 可以从编辑器中的符号发起搜索，但仍需核对 Find navigator 顶部的查询条件。**按名称搜索符号不等于只查一个已确定符号的全部引用。**`Containing` 是包含匹配，`Matching` 是完整名称匹配；大小写设置也会影响结果。同名但属于不同类型的属性，即使完整匹配也可能同时出现。

例如，按名称搜索 `actionHandler` 时，包含匹配可能同时选中 `actionHandler` 和 `ButtonActionHandler`；名称包含相同字符，不代表两个符号存在调用或赋值关系。完整名称匹配可以排除较长名称，但同名结果仍需结合所属类型判断。

### 沿赋值链查找闭包实现

下面声明一个闭包类型别名和一个可选变量：

```swift
typealias ActionHandler = () -> Void
var actionHandler: ActionHandler?
```

`actionHandler` 是保存闭包的可选变量。`ActionHandler` 是闭包类型的别名，不包含传入闭包的具体业务实现。要查看实际闭包体，应沿赋值位置查找：

1. 使用 Xcode 默认快捷键 `⌘⇧F` 打开工作区搜索，选择文本搜索并输入 `actionHandler =`，查找直接赋值位置；实际代码若使用不同空白写法或经参数传递赋值，需要继续沿数据来源查找。
2. 也可对 `actionHandler` 使用右键菜单 `Find → Find Selected Symbol in Workspace`，从符号使用位置中筛选赋值，并核对所属类型。
3. `⌃⌘J` 是 Jump to Definition。对属性名使用通常到属性声明，对类型别名使用到类型别名定义；不能因此直接确定运行时保存的是哪一个闭包。

给 `actionHandler` 赋值的最小示例：

```swift
actionHandler = {
    print("执行操作")
}
```

左侧 `actionHandler` 是保存闭包的变量，右侧是闭包值。执行赋值语句时，不执行闭包体；之后调用 `actionHandler?()` 时，闭包体才执行 `print`。变量、闭包值和闭包内的函数调用是不同概念。如果闭包存放在对象属性中，getter／setter 负责读写闭包值，不负责执行闭包体。

### 属性、访问器与索引记录

**P 表示属性，M 表示方法。** 实例属性的 getter 和 setter 是访问器方法；索引将访问器归类为方法时，可以显示 M。访问器的职责与编译器优化边界见[属性访问器](../language/swift-basics.md#存储属性计算属性与访问器)。

## 工程与构建

### Target、模块与构建产物

**Target 是构建单位：** 指定参加构建的代码、资源、依赖、设置和产物类型。**模块（Module）是代码的编译与访问边界：** 一组代码具有统一模块名，其他模块通过 `import` 使用其中对外可见的声明。**构建产物是生成的文件或包。** 三者不能互相等同。

| Target 或类型 | 产物例子 | 用途 |
|-|-|-|
| DemoApp App Target | `DemoApp.app` | 可执行程序、资源和配置等组成的 App |
| MathKit Framework Target | `MathKit.framework` | 承载库代码、模块接口等，供其他模块使用 |
| 单元测试 Target | `DemoTests.xctest` | 由测试运行器加载的测试包 |
| 资源 Bundle Target | `.bundle` | 可以只放图片、字体等，不包含 Swift 代码模块 |

Target 不都生成 App；构建产物也不都对应代码模块。Swift App 的代码也形成模块，但整个 `.app` 还包含资源等内容。普通 Swift Framework Target 通常对应一个模块，Target 名与模块名经常相同，也可以通过 `Product Module Name` 等配置调整。

**用 MathKit 理解“模块”：** 假设下面两个文件都属于名为 MathKit 的 Swift Framework Target：

```swift
// Add.swift，属于 MathKit 模块
public func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}

// Double.swift，属于同一个 MathKit 模块
public func double(_ value: Int) -> Int {
    return add(value, value)
}
```

两个文件里的代码归为同一模块，`Double.swift` 可直接使用模块内可见的 `add`。另一模块使用它们时：

```swift
import MathKit
let sum = add(2, 3)
let doubled = double(4)
```

MathKit 是模块名，Add.swift 是源文件名，add 是函数名；`public` 让函数可在模块外访问。仅创建同名目录不会自动形成模块，需要相应的构建配置。

### 构建设置与 Debug／Release

| 设置 | 含义 |
|-|-|
| Swift Language Version | 使用哪个 Swift 语言模式 |
| Optimization Level | 编译优化策略，例如 `-Onone` 关闭优化，方便调试 |
| Deployment Target | 支持的最低操作系统版本 |
| Framework／Header Search Paths | 查找依赖框架、头文件的目录 |
| Active Compilation Conditions | 条件编译标记，例如是否进入 `#if DEBUG` 分支 |
| Product Module Name | 生成的代码模块名称 |

这些可在 Target 的 `Build Settings` 查看。哪些文件要编译、哪些资源要复制、怎样链接依赖等任务则由 `Build Phases` 等配置组织，不要把所有页面都叫 Build Settings。

Build Configuration 是一组构建设置的名字，例如 Debug、Release。通常 Debug 偏向调试，Release 偏向运行效率，但实际行为由各项设置决定；名字本身不会自动保证某个优化值。

### podspec 与源码范围

`podspec` 是 CocoaPods 的库描述文件，使用 Ruby 语法编写。CocoaPods 是 iOS 常用的依赖管理工具。podspec 可以描述库的名称、版本、源码规则、依赖，以及支持的平台和最低系统版本。

下面用 `source_files` 指定源码范围：

```ruby
s.source_files = 'Sources/**/*.swift'
```

`s` 是 podspec 的配置对象。`Sources/**/*.swift` 匹配 `Sources` 目录及其子目录中的 Swift 文件。修改源码规则后，需要检查实际匹配到的文件集合。源码文件存在于磁盘中，不等于已经被构建配置纳入编译范围。

### Bazel、编译器与链接器

Bazel 是构建系统。Bazel 根据源码、依赖和构建配置等输入，决定需要执行哪些编译、链接任务，再调用相应工具完成任务。已有构建结果仍然有效时，Bazel 可以复用结果，减少重复构建。

**Bazel 本身不是 Swift 编译器。** Bazel 负责组织构建任务，Swift 编译器 `swiftc` 负责处理 Swift 源码并生成目标代码，链接器再将目标代码和所需库组合成相应产物。具体执行哪些任务取决于构建目标，索引任务不等于完整 App 构建。

为了方便理解，可以把 Bazel 类比为“施工调度员”，把编译器类比为“实际施工的工人”。这个类比只说明职责分工；编译的具体阶段见[编译器与机器码](../language/swift-objc-runtime.md#编译阶段机器码与内联)。

### 编译、链接、嵌入与装载

编译把源代码转换为目标代码。目标文件通常以 `.o` 为扩展名，保存代码、数据以及尚需链接器处理的符号信息。构建时链接把目标文件和库提供的定义组织成可执行文件或库，并处理代码中的引用关系。

App 包是包含主可执行文件、资源和配置等内容的目录。嵌入动态库，是把需要随 App 分发的动态库文件复制到 App 包。装载动态库，是在运行时把动态库映射到进程的地址空间，使进程能够访问其中的代码和数据；这不等于立即把文件的每个页面都读入物理内存。

“参与链接”“复制进 App 包”和“装载到进程”是不同动作。下面讨论常规静态链接和动态链接，不包含把可合并动态库重新合入主产物的特殊构建方式。

| 阶段 | 静态库 | 常规动态库 |
| --- | --- | --- |
| 构建时链接 | 链接器从静态库中取出所需目标文件，再把保留的代码和数据合入目标产物 | 链接器使用动态库的接口信息处理引用，并在产物中记录所需依赖；库的实现通常仍位于独立文件中 |
| App 打包 | 通常不需要把 `.a` 文件作为运行依赖复制进包 | 随 App 分发的第三方动态库需要正确嵌入；系统动态库由系统提供 |
| App 运行 | 相关实现已经属于使用该静态库的可执行文件或动态库 | dyld 准备运行所需动态库，并完成必要的地址关联和初始化 |

### 静态库的目标文件提取与代码裁剪

静态库 `.a` 通常是多个目标文件的归档。默认情况下，链接器根据尚未解析的符号引用，从归档中提取能提供所需定义的目标文件。因此，第一步通常以目标文件为单位，而不是直接按“业务调用了哪些函数”逐个挑选函数。

假设 `A.o` 包含 `foo()` 和 `bar()`，`B.o` 包含 `baz()`。应用只引用 `foo()`，且没有其他保留条件时，链接器通常会提取 `A.o`，不提取 `B.o`。`bar()` 是否最终保留，还要看后续优化和裁剪条件。

dead stripping 是链接阶段删除不需要保留的代码和数据的优化。链接器从程序入口、要求保留的定义等起点分析引用关系，再删除符合裁剪条件的部分。这种分析依据构建时可见的信息，不是根据一次实际运行中有没有调用某个函数。

`-all_load` 要求装入静态归档中的全部目标文件；`-force_load` 针对指定归档执行类似提取；`-ObjC` 会使包含 Objective-C 类或分类实现的目标文件被提取。这些选项改变目标文件的提取范围，并不自动意味着后续绝不进行代码裁剪。初始化代码、运行时元数据和显式保留规则也会影响最终结果。

因此，“静态库中未使用的内容可能不进入最终产物”成立；“凡是没有直接调用的函数都不会进入 App”不成立。需要区分目标文件是否被提取，以及被提取后哪些内容被保留。

### 动态库的打包、启动装载与延迟装载

普通动态库也参与构建时链接。构建时链接器检查所用接口并记录依赖，运行时 dyld 再根据这些信息准备实际动态库。不能把动态链接理解为构建阶段完全不处理动态库。

对于正确嵌入 App 的第三方动态库，库通常以独立二进制文件随 App 分发。链接 App 时，通常不会像处理静态归档那样，只从这个已经生成的动态库中抽取 App 调用的几个函数。因此，动态库中未被该 App 直接调用的代码，也可能随库文件进入 App 包。库自身在生成时仍可能执行优化和裁剪，不能据此推断所有源代码都一定保留。

系统提供的 UIKit 等动态框架不需要复制进 App 包。动态库是否作为依赖保留、是否嵌入 App 包，也受链接和打包配置影响；某些链接选项可以移除未使用的动态库依赖。仅把一个库加入工程，不能保证最终包内一定包含该库。

程序声明的普通必需动态库依赖，通常在启动阶段、进入 `main` 前就需要装载，并完成当时必需的准备工作。不会普遍等到第一次调用库中的函数时才装载整份库。应用如果使用 `dlopen` 显式请求装载一个尚未装载的库，才是另一种运行期间装载路径；能否装载还取决于库是否存在及平台的执行要求。

库的装载与符号绑定也不同。符号绑定是把函数或变量的引用关联到实际地址。某些绑定方式可以延迟到首次使用时完成，但具体行为取决于平台和构建方式。“延迟绑定某个函数”不等于“首次调用时才装载整个库”。

与启动过程的关系见 [main 之前的启动工作与 dyld](../ios/app-lifecycle-and-events.md#main-之前的启动工作与-dyld)。

## 界面预览与 Android 布局预览

界面预览是在开发工具中显示指定视图及其状态，帮助开发者检查布局。Xcode 支持界面预览，但具体页面能否直接预览，取决于界面实现方式、预览入口和项目依赖。

### Xcode 支持哪些方式

| 界面实现方式 | 可视化或预览能力 | 边界 |
| --- | --- | --- |
| Storyboard／XIB | Interface Builder 可以显示和编辑布局。 | 设计时布局不等于完整业务流程的运行结果。 |
| SwiftUI | 可以在 Xcode Canvas 中预览视图。 | 需要满足视图构造、数据和运行环境要求。 |
| UIKit 纯代码 | 支持通过 #Preview 预览 UIView 和 UIViewController。 | 代码需要能创建并显示目标对象，复杂页面可能依赖业务环境。 |

Canvas 是 Xcode 中展示预览的区域。Xcode 15 引入了直接支持 UIKit／AppKit 的 #Preview 入口。这不表示较早版本完全无法预览 UIKit；通过 SwiftUI 包装 UIKit 视图也有相应路径。

### UIKit 页面能够预览的条件

- 预览入口：提供创建目标 UIView 或 UIViewController 的代码，例如 #Preview。只写完一个业务控制器，不会自动得到可用的预览入口。
- 构建与运行环境：Xcode、SDK、所选预览运行环境和相关依赖需要相互兼容。预览需要准备可运行代码，不能理解为跳过编译直接显示任意源码。
- 视图和布局：目标对象能成功初始化，必要的资源、容器尺寸和布局条件可用。
- 数据和服务：页面需要的参数、显示数据以及实际依赖的服务可用。可以准备真实依赖，也可以注入示例数据或模拟服务；不是每个预览都必须额外建立一套模拟系统。

下面是一个不依赖业务服务的示例。SwiftUI 导入用于使用预览能力，返回的仍然是 UIKit 的 UILabel；不需要把 UILabel 重写成 SwiftUI。示例要求使用支持 UIKit #Preview 的 Xcode、SDK 和预览运行环境。

```swift
import UIKit
import SwiftUI

#Preview {
    let label = UILabel()
    label.text = "这是一条示例评论"
    label.numberOfLines = 0
    return label
}
```

预览 API 的可用性和 App 的最低部署版本是不同条件。例如 Xcode 15 的 UIKit #Preview 要求使用 iOS 17 或更高版本的预览运行环境。不能只看到项目支持 iOS 13，或者 Swift Language Version 写着 5.0，就断言项目无法使用预览。应以实际工具链和预览诊断为准。

Android 布局编辑器的示例数据，见[Android View、ViewGroup 与 XML 布局](../android/view-system-and-layout.md#android-布局编辑器的示例数据来自哪里)中的同名章节。

## 自动化测试

### 自动化测试与断言

自动化测试把“准备数据 → 执行操作 → 检查结果”写成程序，由工具重复执行。单元测试直接验证函数或状态逻辑；UI 自动化测试通过操作界面验证交互流程。AI 可以帮助生成用例和代码，是否通过仍由实际执行与断言决定。测试覆盖的场景有限，通过不表示所有场景都没有问题。

下面用“在数组中按 ID 查找评论”演示单元测试：数组包含 A、B、C，查找 B 应返回 B；查找不存在的 D 或查询空数组应返回 nil。示例数据由测试代码构造，不依赖网络接口：

```swift
import XCTest

final class CommentTests: XCTestCase {
    struct Comment {
        let id: String
        let text: String
    }

    func findComment(id: String, in comments: [Comment]) -> Comment? {
        comments.first { $0.id == id }
    }

    func testFindExistingComment() {
        let comments = [
            Comment(id: "A", text: "评论 A"),
            Comment(id: "B", text: "评论 B"),
            Comment(id: "C", text: "评论 C")
        ]
        let result = findComment(id: "B", in: comments)
        XCTAssertEqual(result?.id, "B")
    }

    func testMissingComment() {
        let result = findComment(id: "D", in: [Comment(id: "A", text: "评论 A")])
        XCTAssertNil(result)
    }

    func testEmptyList() {
        XCTAssertNil(findComment(id: "B", in: []))
    }
}
```

`XCTAssertEqual` 检查两值相等，`XCTAssertNil` 检查结果为空；不满足预期就报告失败。若误把查找实现改成无条件返回 `comments.first`，前两项会失败：查 B 却得到 A，查 D 也错误地得到 A。空列表用例仍可能通过，说明需要覆盖不同分支。

需要实际执行测试代码，但不必手动进入页面，也不用在 `main.swift`、AppDelegate 或页面里创建测试类并调用测试方法。启动测试后，XCTest 的测试运行器负责创建测试实例、调用符合规则的测试方法、收集断言结果。页面里手动构造数据也可验证逻辑，区别见[页面验证与独立验证](design-patterns.md#独立验证与页面验证)。

### XCTest 与 Swift Testing

| 对比 | XCTest | Swift Testing |
|-|-|-|
| 导入 | `import XCTest` | `import Testing` |
| 测试组织 | 类继承 `XCTestCase` | 可使用结构体、类或独立函数 |
| 标记测试 | 无参数、返回 Void，方法名以 `test` 开头 | 使用 `@Test` 标记函数 |
| 检查预期 | `XCTAssertEqual(a, b)`、`XCTAssertNil(value)` 等 | `#expect(a == b)`、`#expect(value == nil)` 等 |
| 主要适用范围 | 单元、集成、UI、性能测试 | 单元、集成测试 |

二者可以在同一个测试 Target 中共存。文件选项中的 `Language: Swift` 表示代码语言；继承 `XCTestCase`、使用 `XCTAssert...` 的示例仍属于 XCTest。

### 创建和运行 XCTestCase

1. 在 Xcode 的 `File → New → Target` 中选择对应平台的 `Unit Testing Bundle`。已有单元测试 Target 时直接复用。测试是否需要宿主，按[宿主与运行环境](#宿主-app运行环境与构建范围)判断，不能把关联 App 当成所有测试的固定步骤。
2. 用 `File → New → New File From Template → XCTest Unit Test` 创建文件。配置页 `Class` 填 `CommentTests`（不加 `.swift`），`Subclass of` 保持 `XCTestCase`，`Language` 选 Swift；类名填好后再点击 Next。
3. 保存时选择测试 Target。对已有文件，在 File Inspector 的 `Target Membership` 确认归属；仅放进叫 Tests 的文件夹不代表已经纳入测试构建。
4. 让一个 Scheme 的 Test 操作包含这个测试 Target，或关联包含它的测试计划；具体关系见[Scheme 与测试计划](#scheme-与测试计划)。
5. 选择适用的运行目的地。macOS 测试可以选择 `My Mac`；iOS 测试选择适用的设备或模拟器。点击测试方法左侧菱形里的播放按钮运行单项；`Product → Test` 执行当前配置选中的测试。运行器会编译所需代码并执行测试，绿色／红色表示通过／失败，失败时查看对应断言信息。

被测代码若和测试文件一起编译进测试 Target，可以直接调用。若来自另外一个 Swift 模块，则导入实际模块名；访问其 internal 成员时，需要该模块启用 `Enable Testability`，并使用 `@testable import 模块名`。`@testable` 不会自动让任意 private 成员可见，也不能替代目标依赖和链接配置。独立算法测试示例直接包含原算法源文件，无需导入命令行可执行模块。

### Scheme 与测试计划

Scheme 是构建、运行、测试等操作的执行方案。Scheme 和 Target 是不同概念，即使名称相同也不能互相替代。

| Scheme 中的配置 | 作用或例子 |
|-|-|
| Build 中的 Target 列表及操作勾选 | 指定哪些 Target 参与 Run、Test、Archive 等操作 |
| 各操作的 Build Configuration | 例如 Test 使用 Debug，采用各 Target 对应的构建设置 |
| Run 的 Executable | 例如构建后启动 DemoApp.app |
| 调试器、启动参数和环境变量 | 是否连接调试器，以及向运行进程传入哪些值 |
| Test 中的测试目标或测试计划 | 决定测试操作要执行哪些测试 |

测试计划通常保存为 `.xctestplan`，集中选择测试 Target、测试类、方法，并保存一套或多套测试配置。其常见选项包括：

| 测试计划选项 | 用途 |
|-|-|
| 语言、地区 | 验证中文／英文等环境下的表现 |
| 启动参数、环境变量 | 向被测进程传值；自定义值需要相应代码读取和处理，不会自动模拟网络错误等行为 |
| 执行顺序、重复方式、超时 | 检查顺序依赖、偶发问题和超时 |
| 诊断检查 | 例如 Address Sanitizer、Thread Sanitizer、Main Thread Checker |
| 代码覆盖率 | 记录测试实际执行过的代码范围 |

例如同一套界面测试可在“中文＋中国地区”和“英文＋美国地区”两套配置下分别执行。测试计划配置与 Debug／Release 构建配置不同；部分运行参数也能出现在 Scheme 中。测试计划不是必需，Scheme 可以直接列出测试 Target。

### 宿主 App、运行环境与构建范围

**宿主 App** 是承载单元测试运行的 App。使用宿主的 XCTest 单元测试，会先准备并启动宿主 App，再将测试包加载到它的进程里，由 XCTest 执行测试；仍不需要自己在 App 入口创建测试类。支持无宿主的测试配置则由系统测试运行器承载，例如独立的 macOS 算法测试。这一说明针对单元测试，不能直接套到 UI 自动化测试的进程关系上。

**运行环境** 是代码执行时已经具备的系统能力、服务和状态。例如“加载当前用户评论”可能依赖登录用户 ID、配置好的网络服务、已打开的缓存数据库。App 启动可能初始化其中一些服务，测试也可以自行初始化或注入模拟对象；是否需要宿主，取决于真实依赖和平台／测试工具支持。

`twoSum([2, 7, 11, 15], 9)` 只依赖输入值；前面的评论查找示例也使用测试自己构造的数组。因此这两个例子都没有逻辑上必须依赖宿主 App 的理由。**设置宿主不等于提供真实评论数据。** 数据从哪里来由测试代码决定，可以是构造值、文件、Mock 或真实接口。

点击某个测试方法的菱形只限定执行哪项测试，构建仍需准备测试 Target、所需依赖，以及 Scheme 中为 Test 启用的其他目标；不能理解成只编译被点击的函数。
