# WebView 与 Lynx 双向通信

[返回笔记目录](../README.md)

## 本文目录

- [Web、WebView、Lynx 与 ReactLynx](#webwebviewlynx-与-reactlynx)
- [XML、HTML 与界面描述](#xmlhtml-与界面描述)
- [JSBridge 的职责与接口分工](#jsbridge-的职责与接口分工)
- [WKWebView：JavaScript 调用 Native](#wkwebviewjavascript-调用-native)
- [WKWebView：Native 执行 JavaScript](#wkwebviewnative-执行-javascript)
- [callbackId 与请求响应关联](#callbackid-与请求响应关联)
- [Lynx 原生模块注册与回调](#lynx-原生模块注册与回调)
- [Lynx 事件监听与原生通知](#lynx-事件监听与原生通知)

整理网页和 Lynx 页面与 Native 之间的消息、模块、回调和事件机制。代码为示例，接入时需要匹配实际 SDK 版本。

## Web、WebView、Lynx 与 ReactLynx

**在 App 内展示页面时，Web 页面通常由 WebView 中的浏览器引擎渲染，Lynx 移动端页面由 Lynx 引擎渲染。两者都可以使用 JavaScript，但构建界面的路径不同。**

- **Web** 是基于 HTML、CSS、JavaScript 等标准构建网页的技术体系。HTML 描述内容结构，CSS 描述样式，JavaScript 实现交互逻辑。
- **WebView** 是 App 中承载网页的组件。iOS 的 `WKWebView` 由 WebKit 提供，负责加载和显示网页。WebKit 主要使用 C++ 等语言实现，Apple 平台接口还涉及 Objective-C／Objective-C++；业务代码可以用 Swift 调用这些接口，不能因此认为 WebKit 是用 Swift 实现的。
- **Lynx** 提供跨平台的页面运行、样式、布局和渲染能力。**ReactLynx** 是基于 Lynx 的 React 风格 UI 框架，负责组件、状态和更新计算。二者与 JavaScript 引擎的分工见[框架与引擎](lynx-rendering-and-updates.md#页面描述到平台-ui-的渲染流程)。

以下对比 iOS App 中普通网页与 Lynx 原生渲染页面：

| 对比项 | Web 页面 | Lynx 页面 |
|-|-|-|
| 容器 | `WKWebView` | `LynxView` |
| 渲染 | WebKit 按网页规则布局和绘制 | Lynx 引擎布局，并通过平台 UI 实现呈现界面 |
| 页面元素 | `<div>`、`<span>`、`<img>` 等 HTML 元素 | `<view>`、`<text>`、`<image>` 等 Lynx 元素 |
| JS 环境 | 浏览器提供 `window`、`document` 等对象 | Lynx 提供自己的接口，不能直接假定完整浏览器接口可用 |
| 调用宿主业务 | 可通过 `postMessage` 等机制桥接 | 可通过 `NativeModules` 等机制桥接 |

**使用 JavaScript 不等于运行网页。**`window`、`document` 属于浏览器环境提供的对象，不是 JavaScript 语言自身的能力。例如 `document.getElementById("title").textContent = "商品详情"` 依赖浏览器接口，不能直接搬到普通 ReactLynx 页面。ReactLynx 通常通过组件状态驱动界面更新。

Lynx 渲染原生界面，也不代表 JS 能直接调用任意 Swift 方法；App 的原生业务能力仍需通过模块或组件接口暴露。Lynx 也有 Web 端适配能力，因此上述对比限定为移动端原生渲染场景，不表示 Lynx 只能运行在移动端。

依据：[WKWebView](https://developer.apple.com/documentation/webkit/wkwebview/)、[WebKit Apple 平台接口实现](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/API/Cocoa/WKUserContentController.mm)、[ReactLynx](https://lynxjs.org/react/introduction)、[Lynx 项目](https://github.com/lynx-family/lynx)。

## XML、HTML 与界面描述

**XML 是描述结构化文档和数据的通用标记语言；HTML 专门描述网页结构和内容含义。** 标记语言通过标签表达内容及其组织关系。XML 允许应用定义标签含义，HTML 标准已经规定了常用网页标签的含义。

例如，用 XML 描述商品：

```xml
<product>
    <name>运动鞋</name>
    <price>299</price>
</product>
```

`product`、`name`、`price` 是应用约定的标签。XML 解析器读取内容和层级，但 XML 本身没有规定商品名显示成标题、价格显示成红色。应用需要另外实现这些规则。

用 HTML 描述网页内容：

```html
<h1>运动鞋</h1>
<p>价格：299 元</p>
<button>购买</button>
```

浏览器知道 `h1` 是一级标题、`p` 是段落、`button` 是按钮，并提供默认显示和交互行为。CSS 可以调整外观，JavaScript 可以实现点击后的业务逻辑。

| 对比项 | XML | 普通 HTML |
|-|-|-|
| 标签含义 | 应用按命名规则自行约定 | 内置标签由标准规定，也支持按规则定义自定义元素 |
| 标签名大小写 | 区分，`name` 与 `Name` 不同 | HTML 元素标签名不区分大小写 |
| 闭合要求 | 必须成对，空元素可写为 `<name/>` | 部分结束标签允许省略；`img`、`br` 等元素没有结束标签 |
| 错误处理 | 违反基本结构规则时，解析器报错并停止正常解析 | 浏览器按标准的错误恢复规则处理许多不规范写法 |

XML 也能描述界面，前提是框架规定标签对应什么组件，并负责创建组件。不能把 XML 限定为“只能存数据”。同样，ReactLynx 中的 `<view>`、`<text>` 是 JSX 中的 Lynx 元素描述；标签外观相似，不代表它们就是 HTML 网页元素。

依据：[XML 规范](https://www.w3.org/TR/xml/)、[HTML 语法规范](https://html.spec.whatwg.org/multipage/syntax.html)、[ReactLynx 组件](https://lynxjs.org/react/introduction#different-component-sets)。

## JSBridge 的职责与接口分工

**按广义用法，JSB 是连接 JavaScript 与 Native、让两侧调用功能和传递数据的桥接机制。** JSB 是机制的统称，具体接口负责其中的某个环节。

| 内容 | 在桥接中的职责 |
|-|-|
| `WKUserContentController`、`postMessage`、消息处理器 | 建立并使用 WebView 的 JS → Native 消息通道 |
| `evaluateJavaScript` | Native 在页面 JS 环境中执行脚本，可用来调用前端函数并传参 |
| `callbackId` | 请求与响应的关联字段，本身不负责传输 |
| `LynxModule`、导出方法、注册、`NativeModules` | 提供 Lynx 的 JS → Native 模块调用及结果回调 |
| `sendGlobalEvent`、`GlobalEventEmitter` | 在 Native 发事件的场景中，实现 Native → JS 通知 |

典型的业务桥接链路是：前端调用 → Native 根据注册信息路由到处理对象 → 校验输入 → 执行业务 → 回传结果。桥接成功仅说明消息或调用完成，不等于帖子发布等业务操作已经成功。

不能把 `NativeModules` 或事件通知笼统说成“完全独立于 JSB 的通信方式”。如果项目把一套具体的业务 SDK 命名为“JSB”，需要进一步区分“桥接机制”和“是否接入该 SDK”。

以下为示例代码，未在本次整理中编译或运行。WebKit 与 Lynx 的接口不能相互混用；Lynx 接入细节需要匹配实际 SDK 版本。

## WKWebView：JavaScript 调用 Native

**原生侧先登记消息入口及接收对象。JS 向这个入口发送参数后，WebKit 调用接收对象的方法。**

先编写接收消息的对象：

```swift
import WebKit

final class ProductMessageHandler: NSObject, WKScriptMessageHandler {
    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        guard message.name == "openDetail",
              let body = message.body as? [String: Any],
              let productId = body["productId"] as? String else {
            return
        }
        print("原生收到商品 ID：", productId)
        // 在这里调用实际的商品详情跳转业务。
    }
}
```

创建 `WKWebView` 前登记入口，并把配置传给该 WebView：

```swift
let handler = ProductMessageHandler()
let controller = WKUserContentController()
controller.add(handler, name: "openDetail")

let configuration = WKWebViewConfiguration()
configuration.userContentController = controller
let webView = WKWebView(frame: .zero, configuration: configuration)
```

在这个 WebView 加载的页面中，JS 发送消息：

```javascript
window.webkit.messageHandlers.openDetail.postMessage({
    productId: "123"
});
```

对应关系是：

```text
JS 发送到 openDetail，参数为 { productId: "123" }
  → WebKit 根据登记关系找到 handler
  → 调用 handler 的 userContentController(_:didReceive:)
  → message.name 为 "openDetail"，message.body 包含参数
  → handler 读取参数并执行原生业务
```

`WKUserContentController` 管理消息入口与接收对象的登记关系，并参与 WebKit 的消息分发。真正接收业务回调的是遵循 `WKScriptMessageHandler` 的 `handler` 对象。协议规定了接收方法的形式，协议本身不是接收对象。

`userContentController(_:didReceive:)` 虽然以 `userContentController` 开头，却是 `handler` 实现的方法。第一个参数只是把有关的 `WKUserContentController` 实例传给 `handler`。`openDetail` 也是开发者约定的入口名，不会自动变成同名 Swift 方法或自动完成跳转。

依据：[WKUserContentController](https://developer.apple.com/documentation/webkit/wkusercontentcontroller)、[WKScriptMessageHandler](https://developer.apple.com/documentation/webkit/wkscriptmessagehandler)、[公开实现](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/API/Cocoa/WKUserContentController.mm)。

## WKWebView：Native 执行 JavaScript

**Native 把一段 JS 源代码作为字符串传给 `webView.evaluateJavaScript`。WebKit 在该 WebView 的页面 JS 环境中执行代码，并通过完成回调把执行结果或错误传回 Native。**

页面先定义可调用的函数：

```html
<p id="productLabel"></p>
<script>
window.receiveProduct = function(productId) {
    document.getElementById("productLabel").textContent =
        "商品 ID：" + productId;
    return "更新成功";
};
</script>
```

Native 在这个函数已经定义后调用：

```swift
let script = """
window.receiveProduct("123");
"""

webView.evaluateJavaScript(script) { result, error in
    if let error = error {
        print("脚本执行失败：", error)
        return
    }
    print("JS 返回：", result as Any)
}
```

流程：`Swift 创建字符串 → WebKit 执行字符串中的 JS → receiveProduct("123") 修改页面并返回字符串 → Native 完成回调收到结果`。

Swift 创建 `script` 时只是在创建字符串，尚未执行 JS。JS 函数体由 JavaScript 引擎运行；Swift 的完成闭包由原生侧执行。示例中的 `result` 对应 JS 返回的“更新成功”，不需要再额外调用 `postMessage` 才能获得这个返回值。

调用必须针对正确的 WebView，并确保页面函数已经定义。该接口的完成回调不等于函数内部所有异步任务已经完成；若函数另行发起网络请求，需要按其异步协议接收结果。示例采用固定参数；实际传入动态字符串时应通过 JSON 序列化等方式正确编码，避免直接拼接未转义内容。三引号语法见[Swift 多行字符串](../language/swift-basics.md#多行字符串与脚本执行)。

依据：[evaluateJavaScript](https://developer.apple.com/documentation/webkit/wkwebview/evaluatejavascript(_:completionhandler:))。

## callbackId 与请求响应关联

**`callbackId` 是自行封装桥接协议时，为请求分配的关联标识。** 它不是 `WKWebView` 自动提供的字段，也不是函数本身。这里的“协议”指双方约定的消息格式和处理规则，不是 Swift 的 `protocol`。

例如，前端连续查询两个商品：

```text
请求 A：productId = "123"，callbackId = "cb_1"
请求 B：productId = "456"，callbackId = "cb_2"

前端保存：
"cb_1" → 请求 A 的 JS 回调函数
"cb_2" → 请求 B 的 JS 回调函数
```

1. 前端保存回调函数，并将 `callbackId` 连同请求参数发给 Native。
2. Native 执行对应业务。返回结果时，Native 携带同一个 `callbackId`。
3. 前端统一的响应入口读取 `callbackId`，找到并调用对应回调，把结果传进去。
4. 对一次性请求回调，前端在完成后移除对应记录。

即使 B 先完成，`cb_2` 也能定位到 B 的函数，不必假定响应与请求顺序一致。Native 通过 `evaluateJavaScript` 把响应交给前端入口，随后由页面 JS 查表并执行回调；不是把 JS 函数体交给 Swift 执行。

这是协议设计示例，不代表所有 JSB 都要求业务方手动维护 `callbackId`。Lynx 的[模块回调](#lynx-原生模块注册与回调)已提供函数参数的桥接能力，普通使用方可以直接传函数。

## Lynx 原生模块注册与回调

**`NativeModules` 是 Lynx 提供给 JS 的原生模块访问入口。原生侧实现模块、导出方法并注册后，JS 才能通过模块名和方法名调用对应能力。**

这里的“原生模块”是一组提供给 JS 调用的原生功能，iOS 中通常用类承载。它与 Swift／Xcode 的编译模块不是同一概念：一个 App 或 Framework 模块中可以定义多个 Lynx 原生模块，不需要为每个原生模块建立一个 Target。编译模块的定义见[Target、模块和产物](../engineering/xcode-and-testing.md#target模块与构建产物)。

下面以公开 SDK 的 Swift 写法示意；宿主需要先导入实际 SDK 提供的 `LynxModule` 声明：

```swift
import Foundation

@objcMembers
final class ProductModule: NSObject, LynxModule {
    static var name: String { "ProductModule" }

    static var methodLookup: [String: String] {
        [
            "getProductName": NSStringFromSelector(
                #selector(getProductName(_:callback:))
            )
        ]
    }

    func getProductName(
        _ productId: String,
        callback: (String) -> Void
    ) {
        print("原生收到商品 ID：", productId)
        callback("示例商品")
    }
}
```

`name` 决定 JS 使用的模块名；`methodLookup` 将 JS 方法名映射到 Objective-C 方法名。`@objcMembers` 的范围见[OC 兼容成员](../language/swift-objc-runtime.md#objcmembers-与-objective-c-兼容限制)，Selector 转字符串的过程见[NSStringFromSelector](../language/swift-objc-runtime.md#selector-字符串与方法映射)。

宿主初始化时注册模块，例如：

```swift
globalConfig.registerModule(ProductModule.self)
```

这里的 `globalConfig` 指宿主已经创建、并用于对应 Lynx 环境的配置对象。注册和容器初始化须使用同一套有效配置；具体初始化入口以所接入的 SDK 为准。

前端在 Lynx 的后台 JavaScript 环境中调用：

```javascript
NativeModules.ProductModule.getProductName("123", function(name) {
    console.log("收到商品名称：", name);
});
```

```text
JS 通过 NativeModules 指定模块、方法、参数和回调
  → Lynx 找到已注册的 ProductModule
  → 根据 methodLookup 找到原生方法，并桥接参数
  → 原生方法取得结果，调用 callback("示例商品")
  → Lynx 将结果传回对应的 JS 函数
  → JavaScript 引擎执行函数体，console.log 输出结果
```

**原生侧调用 `callback(...)` 与执行 JS 函数体是两个环节。** Native 拿到的是 Lynx 提供的原生可调用封装。原生业务代码触发封装并传入结果；Lynx 将调用转交给对应 JS 环境，最终由 JavaScript 引擎执行前端函数体。

“后台 JavaScript 环境”指运行在后台 JS 线程的执行环境，不是 App 退到后台。JS 从后台线程调用模块，也不能据此断定原生方法及其全部后续业务都在同一线程执行；线程安排需看 SDK 和原生实现。已有双线程分工见[线程模型](lynx-rendering-and-updates.md#双线程首帧直出与后台接管)。

示例同步返回固定结果，因此不需要因“回调”这个名称就添加 `@escaping`。若原生实现保存闭包，或让闭包可能在当前方法返回后执行，再根据 Swift 的规则使用[逃逸闭包](../language/swift-basics.md#逃逸闭包与执行时机)。

依据：[Lynx Native Modules](https://lynxjs.org/guide/use-native-modules.html)。

## Lynx 事件监听与原生通知

**前端先通过 `GlobalEventEmitter` 登记事件处理函数。Native 发送事件名称和数据后，Lynx 调用该事件对应的前端函数。** 事件由事件名称和本次携带的数据组成；不要求前端先发起一次请求。

前端在后台 JavaScript 环境中登记：

```javascript
const emitter = lynx.getJSModule("GlobalEventEmitter");

function handleLoginChanged(data) {
    console.log("当前是否登录：", data.isLoggedIn);
    // 根据状态更新页面。
}

emitter.addListener("loginStateChanged", handleLoginChanged);
```

`addListener` 保存事件名与函数的对应关系，登记时不立即执行函数。`loginStateChanged` 是双方约定的事件名，`handleLoginChanged` 是开发者编写的函数名，两者不需要同名。

Native 完成登录后，向对应页面的 `LynxView` 实例发送事件：

```swift
lynxView.sendGlobalEvent(
    "loginStateChanged",
    withParams: [["isLoggedIn": true]]
)
```

`lynxView` 指已创建并加载该页面的实例。`withParams` 外层数组表示传给 JS 处理函数的参数列表；内层字典是第一个参数的内容。因此，前端函数的 `data` 收到 `{ isLoggedIn: true }`。

```text
前端先登记 loginStateChanged 的监听函数
  → 原生登录完成，sendGlobalEvent 发送名称和状态
  → Lynx 按事件名找到监听函数
  → JS 引擎执行 handleLoginChanged(data)
```

Native 只需要知道事件名，不需要知道前端函数名。这里的“全局”指对应 Lynx 页面／JS 环境范围，不表示自动通知 App 中所有 Lynx 页面。Native 发事件只是这个事件系统的一种用法，前端也可以使用事件机制。

监听不再需要时，用登记时的同一个函数引用移除：

```javascript
emitter.removeListener("loginStateChanged", handleLoginChanged);
```

事件与请求回调的区别：事件监听可接收多次状态变化，不需要每次先发出请求；一次请求对应一次结果时，则使用模块回调或约定的 `callbackId` 协议。登记前发出的事件不能仅凭上述接口假定会自动补发。

依据：[Lynx 事件传播](https://lynxjs.org/next/guide/interaction/event-handling/event-propagation.html?platform=ios)、[LynxView.sendGlobalEvent](https://lynxjs.org/3.8/api/lynx-native-api/lynx-view/send-global-event.html)。
