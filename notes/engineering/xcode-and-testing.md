# Xcode 工程管理与自动化测试

[返回笔记目录](../README.md)

## 本文目录

- [源码编辑与搜索](#源码编辑与搜索)
- [工程与构建](#工程与构建)
- [自动化测试](#自动化测试)

## 源码编辑与搜索

### 源码编辑器的列选择

当每行的 `*` 位于同一列时，可以使用**列选择**：按矩形范围选中多行中相同列的字符，再一次删除。

1. 按住 **Option（⌥）**，从第一行 `*` 的左侧拖动鼠标，到最后一行 `*` 的右侧，选中整列 `*`。
2. 松开鼠标，**继续按住 Option**，确认选区只覆盖每行的 `*`。
3. 保持 Option 按下，按 **Delete** 删除选中的字符。
4. 删除完成后，再松开 Option。

如果还要删除 `*` 后面的一个空格，拖动时将这一列空格一起选中；不要选中后续代码。

**操作注意（按当前使用反馈记录）：删除过程中不能松开 Option 键，否则会选中一整段。Option 必须从开始选择一直按到删除完成。**

列选择快捷操作参考 [Apple：Xcode 源码编辑器说明](https://developer.apple.com/documentation/xcode-release-notes/source-editor-release-notes-for-xcode-10)。

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

2026-10-05 读取的项目搜索界面为 `Symbols + Containing`，搜索词是 `actionHandler`，关闭 `Aa`（不区分大小写），勾选 `Include References`。在这组条件下，可以理解为按名称选出匹配符号，并同时列出匹配符号的引用。因此，类型别名 `AuthorityAlertActionHandler` 也命中；名称中包含 `ActionHandler`，不代表它与 `XTIFCellViewModel.actionHandler` 有实现关系。改为完整匹配可以排除这类较长名称，但仍需结合所属类型判断同名结果。

### 沿赋值链查找闭包实现

下面是 `actionHandler` 的属性声明：

```swift
public var actionHandler: XTIFBottomBar.BottomBarActionHandler?
```

`actionHandler` 是保存闭包的可选属性。`BottomBarActionHandler` 是闭包类型的别名，不包含传入闭包的具体业务实现。要查看实际闭包体，应沿赋值位置查找：

1. 使用 Xcode 默认快捷键 `⌘⇧F` 打开工作区搜索，选择文本搜索并输入 `actionHandler =`，查找直接赋值位置；实际代码若使用不同空白写法或经参数传递赋值，需要继续沿数据来源查找。
2. 也可对 `actionHandler` 使用右键菜单 `Find → Find Selected Symbol in Workspace`，从符号使用位置中筛选赋值，并核对所属类型。
3. `⌃⌘J` 是 Jump to Definition。对属性名使用通常到属性声明，对类型别名使用到类型别名定义；不能因此直接确定运行时保存的是哪一个闭包。

项目中的两处赋值采用下面的形式：

```swift
actionHandler = { [weak self] in
    self?.handleBottomBarAction(rawItem: rawItem, item: $0, action: $1)
}
```

左侧 `actionHandler` 是属性，右侧是保存到属性中的闭包。执行这条赋值语句时，不执行闭包体；之后调用保存的闭包时，闭包体才尝试调用 `handleBottomBarAction` 方法。属性、闭包值和闭包内的方法调用是不同概念。getter／setter 负责读写闭包值，不是闭包中的业务实现。

### 属性、访问器与索引记录

**P 表示属性，M 表示方法。** 原始资料核查的实例属性访问器在索引中属于方法，因此 getter 和 setter 都可以显示 M。访问器的职责与编译器优化边界见[属性访问器](../language/swift-basics.md#存储属性计算属性与访问器)。

## 工程与构建

### Target、模块与构建产物

**Target 是构建单位：** 指定参加构建的代码、资源、依赖、设置和产物类型。**模块（Module）是代码的编译与访问边界：** 一组代码具有统一模块名，其他模块通过 `import` 使用其中对外可见的声明。**构建产物是生成的文件或包。** 三者不能互相等同。

| Target 或类型 | 产物例子 | 用途 |
|-|-|-|
| Retouch App Target | `Retouch.app` | 可执行程序、资源和配置等组成的 App |
| RTSDKModel Framework Target | `RTSDKModel.framework` | 承载库代码、模块接口等，供其他模块使用 |
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

项目例子中，`Statistics.swift` 是 RTSDKModel 的源文件，其中声明 `StatisticsExtraCommonFlag`。2026-10-02 已核对 JoJo 外部工程：Target 的 `productType` 是 `com.apple.product-type.framework`，声明产物名为 `RTSDKModel.framework`。这说明它是 Framework Target，不是额外的 RTSDKModel App；该配置核查不等于已重新构建或确认磁盘产物。

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

Build Configuration 是一组构建设置的名字，例如 Debug、Release，当前醒图 Scheme 使用过 RetouchDebug。通常 Debug 偏向调试，Release 偏向运行效率，但实际行为由各项设置决定；名字本身不会自动保证某个优化值。2026-10-02 核对到 RTSDKModel 的 RetouchDebug 配置中 `SWIFT_OPTIMIZATION_LEVEL = -Onone`。

### JoJo 与 Xmate 的职责

JoJo 是当前醒图项目使用的构建与工程管理工具链。JoJo 处理模块配置、准备构建输入，并调用 [Bazel](#bazel编译器与链接器) 执行编译或索引构建。

Xmate 提供代码编辑、工程目录展示和跳转等 IDE 功能。在当前项目的外部模块工程生成链路中，Xmate 读取 JoJo 准备的配置，再生成供 IDE 展示的工程结构。JoJo 与 Xmate 的职责不同；源码文件真实存在，不代表 Xmate 已经把文件纳入工程目录。

### podspec 与生成配置的更新链路

`podspec` 是 CocoaPods 的模块描述文件，使用 Ruby 语法编写。CocoaPods 是 iOS 项目常用的依赖管理工具。`XTCommunity.podspec` 描述模块名称、版本、源码规则、依赖，以及支持的平台和最低系统版本。

以 `Retouch` subspec（子规格）中的源码规则为例，下面只摘取一项配置：

```ruby
ss.source_files = ['PostDetail/**/*']
```

`ss` 是该 subspec 的配置对象。这条规则用于匹配 `PostDetail` 目录及其子目录中的文件。原规则 `**/*` 没有固定目录前缀；改成明确目录列表时，仍需要核对实际匹配的文件集合，不能只凭写法判断文件范围不变。

在当前项目中，JoJo 调用 CocoaPods 解析原始 podspec，将模块描述转换成 JSON。Xmate 读取生成的 JSON 来构造外部模块工程。JSON 是结构化数据格式，不是 Swift 源码。

**配置传递顺序：原始 podspec → JoJo 调用 CocoaPods 生成 JSON → Xmate 读取 JSON 生成工程目录。**

“旧 JSON”指生成文件中尚未同步原始 podspec 修改的内容。例如，当原始 podspec 已改成明确目录列表，而生成文件的 `Retouch` subspec 仍包含下面的字段时，Xmate 读取到的仍是旧规则：

```json
"source_files": ["**/*"]
```

上面是 JSON 字段片段，不是完整 JSON 文件。原始配置已修改、生成 JSON 已更新、Xmate 已读取新配置是三个不同状态，不能互相替代。

原始资料记录的 2026-10-04 案例中，Xmate 在 21:58:32 生成工程时仍使用旧 JSON；之后 `index_build` 调用 `bundle exec pod ipc spec`，JSON 在 21:59:38 更新。该记录不能证明 Reload Project 会先解析原始 podspec，也没有证明源码目录与方法跳转已恢复。

### Bazel、编译器与链接器

Bazel 是构建系统。Bazel 根据源码、依赖和构建配置等输入，决定需要执行哪些编译、链接任务，再调用相应工具完成任务。已有构建结果仍然有效时，Bazel 可以复用结果，减少重复构建。

**Bazel 本身不是 Swift 编译器。** Bazel 负责组织构建任务，Swift 编译器 `swiftc` 负责处理 Swift 源码并生成目标代码，链接器再将目标代码和所需库组合成相应产物。具体执行哪些任务取决于构建目标，索引任务不等于完整 App 构建。

在当前项目中，可以简化为：**JoJo 发起构建 → Bazel 安排任务 → `swiftc`、链接器等工具执行具体任务。**

为了方便理解，可以把 Bazel 类比为“施工调度员”，把编译器类比为“实际施工的工人”。这个类比只说明职责分工；编译的具体阶段见[编译器与机器码](../language/swift-objc-runtime.md#编译阶段机器码与内联)。

## 自动化测试

### 自动化测试与断言

自动化测试把“准备数据 → 执行操作 → 检查结果”写成程序，由工具重复执行。单元测试直接验证函数或状态逻辑；UI 自动化测试通过操作界面验证交互流程。AI 可以帮助生成用例和代码，是否通过仍由实际执行与断言决定。测试覆盖的场景有限，通过不表示所有场景都没有问题。

以 Push 评论直达中的“按 ID 找评论”为示例：列表有 A、B、C，查 B 应返回 B；查不存在的 D、查询空列表应返回 nil。以下模型和函数仅用于教学，不是醒图真实评论链路，也不请求真实评论数据：

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
5. 选择适用的运行目的地。AlgorithmPractice 示例 是 macOS 工程，选择 `My Mac`。点击测试方法左侧菱形里的播放按钮运行单项；`Product → Test` 执行当前配置选中的测试。运行器会编译所需代码并执行测试，绿色／红色表示通过／失败，失败时查看对应断言信息。

被测代码若和测试文件一起编译进测试 Target，可以直接调用。若来自另外一个 Swift 模块，则导入实际模块名；访问其 internal 成员时，需要该模块启用 `Enable Testability`，并使用 `@testable import 模块名`。`@testable` 不会自动让任意 private 成员可见，也不能替代目标依赖和链接配置。独立算法测试示例直接包含原算法源文件，无需导入命令行可执行模块。

### Scheme 与测试计划

Scheme 是构建、运行、测试等操作的执行方案。Scheme 和 Target 是不同概念，即使名称相同也不能互相替代。

| Scheme 中的配置 | 作用或例子 |
|-|-|
| Build 中的 Target 列表及操作勾选 | 指定哪些 Target 参与 Run、Test、Archive 等操作 |
| 各操作的 Build Configuration | 例如 Test 使用 Debug 或 RetouchDebug，采用各 Target 对应的构建设置 |
| Run 的 Executable | 例如构建后启动 Retouch.app |
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

例如同一套界面测试可在“中文＋中国地区”和“英文＋美国地区”两套配置下分别执行。测试计划配置与 Debug／Release 构建配置不同；部分运行参数也能出现在 Scheme 中。测试计划不是必需，Scheme 可以直接列出测试 Target；AlgorithmPractice 示例Tests 使用直接关联方式。

### 宿主 App、运行环境与构建范围

**宿主 App** 是承载单元测试运行的 App。使用宿主的 XCTest 单元测试，会先准备并启动宿主 App，再将测试包加载到它的进程里，由 XCTest 执行测试；仍不需要自己在 App 入口创建测试类。支持无宿主的测试配置则由系统测试运行器承载，例如独立的 macOS 算法测试。这一说明针对单元测试，不能直接套到 UI 自动化测试的进程关系上。

**运行环境** 是代码执行时已经具备的系统能力、服务和状态。例如“加载当前用户评论”可能依赖登录用户 ID、配置好的网络服务、已打开的缓存数据库。App 启动可能初始化其中一些服务，测试也可以自行初始化或注入模拟对象；是否需要宿主，取决于真实依赖和平台／测试工具支持。

`twoSum([2, 7, 11, 15], 9)` 只依赖输入值；前面的评论查找示例也使用测试自己构造的数组。因此这两个例子都没有逻辑上必须依赖醒图 App 的理由。**设置宿主不等于提供真实评论数据。** 数据从哪里来由测试代码决定，可以是构造值、文件、Mock 或真实接口。

点击某个测试方法的菱形只限定执行哪项测试，构建仍需准备测试 Target、所需依赖，以及 Scheme 中为 Test 启用的其他目标；不能理解成只编译被点击的函数。2026-10-02 检查时，DemoTests 的 `TEST_HOST` 指向 Retouch.app，`BUNDLE_LOADER` 引用该宿主，所以会涉及醒图 App 及其依赖。该配置说明构建范围较大；未取得构建耗时报告，不能断言具体哪个阶段最慢或量化提速。
