# iOS 本地多语言与即时切换

[返回笔记目录](../README.md)

本文说明如何用随 App 打包的中英文资源切换文案，并在不访问服务器的情况下刷新已经创建的界面。

## 本文目录

- [本地化资源与文案键](#本地化资源与文案键)
- [String Catalog 的作用](#string-catalog-的作用)
- [选择语言资源与保存偏好](#选择语言资源与保存偏好)
- [刷新已经创建的界面](#刷新已经创建的界面)
- [Locale 与日期数字格式](#locale-与日期数字格式)
- [离线切换的范围与限制](#离线切换的范围与限制)

## 本地化资源与文案键

本地化是为特定语言或地区提供对应的文字、格式和资源。文案键是标识一条文案的固定字符串，各语言的资源文件使用同一个文案键；程序根据文案键和当前语言，查找对应的翻译。

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

Bundle 是访问一组打包资源的对象。程序通常调用 NSLocalizedString 或 Bundle.localizedString 查找文案。默认情况下，系统根据系统语言偏好和 App 支持的语言选择使用哪个 .lproj 目录。需要在 App 内即时切换语言时，程序自己加载选中语言的 .lproj 目录。

也可以把翻译维护在本地 JSON 文件中，再用自定义代码按键查找。JSON 方案要自己实现文件加载、缺失文案的处理和复数规则；系统本地化接口提供的这些功能，JSON 方案都需要重新实现。

## String Catalog 的作用

String Catalog 是 Xcode 15 及以上支持的本地化资源管理格式，文件扩展名为 .xcstrings。Xcode 以表格界面展示文案键、源语言、翻译和翻译状态；文件本身采用 JSON 结构。

一个 Localizable.xcstrings 文件可以集中维护多种语言。sourceLanguage 字段标识源语言；每个文案条目保存各语言的本地化值和翻译状态。文案需要随数量变化时，可以维护复数变体，例如英语中的“1 item”和“2 items”。系统按每种语言自己的复数规则选择形式，各语言的复数形式数量不同，有的语言多于两种。

构建时，Xcode 把 String Catalog 编译成运行时可读取的本地化资源。运行时代码通常仍通过本地化 API 查找文案，不直接解析 .xcstrings 文件。String Catalog 只改变资源的维护方式；已经赋值的 UILabel.text 仍需要程序重新赋值才会更新。

## 选择语言资源与保存偏好

语言管理对象保存当前语言和对应的资源 Bundle，并提供统一的文案查找方法。UserDefaults 只保存语言码，例如 "en"；下次启动时，语言管理对象从 UserDefaults 读取语言码。UserDefaults 支持的类型和读写机制见 [iOS 数据持久化](../network-and-storage/ios-data-persistence.md#userdefaults-的最小示例)。

下面的示例只支持简体中文和英语，没有保存过偏好时使用英语。两种语言的资源都必须打包在主 Bundle 中。主 Bundle 中缺少某种支持语言的资源属于打包错误，示例遇到这种情况直接调用 preconditionFailure 报错。

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

LanguageManager 只在主线程使用。初始化时，LanguageManager 先读取偏好，再加载对应语言的 Bundle；偏好不存在或不在 AppLanguage 中时，使用英语。切换时，`select(_:)` 依次加载新语言的 Bundle、更新 language、保存偏好，最后发送 didChange 通知。UserDefaults 异步写入磁盘；正常情况下，App 重启后仍能读到保存的语言。

文案键缺失时，`text(_:)` 可能返回文案键本身，界面上会直接显示文案键。所以要在资源维护和发布校验时检查每种支持语言的文案是否齐全。真实项目用系统语言作为初始值时，需要定义匹配规则，例如把系统返回的 "en-US" 映射到本示例中的 "en"。

## 刷新已经创建的界面

UILabel.text 保存的是赋值时查到的字符串。LanguageManager 更换 Bundle 后，UILabel.text 仍是旧字符串。页面要重新查找文案并赋值。

NotificationCenter 是向已注册观察者分发通知的系统对象。LanguageManager 在语言切换完成后发送 didChange 通知；页面收到通知后调用 refreshTexts()。通知注册与清理机制见 [响应式编程与事件通知](reactive-programming.md#notificationcenter-与通知回调)。

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

示例在主线程同步发送通知，观察者也在主线程同步执行 refreshTexts()。已经加载但暂未显示的页面同样收到通知并立即更新；尚未加载的页面在 viewDidLoad 中读取当前语言。列表页面要刷新可见 Cell，之后配置的 Cell 会自动使用当前语言。

刷新时保留输入内容、选中项和滚动位置等用户状态，只更新受语言影响的文字、图片和格式。文案长度变化时，触发布局重新计算。必须重建页面时，先确定要恢复哪些状态，重建后恢复这些状态。

## Locale 与日期数字格式

Locale 是描述语言、地区及格式习惯的对象。语言决定使用哪套文案；地区决定日期顺序、小数分隔符和货币等格式。同一种语言可以搭配不同地区。

例如 en_US 和 en_GB 的语言都是英语，en_US 使用美国的格式习惯，en_GB 使用英国的格式习惯。下面固定日历、时区和格式模板，展示同一个日期：

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

具体标点和补零由系统的地区数据决定。时区由 DateFormatter.timeZone 单独控制，修改 locale 不改变时区。货币金额的币种由数据本身决定，例如一笔人民币金额在英语界面中仍显示为人民币。

切换格式时，代码要更新 DateFormatter、NumberFormatter 等对象的 locale，再重新生成显示字符串。已经生成的格式化字符串不会随文案 Bundle 的更换而更新。

## 离线切换的范围与限制

本地资源齐全时，静态文案可以完全离线切换。图片中的文字需要各语言的图片资源；日期、数字和复数需要按当前语言格式化。

修改 UserDefaults 中的 AppleLanguages 后，已经创建的 UIKit 界面不会刷新，要使用本文的资源选择与刷新机制。App 自己保存语言偏好，App 内的语言选择就与系统语言相互独立。

服务器只返回中文正文时，客户端切换语言后，正文仍是中文。要显示英文正文，需要服务器提供多语言内容、随 App 打包对应内容，或者接入翻译能力。切换静态资源只影响打包的文案，用户输入和其他动态内容保持原样。
