# iOS 本地多语言与即时切换

[返回笔记目录](../README.md)

本文面向熟悉 Swift 与 UIKit 的读者，说明如何用随 App 打包的中英文资源切换文案，并在不依赖服务器的条件下刷新已有界面。

## 本文目录

- [本地化资源与文案键](#本地化资源与文案键)
- [String Catalog 的作用](#string-catalog-的作用)
- [选择语言资源与保存偏好](#选择语言资源与保存偏好)
- [刷新已经创建的界面](#刷新已经创建的界面)
- [Locale 与日期数字格式](#locale-与日期数字格式)
- [离线切换的范围与限制](#离线切换的范围与限制)

## 本地化资源与文案键

本地化是为特定语言或地区提供合适的文字、格式和资源。文案键是标识一条文案的稳定字符串；程序根据文案键和当前语言，查找对应的翻译。

离线切换需要把支持语言的资源随 App 打包。以 Localizable.strings 为例，资源分别位于语言目录：

```text
zh-Hans.lproj/Localizable.strings
en.lproj/Localizable.strings
```

两份文件使用相同文案键：

```text
// zh-Hans.lproj/Localizable.strings
"settings.title" = "设置";

// en.lproj/Localizable.strings
"settings.title" = "Settings";
```

Bundle 是访问一组打包资源的对象。通常通过 NSLocalizedString 或 Bundle.localizedString 查找文案。默认资源选择由系统语言偏好和 App 支持的语言决定；需要 App 内即时切换时，可以显式加载选中语言的 .lproj 目录。

也可以把翻译维护在本地 JSON 文件中，再用自定义代码按键查找。JSON 方案需要自行实现文件加载、缺失文案规则和复数等能力，不会自动获得系统本地化接口的全部行为。

## String Catalog 的作用

String Catalog 是 Xcode 15 及以上支持的本地化资源管理格式，文件扩展名为 .xcstrings。Xcode 以表格界面展示文案键、源语言、翻译和翻译状态；文件本身采用 JSON 结构。

一个 Localizable.xcstrings 可以集中维护多种语言。sourceLanguage 标识源语言；各文案条目保存不同语言的本地化值及状态。需要表达数量变化时，可以维护复数变体，例如英语中的“1 item”和“2 items”；实际形式按语言的复数规则选择，不能把所有语言都简化为单复数两种形式。

String Catalog 在构建时转化为可供运行时读取的本地化资源；运行时代码通常仍通过本地化 API 查找文案，不直接解析 .xcstrings。它改善资源维护方式，不会自动为已经赋值的 UILabel 更新 text。

## 选择语言资源与保存偏好

语言管理对象负责保存当前语言和资源对象，并提供统一的文案查找方法。UserDefaults 只持久化语言码，例如 "en"；下一次启动时，语言管理对象读取该选择。UserDefaults 支持的类型和读写机制见 [iOS 数据持久化](../network-and-storage/ios-data-persistence.md#userdefaults-的最小示例)。

下面的示例只支持简体中文和英语。约定未保存偏好时使用英语；两种资源必须完整打包在主 Bundle 中。Bundle 中不存在声明支持的语言资源属于打包错误，示例直接报告该错误，不拼接其他资源来源。

```swift
import UIKit

enum AppLanguage: String {
    case simplifiedChinese = "zh-Hans"
    case english = "en"
}

@MainActor
final class LanguageManager {
    static let shared = LanguageManager()
    static let didChange = Notification.Name("AppLanguageDidChange")
    private static let preferenceKey = "preferredLanguage"

    private(set) var language: AppLanguage
    private var resourceBundle: Bundle

    private init() {
        let saved = UserDefaults.standard.string(forKey: Self.preferenceKey)
        let selected = saved.flatMap { AppLanguage(rawValue: $0) } ?? .english
        language = selected
        resourceBundle = Self.loadBundle(for: selected)
    }

    private static func loadBundle(for language: AppLanguage) -> Bundle {
        guard let path = Bundle.main.path(
            forResource: language.rawValue,
            ofType: "lproj"
        ), let bundle = Bundle(path: path) else {
            preconditionFailure("缺少语言资源：\(language.rawValue)")
        }
        return bundle
    }

    func text(_ key: String) -> String {
        resourceBundle.localizedString(
            forKey: key,
            value: nil,
            table: "Localizable"
        )
    }

    func select(_ newLanguage: AppLanguage) {
        guard newLanguage != language else { return }
        resourceBundle = Self.loadBundle(for: newLanguage)
        language = newLanguage
        UserDefaults.standard.set(newLanguage.rawValue, forKey: Self.preferenceKey)
        NotificationCenter.default.post(name: Self.didChange, object: self)
    }
}
```

LanguageManager 在主线程使用。启动时先读取偏好，再建立资源对象；切换时先更新资源与语言、保存偏好，最后发送通知。初始化时遇到不存在或不再支持的偏好值，会按示例约定使用英语。正常情况下，保存后 App 重启仍能读取选择；UserDefaults 的磁盘写入是异步的。

文案键缺失时，该查找方法最终可能返回键本身，不能视为翻译成功。应在资源维护和发布校验中检查支持语言是否齐全。真实项目若选择系统语言作为初始值，需要明确匹配规则，不能直接把 "en-US" 当作本示例枚举中的 "en"。

## 刷新已经创建的界面

UILabel.text 保存赋值时得到的字符串。语言管理对象更换 Bundle 后，UILabel 不会重新执行之前的查找。页面需要再次读取文案并赋值。

NotificationCenter 是向已注册观察者分发通知的系统对象。这里由 LanguageManager 在语言切换完成后发送通知；页面收到通知后调用 refreshTexts()。通知注册与清理机制见 [响应式编程与事件通知](reactive-programming.md#notificationcenter-与通知回调)。

下面的控制器在首次加载和收到通知时使用同一刷新方法：

```swift
final class SettingsViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(languageDidChange),
            name: LanguageManager.didChange,
            object: LanguageManager.shared
        )
        refreshTexts()
    }

    @objc private func languageDidChange(_ notification: Notification) {
        refreshTexts()
    }

    private func refreshTexts() {
        title = LanguageManager.shared.text("settings.title")
    }

    deinit {
        NotificationCenter.default.removeObserver(self)
    }
}

// 由主线程上的语言选择操作调用：
// LanguageManager.shared.select(.simplifiedChinese)
```

示例由主线程同步发送通知，页面也在主线程刷新。已经加载但暂未显示的页面可以同步更新；尚未加载的页面在 viewDidLoad 中读取当前语言。列表也需要更新可见 Cell，并让后续配置使用当前语言。

刷新时保留输入内容、选中项和滚动位置等用户状态。只更新受语言影响的文字、图片和格式；文案长度变化时，让布局重新计算。必须重建页面时，应先明确哪些状态需要恢复，不能通过丢失页面状态来换取文案刷新。

## Locale 与日期数字格式

Locale 是描述语言、地区及格式习惯的对象。语言主要决定使用哪套文案；地区影响日期顺序、小数分隔符和货币等格式。两者有关联，但不能等同。

例如 en_US 和 en_GB 都表示英语环境，但前者使用美国地区习惯，后者使用英国地区习惯。下面固定日历、时区和格式模板，展示同一个日期：

```swift
let formatter = DateFormatter()
formatter.calendar = Calendar(identifier: .gregorian)
formatter.timeZone = TimeZone(secondsFromGMT: 0)!
let date = Date(timeIntervalSince1970: 0)

formatter.locale = Locale(identifier: "en_US")
formatter.setLocalizedDateFormatFromTemplate("yMd")
let usDate = formatter.string(from: date) // 典型结果：1/1/1970

formatter.locale = Locale(identifier: "en_GB")
formatter.setLocalizedDateFormatFromTemplate("yMd")
let gbDate = formatter.string(from: date) // 典型结果：01/01/1970
```

具体标点和补零由系统的地区数据决定。Locale 不负责把日期转换到另一个时区；DateFormatter.timeZone 单独控制时区。货币金额的币种也应按数据含义确定，不能因为用户切到英语就把人民币自动变成美元。

需要切换格式时，代码应更新 DateFormatter、NumberFormatter 等对象的 locale，并重新生成显示字符串。只更换文案 Bundle 不会自动更新已有格式化结果。

## 离线切换的范围与限制

本地资源齐全时，静态文案可以完全离线切换。图片中的文字、日期数字、复数和其他语言相关内容，也需要相应资源或格式化逻辑。

修改 AppleLanguages 不能自动刷新已经创建的 UIKit 界面，也不能代替明确的资源选择与刷新机制。App 内保存独立语言偏好，可以让该选择与系统语言分开。

服务器只返回中文正文时，本地切换不能自动生成英文正文。需要由服务器提供多语言内容、随 App 保存对应内容，或另行引入翻译能力；仅切换静态资源不会翻译用户输入和任意动态内容。
