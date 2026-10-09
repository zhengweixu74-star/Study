# 知识笔记

[返回 Study](../README.md)

按 7 个主题整理的 15 篇学习笔记。每篇文档保留概念说明、工作机制、代码示例和参考资料；通过篇内目录与文档间链接查阅相关内容。

整理日期：2026-10-09。

## 语言与运行时

| 文档 | 内容范围 |
| --- | --- |
| [Swift 语言基础](language/swift-basics.md) | 类型、属性、协议与泛型、闭包、集合和相等规则；Swift 与 C 接口互操作 |
| [Swift / Objective-C 运行时与方法派发](language/swift-objc-runtime.md) | 对象布局、关联对象、方法派发、Objective-C 互操作与 KVO；+load 与自动注册 |

## iOS 界面、事件与调度

| 文档 | 内容范围 |
| --- | --- |
| [iOS 应用启动、窗口与输入事件](ios/app-lifecycle-and-events.md) | UIApplication、AppDelegate、启动到首帧、UIWindow、触摸分发与键盘输入 |
| [UIKit 布局、列表与 MVVM](ios/uikit-layout-and-mvvm.md) | 生命周期、视图层级、布局、列表复用、ViewModel 与容器控制器 |
| [响应式编程与事件通知](ios/reactive-programming.md) | NotificationCenter、OpenCombine、订阅生命周期、状态与事件、调度 |
| [iOS 线程与任务调度](ios/threads-and-scheduling.md) | 进程与线程、GCD、线程安全、锁、RunLoop、Timer 与 CADisplayLink |
| [Core Animation 渲染与性能分析工具](ios/rendering-and-performance.md) | 图层动画、渲染管线、卡顿监测、Time Profiler、Animation Hitches 与 MetricKit |

## 跨平台与宿主通信

| 文档 | 内容范围 |
| --- | --- |
| [WebView 与 Lynx 双向通信](cross-platform/webview-and-lynx-bridge.md) | WebView 和 Lynx 的消息、原生模块、请求回调与事件通知 |
| [Lynx 渲染、线程模型与资源更新](cross-platform/lynx-rendering-and-updates.md) | Lynx 渲染、双线程、首帧直出、后台接管与动态资源更新 |

## 软件设计与工程工具

| 文档 | 内容范围 |
| --- | --- |
| [设计模式与 Swift 实践](engineering/design-patterns.md) | 策略、工厂、单例、建造者，以及 Swift 示例与独立验证 |
| [Xcode 工程管理与自动化测试](engineering/xcode-and-testing.md) | 源码编辑与搜索、工程与构建、XCTest、Swift Testing 与测试计划 |

## 网络与存储

| 文档 | 内容范围 |
| --- | --- |
| [网络请求与代理](network-and-storage/network-and-proxy.md) | 连接与超时、幂等重试、TCP、代理、HTTPS 抓包与证书信任 |
| [iOS 文件存储](network-and-storage/ios-file-storage.md) | Documents、Caches 和 tmp 的用途与备份边界 |

## 实验与统计

| 文档 | 内容范围 |
| --- | --- |
| [A/B 实验与统计](statistics/ab-testing.md) | A/B 显著性、p 值、有效样本量与结果解读 |

## AI 辅助开发

| 文档 | 内容范围 |
| --- | --- |
| [AI Agent 原理与开发验证](ai/agents-and-development.md) | 模型与 Agent、工具调用、Skill / Tool / MCP、代码审查与开发验证 |

## 文档锚点、内部链接与交叉引用

锚点是文档中用于定位跳转目标的标记。HTML 可用 `id` 声明锚点，Markdown 链接可通过片段标识指向该位置。

```html
<a id="symbol-search"></a>
```

```markdown
[符号搜索](#symbol-search)
```

同一文档中的锚点 id 应保持唯一。修改标题但保留 id 时，基于该 id 的链接仍可定位。具体是否支持 HTML 锚点，由文档渲染器决定。

交叉引用是在一处内容中指向另一处相关内容。本笔记将原始单文件内部引用转换为分类文档链接，方便从不同主题跳转。
