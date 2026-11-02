# CocoaPods、Podfile 与依赖管理

[返回笔记目录](../README.md)

## 本文目录

- [工具与配置文件的职责](#工具与配置文件的职责)
- [Podfile 的 Ruby 代码与对象上下文](#podfile-的-ruby-代码与对象上下文)
- [依赖声明与版本范围](#依赖声明与版本范围)
- [Molinillo 解析依赖与 Xcodeproj 集成工程](#molinillo-解析依赖与-xcodeproj-集成工程)
- [Podfile.lock 与安装、更新](#podfilelock-与安装更新)
- [YAML 与锁文件中的数据](#yaml-与锁文件中的数据)
- [Gemfile 与 Ruby 工具版本](#gemfile-与-ruby-工具版本)
- [用 Python 读取 Podfile](#用-python-读取-podfile)

## 工具与配置文件的职责

CocoaPods 是用于管理 iOS 等 Apple 平台项目依赖的工具，主要使用 Ruby 实现。依赖是项目或某个库需要使用的其他库；应用在 Podfile 中直接声明的库是直接依赖，这些库继续需要的其他库是间接依赖。CocoaPods 负责确定依赖版本、获取依赖并配置 Xcode 工程。

| 文件 | 管理的内容 |
| --- | --- |
| `Podfile` | 应用各个 target 需要哪些 pod，以及允许的版本、来源和集成配置 |
| `Podfile.lock` | CocoaPods 已确定的依赖版本及相关记录 |
| `Gemfile` | 当前开发环境需要哪些 Ruby 软件包及版本要求，例如 CocoaPods 工具 |
| `Gemfile.lock` | Bundler 已确定的 Ruby 软件包版本及相关记录 |

target 是 Xcode 中描述一个构建目标及其配置的单位，例如一个应用或测试目标。Podfile 中的 target 声明用于组织该目标的依赖配置。库作者还通过 podspec 描述库的源码范围、依赖和平台要求，具体见[podspec 与源码范围](xcode-and-testing.md#podspec-与源码范围)。

## Podfile 的 Ruby 代码与对象上下文

Podfile 是可执行的 Ruby 代码。CocoaPods 在 Ruby 上提供了面向依赖配置的专用写法，称为 DSL（领域专用语言）：通过 `target`、`pod` 等方法表达配置，同时仍可以使用 Ruby 变量、条件判断和方法调用。

```ruby
library_name = 'Alamofire'

target 'Demo' do
  if ENV['USE_NETWORK'] == '1'
    pod library_name, '~> 5.9'
  end
end
```

这段代码先把库名保存到 `library_name`。执行 `target 'Demo'` 时，CocoaPods 建立或进入 Demo 对应的依赖配置。随后代码读取进程环境变量 `USE_NETWORK`；只有值恰好为字符串 `'1'` 时，才声明 Demo 使用 Alamofire。变量未设置或值为其他字符串时，本段代码不添加这条声明；Demo 是否还依赖 Alamofire，仍要看其他声明和间接依赖。

`do ... end` 包围的是 Ruby 代码块，调用方把代码块交给 `target` 方法执行。`ENV` 提供当前进程的环境变量。在这个例子中，修改环境变量会改变 Podfile 执行后得到的依赖配置。

`Pod::Podfile` 是 CocoaPods 用来表示整份 Podfile 配置的 Ruby 类，`Pod` 是命名空间，`Podfile` 是类名。CocoaPods 加载一份 Podfile 时，会创建对应的 `Pod::Podfile` 对象，并在该对象作为 `self` 的上下文中执行文件内容。`self` 表示当前方法调用默认作用的对象，因此省略接收者的 `target` 和 `pod` 调用能够找到 CocoaPods 提供的方法。

进入 `target` 的代码块，不表示为每个 pod 创建一个新的 `Pod::Podfile` 对象。CocoaPods 在整份配置对象中维护各个目标的配置对象 `TargetDefinition`，并把当前执行的 `pod` 声明记录到当前目标。每次 `pod` 调用增加的是依赖声明，不是另一份 Podfile。

```text
一个 Podfile 对应的配置对象
└── Demo 的目标配置
    └── Alamofire 的依赖声明（条件成立时）
```

## 依赖声明与版本范围

`pod library_name, '~> 5.9'` 的作用是记录“当前目标需要 Alamofire，版本应满足这个范围”。执行这一行不等于立即下载 Alamofire，也不意味着所有符合范围的版本都会被安装。CocoaPods 后续要结合其他依赖要求确定一个可用版本。

`~>` 是版本要求运算符。上界由运算符和后面版本号的位数共同决定：

| 写法 | 允许范围 | 示例 |
| --- | --- | --- |
| `'~> 5.9'` | `>= 5.9` 且 `< 6.0` | 允许 `5.9.1`、`5.10.0`，不允许 `6.0.0` |
| `'~> 5.9.0'` | `>= 5.9.0` 且 `< 5.10.0` | 允许 `5.9.1`，不允许 `5.10.0` |
| `'5.9.1'` | 固定为 `5.9.1` | 不自动改为其他版本 |

这里以普通正式版本说明范围，不展开预发布版本的选择规则。`'~> 5.9'` 没有单独写出 `< 6.0`，但 Ruby 的版本要求规则已经通过 `~>` 和 `5.9` 两段版本号表达了这个上界。

## Molinillo 解析依赖与 Xcodeproj 集成工程

依赖解析是从各个库的可用版本中，找到一组同时满足版本范围、平台要求和依赖关系的版本组合。CocoaPods 使用 Molinillo 提供的解析能力，并向解析器提供候选版本、依赖关系和锁定要求。

假设库 A 要求公共依赖 C 的版本 `>= 1.0` 且 `< 3.0`，库 B 要求 C 的版本 `>= 2.0` 且 `< 4.0`。两个要求的交集是 `>= 2.0` 且 `< 3.0`。如果存在 C 2.5，且其他条件都满足，2.5 就可以成为候选版本。若 A 要求 `< 2.0`，B 又要求 `>= 2.0`，则这两个要求没有共同版本。

Molinillo 会尝试候选版本，并继续检查候选版本引入的依赖。发现冲突时，解析器可以撤销先前的版本选择，尝试其他候选版本；这称为回溯。找不到满足全部要求的组合时，CocoaPods 报告依赖冲突。仅检查每一条 `pod` 声明自身是否合法，不能代替整组依赖解析。

解析结果需要进一步接入 Xcode。Xcodeproj 是用于读取、创建和修改 Xcode 工程等文件的 Ruby 库。CocoaPods 决定应生成哪些目标、文件引用和构建配置，再通过 Xcodeproj 等组件写入工程文件。Xcodeproj 本身不负责从版本范围中选择 pod 版本。

常规 `pod install` 流程可以概括为：

1. CocoaPods 执行 Podfile，读取已有锁文件，收集依赖描述和配置。
2. CocoaPods 解析版本及依赖关系，获取需要的依赖源码或其他交付文件。
3. CocoaPods 生成依赖工程、构建配置和必要的集成脚本，并把这些配置接入用户工程。
4. CocoaPods 写入锁文件等安装结果。

这不是“依赖解析完成前绝不会下载任何文件”的保证。对于某些外部来源，CocoaPods 可能先获取文件，以取得解析所需的 podspec；缓存也可能使部分依赖无需重新下载。

默认单个 Pods 工程配置下，常见产物包括 `Pods/Pods.xcodeproj`、保存构建参数的 `.xcconfig` 文件，以及把用户工程和 Pods 工程组织在一起的 `.xcworkspace`。CocoaPods 还会配置用户工程对依赖的引用及必要的链接、资源处理等步骤。不同选项下产物结构可以不同。`pod install` 完成依赖安装与工程集成后，应用的实际编译和链接仍由后续 Xcode 构建执行。

## Podfile.lock 与安装、更新

Podfile.lock 记录 CocoaPods 已经确定的依赖版本，使后续安装能够沿用同一组版本。Podfile 表示允许选择什么，Podfile.lock 表示已经选定什么。

例如 Podfile 声明 `'~> 5.9'`，一次成功安装选中了 Alamofire 5.9.1。即使之后出现符合范围的 5.9.2，在保留锁文件、声明仍然兼容且没有请求更新该依赖的条件下，再次执行 `pod install` 会沿用锁定的 5.9.1。

团队将 Podfile 和 Podfile.lock 一起纳入版本控制后，其他成员可以依据相同锁文件安装依赖。锁文件还记录间接依赖等信息，作用不只限于 Podfile 中直接出现的库。

| 命令 | 主要用途 |
| --- | --- |
| `pod install` | 根据当前 Podfile 安装依赖，并尽量沿用现有锁定版本；新增或变化的要求可能需要重新解析 |
| `pod update Alamofire` | 请求重新选择 Alamofire 的版本，并按依赖关系更新必要的相关依赖和锁文件 |
| `pod update` | 请求更新全部依赖，不继续把原有版本锁定作为更新限制 |

更新仍受 Podfile 的版本要求等条件限制。`pod update Alamofire` 不表示可以忽略 `'~> 5.9'` 而直接升级到 6.0。锁文件也不是完整运行环境的快照；Ruby 工具版本、系统环境或可变的本地依赖等条件仍可能影响结果。

修改 Podfile 后，已有 Podfile.lock 在下一次成功安装前可能仍代表旧结果。读取锁文件只能说明其中记录了什么，不能单凭锁文件断言当前工作目录已完成相应安装。

## YAML 与锁文件中的数据

YAML 是用文本表示结构化数据的格式，能够表达键值映射、列表和字符串等数据。缩进表示层级，常见的 `键: 值` 表示映射项，`-` 开头的条目表示列表项。层级缩进使用空格，不使用制表符。

Podfile.lock 使用 YAML 保存数据。下面是经过简化的片段，不是完整锁文件：

```yaml
PODS:
  - Alamofire (5.9.1)

DEPENDENCIES:
  - Alamofire (~> 5.9)
```

在这个例子中，`PODS` 和 `DEPENDENCIES` 是两个键，每个键对应一个列表。YAML 解析器把 `Alamofire (5.9.1)` 读取为一个字符串；库名和括号内版本的含义由 CocoaPods 定义，并不是 YAML 自动生成的两个字段。

Podfile 使用 Ruby 表达并执行配置逻辑，Podfile.lock 使用 YAML 保存结果数据。YAML 本身不执行 Podfile 中的 `if`、`target` 或 `pod` 调用。

## Gemfile 与 Ruby 工具版本

gem 是 Ruby 软件包的发布单位。CocoaPods 自身以名为 `cocoapods` 的 gem 发布。Gemfile 声明项目使用的 Ruby 软件包及版本要求，Bundler 是根据 Gemfile 安装这些软件包并管理版本组合的工具。

例如下面把 CocoaPods 工具版本固定为 1.16.2；该版本号仅用于示例，不代表最新版：

```ruby
source 'https://rubygems.org'

gem 'cocoapods', '1.16.2'
```

`source` 指定获取 Ruby 软件包的来源。Gemfile 也采用 Ruby DSL，但由 Bundler 解释；Podfile 则由 CocoaPods 解释，两者的配置方法和管理对象不同。

在包含 Gemfile 的项目目录中执行：

```sh
bundle install
bundle exec pod install
```

第一条命令安装 Gemfile 要求的 Ruby 软件包，并依据已有 Gemfile.lock 使用锁定版本；首次确定版本时会生成 Gemfile.lock。第二条命令在 Bundler 选定的 Ruby 软件包环境中运行 `pod install`，再由 CocoaPods 安装 Podfile 描述的 iOS 依赖。

因此，Gemfile/Gemfile.lock 可以控制“使用哪个版本的 CocoaPods 工具”，Podfile/Podfile.lock 则控制“应用使用哪些 pod 版本”。仅共享 Podfile.lock，不等于已经统一 CocoaPods 工具版本。

## 用 Python 读取 Podfile

如果目标是获取 Podfile 执行后的依赖配置，可以由 Python 启动 CocoaPods 命令，让 Ruby 和 CocoaPods 按实际规则执行文件，再读取结构化输出。仅用正则表达式寻找 `pod '名称'` 会漏掉变量、条件判断、循环、辅助方法和加载其他文件等情况。

CocoaPods 的两个命令输出格式不同：

| 命令 | 输出 |
| --- | --- |
| `pod ipc podfile Podfile` | YAML |
| `pod ipc podfile-json Podfile` | JSON |

JSON 是另一种结构化数据格式，可以由 Python 标准库的 `json` 模块解析。不能直接把第一个命令的 YAML 输出交给 `json.loads`。

下面的示例假设 Podfile 所在目录有适用的 Gemfile，且已经执行过 `bundle install`。代码使用该 Bundler 环境中的 CocoaPods：

```python
import json
import subprocess
from pathlib import Path


def parse_podfile(path: str) -> dict:
    podfile = Path(path).expanduser().resolve(strict=True)
    result = subprocess.run(
        [
            "bundle", "exec", "pod", "ipc", "podfile-json", str(podfile),
            f"--project-directory={podfile.parent}",
            "--no-ansi",
        ],
        cwd=podfile.parent,
        stdout=subprocess.PIPE,
        encoding="utf-8",
        check=True,
    )
    return json.loads(result.stdout)
```

Python 先把路径解析为存在的绝对路径，再运行 CocoaPods。`cwd` 指定子进程的工作目录，`--project-directory` 指定 CocoaPods 的项目目录。`stdout` 保存正常输出，错误输出仍单独发送到标准错误流，避免错误信息被混入 JSON。`check=True` 让非零退出状态触发异常；命令成功后，`json.loads` 把输出转成 Python 数据。

没有使用 Bundler 的环境，可以明确选择直接运行 `pod ipc podfile-json`。不要在 Bundler 执行失败后自动改用另一个系统版本的 CocoaPods，否则同一输入可能得到不同结果。

这个命令得到的是执行后的依赖声明，不是一次 `pod install`：命令本身不完成依赖解析、下载和工程集成，也不运行正常安装流程中的 `post_install` 钩子。钩子是 Podfile 注册给 CocoaPods、由指定阶段调用的代码块。Podfile 顶层的任意 Ruby 代码仍会执行，因此读取工具只能执行可信文件，不能把这一方式当作无副作用的纯文本解析。

结果中的 `target_definitions` 及目标的 `children` 表示目标层级。处理工具需要保留层级，并按 `inherit!` 等设置解释父子目标之间的依赖继承；不能把所有目标的依赖合并后，当作每个 target 都拥有同一组依赖。实际已确定的版本应另外读取 Podfile.lock，并考虑锁文件可能尚未更新的情况。

环境变量、被加载的辅助文件和 CocoaPods 工具版本都可能影响结果。以上条件需要与实际安装时一致。例如 `USE_NETWORK=1` 与变量未设置时，前面的条件示例会产生不同声明。Podfile 自己向标准输出打印文本，也可能污染 JSON 输出；文件和辅助代码应把诊断信息写到标准错误流，不能由读取工具随意截取或忽略异常输出。

结构化输出通常不保留原始代码、注释和格式，因此不适合直接用于保真改写 Podfile。若必须在不执行 Ruby 的情况下分析文件，可以使用 Ruby 语法解析器构建 AST，即表示代码语法结构的树。静态工具应只解释明确支持的写法；无法确定的动态表达式应报告为未知，不能静默忽略并声称已得到完整依赖集合。
