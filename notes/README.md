# 知识笔记

[返回 Study](../README.md)

按 10 个主题整理的 61 篇学习笔记，内容限于通用概念、工作机制和独立示例。每篇笔记开头有本文目录，可以直接跳到具体章节。

## 分类目录

| 分类 | 文档数 |
| --- | --- |
| [语言与运行时](#语言与运行时) | 11 |
| [iOS 界面、事件与调度](#ios-界面事件与调度) | 13 |
| [Android 应用开发](android/README.md) | 14 |
| [跨平台与宿主通信](#跨平台与宿主通信) | 3 |
| [软件设计与工程工具](#软件设计与工程工具) | 5 |
| [网络与存储](#网络与存储) | 8 |
| [实验与统计](#实验与统计) | 1 |
| [AI 辅助开发](#ai-辅助开发) | 2 |
| [数据结构](data-structures/README.md) | 1 |
| [操作系统基础](operating-systems/README.md) | 3 |

## 语言与运行时

| 文档 | 内容范围 |
| --- | --- |
| [Swift 基础语法与类型系统](language/swift-basics.md) | 类型安全与动态能力、类型推断与数值转换、运行时类型转换；可选类型与可选绑定；函数签名、回调类型与错误处理；逃逸闭包、捕获列表与可选回调；协议继承、some 与 any；符号引用与对象引用、声明与定义；Swift 与 C 接口互操作；多行字符串、Data 与字符串解码、元组与 Void、defer |
| [Swift 内存管理、值语义与对象引用](language/swift-memory-and-references.md) | 类型、实例与对象引用；值语义、值类型与引用类型的选择、mutating、函数参数、inout 与独占访问、值的生命周期；ARC 与 Android GC、编译器插入 retain 与 release；weak 与 unowned；闭包与 Delegate；浅拷贝与深拷贝；相等与对象身份 |
| [Swift 并发：actor、Sendable 与 Task](language/swift-concurrency.md) | actor 的默认隔离与重入、MainActor 与主队列；Sendable 与 @Sendable；Swift 6 并发检查与版本条件；Task 与 Task.detached、任务结果类型、上下文继承与取消、async let、TaskLocal |
| [Swift 属性、初始化方法与扩展限制](language/swift-properties-and-initialization.md) | 存储属性、计算属性与访问器、lazy 属性；实例属性初始化、指定与便利初始化方法、required 与 override、init?(coder:)、两阶段初始化；扩展中的类型属性与实例存储限制 |
| [Swift 集合、KeyPath 与写时复制](language/swift-collections.md) | map 与 KeyPath、Sequence、Collection 与 Set、for-in 遍历、stride、last 与 reversed、数组值语义与写时复制、ArraySlice、集合与泛型协议约束 |
| [Swift Codable 与序列化复制](language/swift-codable-and-serialization.md) | Codable 编码与解码、CodingKeys，以及编码再解码复制对象时的状态保留、对象共享与循环引用问题 |
| [Swift / Objective-C 运行时与方法派发](language/swift-objc-runtime.md) | 分类与类扩展、自动释放池、+load 与 +initialize、关联对象、实例布局与类型元数据、isa、静态派发与虚方法表、内联与编译优化选项、协议见证表、Selector 与 IMP、消息转发、Method Swizzling、@objcMembers、KVC 与 KVO |
| [TypeScript、JavaScript 与 Worker 线程通信](language/typescript-javascript-and-workers.md) | Node.js 与浏览器、类型标注与推断、number 与 Int／Double、接口与泛型、类型擦除、事件循环与异步、Worker 消息通信及回调执行位置 |
| [C++ 静态存储期对象与初始化](language/cpp-static-storage-and-initialization.md) | 静态存储期、静态数据成员、构造函数、全局对象与静态局部对象的初始化时机 |
| [Kotlin 基础语法与 Swift 对照](language/kotlin-basics.md) | 变量与空安全、val 与 const val、Swift let 对照、Array／List／MutableList 与引用语义、类与继承、data class、object、companion object、sealed class、函数、扩展函数与 lambda、when 表达式、作用域函数 |
| [Kotlin 协程与 suspend](language/kotlin-coroutines-and-suspend.md) | 协程与挂起函数、启动与执行顺序、挂起与阻塞，以及与 Swift Task、async、await 的对应关系 |

## iOS 界面、事件与调度

| 文档 | 内容范围 |
| --- | --- |
| [Darwin、应用框架与系统服务](ios/darwin-frameworks-and-services.md) | Darwin 与 XNU、基础系统组件、应用框架、系统服务及通知协作示例 |
| [iOS 应用生命周期、启动与窗口](ios/app-lifecycle-and-events.md) | 应用运行状态与生命周期回调、Scene 生命周期、启动到首帧、LaunchScreen 与自定义启动页、main 之前的 dyld 工作、启动计时与启动优化、动态库数量与地址修正；多个 UIWindow 与关键窗口 |
| [iOS 后台执行机制](ios/background-execution.md) | 挂起与后台执行的前提、beginBackgroundTask、后台模式、BGTaskScheduler、后台 URLSession、静默推送，以及与 Android 后台执行方式的对照 |
| [UIKit 触摸事件、手势与键盘输入](ios/touch-events-and-gestures.md) | 触摸事件分发路径、局部坐标与 bounds、hitTest 命中测试、事件穿透、扩大点击区域；响应链与触摸回调；手势同时识别、点击与长按协调、长按状态；屏幕键盘与第一响应者 |
| [UIKit 布局、列表与 MVVM](ios/uikit-layout-and-mvvm.md) | 控制器初始化与视图加载、push 与 present、sheet 样式、viewDidLayoutSubviews、UIButton 与 tintColor；frame 与 Auto Layout、transform、autoresizingMask、固有内容尺寸与优先级、约束复用；视图层级与前后顺序、子视图回调、容器控制器、分栏界面；UITableView 重用、Cell 倒计时、行删除与批量更新、UICollectionView；Cell ViewModel、MVVM、长列表性能与高度缓存、reload 后定位 |
| [响应式编程与事件通知](ios/reactive-programming.md) | NotificationCenter；OpenCombine 的 Subject、@Published、CurrentValueSubject、receive(on:)、removeDuplicates；CombineX 的订阅关系；RxSwift 的 Observable、debounce、distinctUntilChanged、observe(on:)、flatMapLatest 与 flatMap |
| [iOS 线程、GCD 与同步](ios/threads-and-scheduling.md) | 进程与线程、逻辑 CPU 与线程状态、线程接口与任务调度接口；数据竞争、i++ 与线程安全方式、CAS 与 ABA、锁的分类与等待方式、优先级反转；GCD 的 sync 与 async、barrier 单写多读、死锁；DispatchGroup 与 DispatchSemaphore；OperationQueue 的依赖、取消与异步任务限流 |
| [RunLoop、Timer 与 CADisplayLink](ios/runloop-timer-and-displaylink.md) | RunLoop 与运行模式、Source0 与 Source1、Timer 的调度与重复触发、倒计时与单调时钟、CADisplayLink 的回调时机与动画进度、Timer 与 CADisplayLink 的停止与引用、弱引用代理 |
| [Core Animation 渲染与性能分析工具](ios/rendering-and-performance.md) | 图层动画与渲染管线、anchorPoint 与 position、UIViewPropertyAnimator、图层重用、圆角裁剪与离屏渲染、shouldRasterize、卡顿监测、Time Profiler、Animation Hitches 与 MetricKit |
| [iOS 图片加载、解码与内存](ios/image-loading-decoding-and-memory.md) | 共享加载服务、压缩数据与像素内存、降采样与 ImageIO、提前解码、缓存与释放、长列表图片任务调度、过期回调校验；Kingfisher 与 SDWebImage |
| [iOS 进程间通信与应用扩展](ios/interprocess-communication-and-extensions.md) | App Extension、App Groups 共享目录、Darwin 通知、UNIX domain socket，以及 XPC 的使用条件 |
| [iOS 内存泄漏与无用对象排查](ios/memory-leak-diagnosis.md) | 预期释放时机、Allocations 分代筛选、Memory Graph 强引用追踪、分配调用栈与 Leaks，以及修改后的复测 |
| [iOS 本地多语言与即时切换](ios/localization-and-language-switching.md) | 文案键、.strings 与 String Catalog、本地 Bundle 选择、语言偏好保存、通知刷新已有界面，以及 Locale 的语言与地区 |

## Android 应用开发

14 篇文档，列表见 [Android 分类目录](android/README.md)。

## 跨平台与宿主通信

| 文档 | 内容范围 |
| --- | --- |
| [WebView 与 Lynx 双向通信](cross-platform/webview-and-lynx-bridge.md) | WebView 和 Lynx 的消息、原生模块注册、请求回调与事件通知 |
| [Lynx 渲染、线程模型与资源更新](cross-platform/lynx-rendering-and-updates.md) | Lynx 渲染、双线程、首帧直出、后台接管与动态资源更新 |
| [KMP 跨平台编译与鸿蒙库集成](cross-platform/kmp-compilation-and-harmony-libraries.md) | 共享源码与平台产物、源码依赖与已发布库、.so、ArkTS、HAR，以及接口和版本匹配 |

## 软件设计与工程工具

| 文档 | 内容范围 |
| --- | --- |
| [设计模式与 Swift 实践](engineering/design-patterns.md) | 耦合与解耦、依赖反转与依赖注入，策略、工厂、单例、建造者模式及 Swift 示例 |
| [Xcode 工程管理与自动化测试](engineering/xcode-and-testing.md) | 源码搜索与索引、界面预览、第三方库接入；编译、链接、嵌入与装载，静态与动态链接、Rebase 与 Binding、IPA 与 Mach-O、ASLR、包体积优化；XCTest、Swift Testing、测试计划与 UI 测试 |
| [CocoaPods、Podfile 与依赖管理](engineering/cocoapods-and-podfile.md) | Ruby DSL、依赖声明与版本范围、Molinillo 与 Xcodeproj、锁文件、YAML、Gemfile，以及用 Python 读取 Podfile |
| [iOS 组件化：职责、接口与依赖](engineering/ios-modularization.md) | 组件边界、协议协作、模块与仓库、SDK 与 API、源码与二进制交付、XCFramework、Swift Package／Framework／CocoaPods 的用途，以及收益和维护成本 |
| [Gradle 构建脚本与 .kts 文件](engineering/gradle-build-scripts.md) | 构建脚本、.kt 与 .kts 的区别，以及 Groovy 与 Kotlin 两种写法 |

## 网络与存储

| 文档 | 内容范围 |
| --- | --- |
| [网络请求与代理](network-and-storage/network-and-proxy.md) | 连接与超时、幂等重试、TCP、代理、HTTPS 抓包与证书信任；HTTP 方法与文件上传、请求正文格式、gzip；URLSession、会话配置、重定向与 URLCache；Alamofire 的请求与响应模型 |
| [iOS 文件存储](network-and-storage/ios-file-storage.md) | Documents、Application Support、Caches 和 tmp 的用途、清理与备份规则 |
| [iOS 数据持久化](network-and-storage/ios-data-persistence.md) | UserDefaults、文件、Keychain、SQLite、Core Data 与 SwiftData；数据库事务、ACID 与崩溃恢复；SQLite 建表约束、参数绑定、WAL 并发和数据库迁移 |
| [数据库索引与 B+ 树](network-and-storage/database-index-and-btree.md) | 索引与全表扫描、B+ 树与 B 树、InnoDB 与自适应哈希索引、聚簇索引与回表、覆盖索引、联合索引与最左前缀、索引下推、索引失效、查询计划与深分页 |
| [SQL 多表查询：JOIN 与 UNION](network-and-storage/sql-join-and-union.md) | 各种连接与全外连接的替代写法、别名、UNION 与 UNION ALL、ON 与 WHERE、子句的逻辑执行顺序、嵌套循环连接与驱动表 |
| [Android 数据存储：私有目录、键值存储与数据库](network-and-storage/android-data-storage.md) | filesDir 与 cacheDir 的用途、删除时机与备份规则，SharedPreferences 的 apply 与 commit、DataStore、Room，以及与 iOS 存储方式的对照 |
| [数据库范式：第一、第二、第三范式](network-and-storage/database-normal-forms.md) | 主键、候选键与函数依赖，第一范式，冗余与更新、插入、删除异常，部分依赖与第二范式，传递依赖与第三范式，拆表后的连接查询与反范式 |
| [MySQL 事务并发、锁与日志](network-and-storage/mysql-transactions-and-logs.md) | 并发异常与隔离级别、MVCC、快照读与当前读、索引锁、乐观锁与悲观锁、redo log 与 binlog |

## 实验与统计

| 文档 | 内容范围 |
| --- | --- |
| [A/B 实验与统计](statistics/ab-testing.md) | 显著性、p 值与有效样本量；使用人数少时的功能上线判断；客户端精准曝光实验与服务端实验；实验参与范围、统计分母、绝对提升与相对提升 |

## AI 辅助开发

| 文档 | 内容范围 |
| --- | --- |
| [AI Agent 运行原理与上下文管理](ai/agent-principles.md) | Agent 与 Harness、工具调用循环、模型幻觉、Skill／Tool／MCP、Skill 格式与上下文控制、AGENTS.md 的加载、用项目导航定位代码 |
| [AI 辅助开发的检查与验证](ai/development-review-and-verification.md) | AI 辅助开发的流程、判断条件、技术方案检查、双端分别实现与共享实现的选择、代码审查、验证证据的范围、AI 辅助验收与设备操作、跨端 UI 一致性验证 |

## 数据结构

[HashMap 的结构与实现原理](data-structures/hash-map.md)：键值映射、哈希与相等规则、冲突链、插入查找删除、负载因子、扩容与复杂度。

## 操作系统基础

| 文档 | 内容范围 |
| --- | --- |
| [虚拟内存、按需分配与缺页异常](operating-systems/virtual-memory-and-page-faults.md) | 虚拟地址与物理内存、页表、按需分配、缺页异常及非法访问 |
| [macOS 隔离标记](operating-systems/macos-quarantine.md) | 隔离标记的含义、首次打开时的影响，以及查看与移除 |
| [内核、用户态与系统调用](operating-systems/kernel-and-user-mode.md) | 内核的职责、用户态与内核态、系统调用，以及进入内核、线程阻塞与线程切换的区别 |

## 文档锚点、内部链接与交叉引用

锚点是文档中跳转目标的标记。HTML 用 `id` 声明锚点，Markdown 链接用 `#` 加片段标识跳到这个位置。

```html
<a id="symbol-search"></a>
```

```markdown
[符号搜索](#symbol-search)
```

同一文档中的锚点 id 必须唯一。修改标题时保留原来的 id，基于这个 id 的链接仍然有效。是否支持 HTML 锚点，取决于文档渲染器。

交叉引用是在一处内容中指向另一处相关内容，本笔记之间通过相对链接互相引用。
