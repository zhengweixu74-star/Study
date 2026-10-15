# 知识笔记

[返回 Study](../README.md)

按 8 个主题整理的 22 篇学习笔记。内容限于通用概念、工作机制和独立示例；通过篇内目录与笔记之间的内部链接查阅相关内容。

整理日期：2026-10-09。

## 分类目录

| 分类 | 文档数 |
| --- | --- |
| [语言与运行时](language/) | 2 |
| [iOS 界面、事件与调度](ios/) | 7 |
| [Android 界面与动画](android/) | 4 |
| [跨平台与宿主通信](cross-platform/) | 2 |
| [软件设计与工程工具](engineering/) | 3 |
| [网络与存储](network-and-storage/) | 2 |
| [实验与统计](statistics/) | 1 |
| [AI 辅助开发](ai/) | 1 |

## 语言与运行时

| 文档 | 内容范围 |
| --- | --- |
| [Swift 语言基础](language/swift-basics.md) | 类型、属性、协议与泛型、闭包、集合和相等规则；Swift 与 C 接口互操作 |
| [Swift / Objective-C 运行时与方法派发](language/swift-objc-runtime.md) | 对象布局、关联对象、方法派发、Objective-C 互操作与 KVO；+load 与自动注册 |

相关专题：[协议继承与类继承的区别](language/swift-basics.md#协议继承与类继承的区别)。

相关专题：[ARC 与 Android GC](language/swift-basics.md#arc-与-android-gc引用计数可达性与引用环)。内容范围：强引用计数、GC Roots、可达性、引用环与内存泄漏。

相关专题：[结构体、枚举与值的生命周期](language/swift-basics.md#结构体枚举与值的生命周期)。内容范围：ARC 与 GC 的管理边界、值内部的强引用、作用域与生命周期，以及局部值、替换和返回值的清理规则。

## iOS 界面、事件与调度

[Darwin、应用框架与系统服务](ios/darwin-frameworks-and-services.md)：Darwin 与 XNU 的关系、基础系统组件、应用框架、系统服务及通知协作示例。

| 文档 | 内容范围 |
| --- | --- |
| [iOS 应用启动、窗口与输入事件](ios/app-lifecycle-and-events.md) | UIApplication、AppDelegate、启动到首帧、UIWindow、触摸分发与键盘输入 |
| [UIKit 布局、列表与 MVVM](ios/uikit-layout-and-mvvm.md) | 生命周期、视图层级、布局、列表复用、ViewModel 与容器控制器 |
| [响应式编程与事件通知](ios/reactive-programming.md) | NotificationCenter、OpenCombine、订阅生命周期、状态与事件、调度 |
| [iOS 线程与任务调度](ios/threads-and-scheduling.md) | 进程与线程、GCD、线程安全、锁、RunLoop、Timer 与 CADisplayLink |
| [Core Animation 渲染与性能分析工具](ios/rendering-and-performance.md) | 图层动画、渲染管线、卡顿监测、Time Profiler、Animation Hitches 与 MetricKit |

相关专题：[内核、逻辑 CPU 与线程并发](ios/threads-and-scheduling.md#内核的职责与进程信息)；[进程创建时间与启动计时起点](ios/app-lifecycle-and-events.md#进程创建时间与启动计时起点)。

相关专题：[获取锁失败后的等待行为](ios/threads-and-scheduling.md#获取锁失败后的等待行为)；[局部坐标、命中测试与点击长按协调](ios/app-lifecycle-and-events.md#uiview-的局部坐标与-bounds)；[UICollectionView 分区、重点回调与布局协议](ios/uikit-layout-and-mvvm.md#分区项目与分区边距)。

[iOS 图片加载、解码与内存](ios/image-loading-decoding-and-memory.md)：压缩数据与像素内存、降采样与服务器缩略图、ImageIO、提前解码、缓存与并发控制。

相关专题：[长列表图片任务调度](ios/image-loading-decoding-and-memory.md#长列表图片任务调度)；[Operation 与 OperationQueue](ios/threads-and-scheduling.md#operation-与-operationqueue)。内容范围：下载与解码分别限流、预取回调、共享任务取消、快速滚动的启动控制，以及任务队列的并发限制与协作式取消。

## Android 界面与动画

[Android 分类目录](android/README.md)：4 篇文档。

| 文档 | 内容范围 |
| --- | --- |
| [ValueAnimator、插值器与滚动动画](android/valueanimator-and-interpolation.md) | 每帧执行顺序、插值器对象与动画进度、动画值、滚动距离与取消行为 |

[Android View、ViewGroup 与 XML 布局](android/view-system-and-layout.md)：View 与 ViewGroup 的职责、容器嵌套、UIKit 对照、XML 布局，以及 Android 布局预览的示例数据。

[Android 页面生命周期与状态保存](android/activity-lifecycle-and-state.md)：Activity 配置变化与重建、Jetpack ViewModel、进程结束后的状态恢复、SavedStateHandle 与异步结果。

[Activity、Fragment 与页面组织](android/activity-fragment-and-navigation.md)：Activity 与 Fragment 的职责、宿主和窗口关系、页面组合、清单声明、启动流程及 UIViewController 对照。

[Android 页面生命周期与状态保存](android/activity-lifecycle-and-state.md)：补充 Activity 的基本回调、Fragment 实例与其视图的生命周期，以及视图引用的清理时机。

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

相关专题：[耦合与解耦](engineering/design-patterns.md#耦合与解耦)；[依赖反转与依赖注入](engineering/design-patterns.md#依赖反转与依赖注入)。内容范围：耦合的含义、解耦目的与三个实例、源码依赖方向、模块边界，以及依赖反转与依赖注入的区别。

[Xcode：界面预览与 Android 布局预览](engineering/xcode-and-testing.md#界面预览与-android-布局预览)：Xcode 预览能力与 UIKit 接入条件。

## 网络与存储

| 文档 | 内容范围 |
| --- | --- |
| [网络请求与代理](network-and-storage/network-and-proxy.md) | 连接与超时、幂等重试、TCP、代理、HTTPS 抓包与证书信任 |
| [iOS 文件存储](network-and-storage/ios-file-storage.md) | Documents、Caches 和 tmp 的用途与备份边界 |

相关专题：[HTTP 请求方法与文件上传](network-and-storage/network-and-proxy.md#http-请求方法安全性与幂等性)。内容范围：HTTP 方法、安全性与幂等性、GET 与 POST、multipart 字段与 boundary、两层头部、服务端解析，以及 Swift URLSession 上传。

相关专题：[请求正文格式与 JSON 字段](network-and-storage/network-and-proxy.md#请求正文格式与-json-字段)；[使用 JSON 上传文件](network-and-storage/network-and-proxy.md#使用-json-上传文件)；[gzip 压缩与抓包正文显示](network-and-storage/network-and-proxy.md#gzip-压缩与抓包正文显示)。内容范围：JSON 与 multipart 字段表示、Base64 文件上传、正文格式与 gzip 压缩的区别，以及抓包预览与原始字节。

## 实验与统计

| 文档 | 内容范围 |
| --- | --- |
| [A/B 实验与统计](statistics/ab-testing.md) | A/B 显著性、p 值、有效样本量与结果解读 |

实验与统计补充：[使用人数少时的功能上线判断](statistics/ab-testing.md#使用人数少时的功能上线判断)。内容范围：人数与数据核对、体验价值与开发维护成本、可能损失、观察与停止条件，以及逐步开放的依据。

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
