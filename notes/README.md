# 知识笔记

[返回 Study](../README.md)

按 10 个主题整理的 29 篇学习笔记。内容限于通用概念、工作机制和独立示例；通过篇内目录与笔记之间的内部链接查阅相关内容。

整理日期：2026-10-09。

## 分类目录

| 分类 | 文档数 |
| --- | --- |
| [语言与运行时](language/) | 5 |
| [iOS 界面、事件与调度](ios/) | 8 |
| [Android 界面与动画](android/) | 6 |
| [跨平台与宿主通信](cross-platform/) | 2 |
| [软件设计与工程工具](engineering/) | 2 |
| [网络与存储](network-and-storage/) | 2 |
| [实验与统计](statistics/) | 1 |
| [AI 辅助开发](ai/) | 1 |
| [数据结构](data-structures/) | 1 |
| [操作系统基础](operating-systems/README.md) | 1 |

## 语言与运行时

[Kotlin 协程与 suspend](language/kotlin-coroutines-and-suspend.md)：协程与挂起函数、启动与顺序执行、挂起与阻塞的区别，以及与 Swift Task、async 和 await 的对应关系。

| 文档 | 内容范围 |
| --- | --- |
| [Swift 语言基础](language/swift-basics.md) | 类型安全、可选类型、条件逗号与逻辑与、枚举与类型转换；属性与 mutating、协议泛型及 some/any、闭包捕获、weak/unowned 与委托、集合遍历、数组 last、reverse() 与 reversed()；Swift 与 C 接口互操作；Task、Task.detached、任务局部值与相关调用语法 |
| [Swift / Objective-C 运行时与方法派发](language/swift-objc-runtime.md) | 自动释放池、Page 与嵌套边界；对象布局、关联对象、方法派发、消息转发与 Method Swizzling；Objective-C 互操作、KVO、+load 与自动注册 |

[TypeScript、JavaScript 与 Worker 线程通信](language/typescript-javascript-and-workers.md)：类型标注与推断、接口与泛型、类型擦除、异步与线程、Worker 消息通信及回调执行位置。

相关专题：[协议继承与类继承的区别](language/swift-basics.md#协议继承与类继承的区别)。

相关专题：[ARC 与 Android GC](language/swift-basics.md#arc-与-android-gc引用计数可达性与引用环)。内容范围：强引用计数、GC Roots、可达性、引用环与内存泄漏。

相关专题：[结构体、枚举与值的生命周期](language/swift-basics.md#结构体枚举与值的生命周期)。内容范围：ARC 与 GC 的管理边界、值内部的强引用、作用域与生命周期，以及局部值、替换和返回值的清理规则。

相关专题：[defer 与作用域退出](language/swift-basics.md#defer-与作用域退出)。内容范围：延后执行、退出作用域、提前返回时解锁，以及函数无法退出时的执行边界。

相关专题：[实例存储属性与局部变量的初始化](language/swift-basics.md#实例存储属性与局部变量的初始化)；[为什么扩展不能添加实例存储属性](language/swift-basics.md#为什么扩展不能添加实例存储属性)。内容范围：声明与初始化、读取前赋值、可选属性的 nil、lazy 延迟初始化，以及实例存储与扩展的限制。

[C++ 静态存储期对象与初始化](language/cpp-static-storage-and-initialization.md)：静态存储期、静态数据成员、构造函数、全局对象与静态局部对象的初始化时机。

补充专题：[inout 与内存独占访问](language/swift-basics.md#inout-与内存独占访问)；[actor 的默认隔离与重入](language/swift-basics.md#actor-的默认隔离与重入)；[MainActor、主队列与异步调用](language/swift-basics.md#mainactor主队列与异步调用)；[数组值语义、写时复制与返回值](language/swift-basics.md#数组值语义写时复制与返回值)；[自定义 CoW 与引用唯一性检查](language/swift-basics.md#自定义-cow-与引用唯一性检查)；[ArraySlice 的存储保留与索引](language/swift-basics.md#arrayslice-的存储保留与索引)；[类的两阶段初始化与 self 使用时机](language/swift-basics.md#类的两阶段初始化与-self-使用时机)。

并发补充：[Task、Task.detached 与任务局部值](language/swift-basics.md#tasktaskdetached-与任务局部值)；[DispatchGroup 与 DispatchSemaphore](ios/threads-and-scheduling.md#dispatchgroup-与-dispatchsemaphore)。内容范围：任务与闭包、上下文继承、自动调度、完成通知、信号量初始值与配对规则。

并发与函数专题：[函数签名与回调类型](language/swift-basics.md#函数签名与回调类型)；[Sendable 与 @Sendable](language/swift-basics.md#sendable-与-sendable)；[Swift 6 的并发检查与版本条件](language/swift-basics.md#swift-6-的并发检查与版本条件)。内容范围：函数值、捕获检查、actor 串行执行器、MainActor，以及并发检查和执行规则的版本条件。

## iOS 界面、事件与调度

相关专题：[触摸回调与响应链转发](ios/app-lifecycle-and-events.md#触摸回调与响应链转发)；[父子视图的手势识别与同时识别](ios/app-lifecycle-and-events.md#父子视图的手势识别与同时识别)。内容范围：触摸回调的转发条件、super 与 next、识别器的观察范围和同时识别条件。

[iOS 进程间通信与应用扩展](ios/interprocess-communication-and-extensions.md)：App Extension、主要通信方式、App Groups 共享目录、Darwin 通知、UNIX domain socket 路径与数据传递，以及 XPC 的使用边界。

[Darwin、应用框架与系统服务](ios/darwin-frameworks-and-services.md)：Darwin 与 XNU 的关系、基础系统组件、应用框架、系统服务及通知协作示例。

| 文档 | 内容范围 |
| --- | --- |
| [iOS 应用启动、窗口与输入事件](ios/app-lifecycle-and-events.md) | UIApplication、AppDelegate、启动到首帧、UIWindow、触摸分发与键盘输入 |
| [UIKit 布局、列表与 MVVM](ios/uikit-layout-and-mvvm.md) | 生命周期、视图层级、布局、UITableView 重用与预取、ViewModel 与容器控制器 |
| [响应式编程与事件通知](ios/reactive-programming.md) | NotificationCenter、OpenCombine、订阅生命周期、状态与事件、调度 |
| [iOS 线程与任务调度](ios/threads-and-scheduling.md) | 进程与线程、GCD、DispatchGroup 与 DispatchSemaphore、线程安全、锁、RunLoop、Timer 与 CADisplayLink；Source0／Source1、Mach 消息 |
| [Core Animation 渲染与性能分析工具](ios/rendering-and-performance.md) | 图层动画、渲染管线、卡顿监测、Time Profiler、Animation Hitches 与 MetricKit |

相关专题：[圆角、边界裁剪与离屏渲染](ios/rendering-and-performance.md#圆角边界裁剪与离屏渲染)。内容范围：cornerRadius、masksToBounds、clipsToBounds、离屏渲染条件、shadowPath，以及圆角图片与外部阴影的组合。

布局与图层专题：[viewDidLayoutSubviews 的调用时机与用途](ios/uikit-layout-and-mvvm.md#viewdidlayoutsubviews-的调用时机与用途)；[anchorPoint、position 与图层几何](ios/rendering-and-performance.md#anchorpointposition-与图层几何)。内容范围：重复布局回调、按 bounds 更新图层、锚点与 position 的关系，以及改变锚点时的位移。

相关专题：[内核、逻辑 CPU 与线程并发](ios/threads-and-scheduling.md#内核的职责与进程信息)；[进程创建时间与启动计时起点](ios/app-lifecycle-and-events.md#进程创建时间与启动计时起点)。

相关专题：[获取锁失败后的等待行为](ios/threads-and-scheduling.md#获取锁失败后的等待行为)；[局部坐标、命中测试与点击长按协调](ios/app-lifecycle-and-events.md#uiview-的局部坐标与-bounds)；[UICollectionView 分区、重点回调与布局协议](ios/uikit-layout-and-mvvm.md#分区项目与分区边距)。

[iOS 图片加载、解码与内存](ios/image-loading-decoding-and-memory.md)：共享加载服务、压缩数据与像素内存、降采样与 ImageIO、提前解码、缓存引用与释放、共享任务取消、过期回调校验、内存预算与验证。

相关专题：[长列表图片任务调度](ios/image-loading-decoding-and-memory.md#长列表图片任务调度)；[Operation 与 OperationQueue](ios/threads-and-scheduling.md#operation-与-operationqueue)。内容范围：下载与解码分别限流、预取回调、共享任务取消、快速滚动的启动控制，以及任务队列的并发限制与协作式取消。

相关专题：[main 之前的启动工作与 dyld](ios/app-lifecycle-and-events.md#main-之前的启动工作与-dyld)；[UIApplication 与主线程 RunLoop](ios/app-lifecycle-and-events.md#uiapplication-与主线程-runloop)。内容范围：pre-main 初始化、UIApplication 单例、主事件循环与主线程事件处理。

补充专题：[优先级反转与优先级继承](ios/threads-and-scheduling.md#优先级反转与优先级继承)。

相关专题：[事件穿透与重叠视图](ios/app-lifecycle-and-events.md#事件穿透与重叠视图的触摸目标)、[启动优化的三个方向](ios/app-lifecycle-and-events.md#启动优化的三个方向)、[约束复用与布局计算](ios/uikit-layout-and-mvvm.md#约束对象的创建激活与复用)、[Cell 高度缓存](ios/uikit-layout-and-mvvm.md#不同高度-cell-的测量与缓存)、[SDWebImage 两级缓存与 UIImage 解码](ios/image-loading-decoding-and-memory.md#sdwebimage-的图片加载与两级缓存)。

## Android 界面与动画

[Android 分类目录](android/README.md)：6 篇文档。

[Android 触摸事件与 View 事件处理](android/touch-events-and-view-dispatch.md)：MotionEvent 与触摸序列、ACTION_DOWN 用法、监听器返回值、OnTouchListener 与 onTouchEvent 的调用关系，以及取消处理与 UIKit 对照。

[Android 列表复用：ListView、RecyclerView 与 ViewHolder](android/listview-recyclerview-and-viewholder.md)：ListView 与 RecyclerView 的区别、视图复用、ViewHolder、创建与绑定流程、局部更新及性能边界。

| 文档 | 内容范围 |
| --- | --- |
| [ValueAnimator、插值器与滚动动画](android/valueanimator-and-interpolation.md) | 每帧执行顺序、插值器对象与动画进度、动画值、滚动距离与取消行为 |

[Android View、ViewGroup 与 XML 布局](android/view-system-and-layout.md)：View 与 ViewGroup 的职责、容器嵌套、UIKit 对照、XML 布局，以及 Android 布局预览的示例数据。

[Android 页面生命周期与状态保存](android/activity-lifecycle-and-state.md)：Activity 配置变化与重建、Jetpack ViewModel、进程结束后的状态恢复、SavedStateHandle 与异步结果。

布局补充：[常见布局容器及与 UIKit 的对照](android/view-system-and-layout.md#常见布局容器及与-uikit-的对照)。内容范围：LinearLayout、FrameLayout、ConstraintLayout、RelativeLayout 与 XML 的关系。

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

相关专题：[编译、链接、嵌入与装载](engineering/xcode-and-testing.md#编译链接嵌入与装载)；[静态库的目标文件提取与代码裁剪](engineering/xcode-and-testing.md#静态库的目标文件提取与代码裁剪)；[动态库的打包、启动装载与延迟装载](engineering/xcode-and-testing.md#动态库的打包启动装载与延迟装载)。

相关专题：[IPA、.app 与 Mach-O](engineering/xcode-and-testing.md#ipaapp-与-mach-o)；[Mach-O 的加载信息](engineering/xcode-and-testing.md#mach-o-的加载信息)；[预设虚拟地址、ASLR 与物理内存](engineering/xcode-and-testing.md#预设虚拟地址aslr-与物理内存)。内容范围：安装分发包与应用目录、可执行文件的加载信息、启动地址随机化，以及虚拟地址与物理地址的映射。

相关专题：[Swift 编译优化选项与体积](language/swift-objc-runtime.md#swift-编译优化选项与体积)；[iOS 包体积优化](engineering/xcode-and-testing.md#ios-包体积优化)。内容范围：`-Osize` 与内联、Dead Code Stripping、资源与依赖精简、符号与 `dSYM`、资源变体和 App Thinning。

## 网络与存储

| 文档 | 内容范围 |
| --- | --- |
| [网络请求与代理](network-and-storage/network-and-proxy.md) | 连接与超时、幂等重试、TCP、代理、HTTPS 抓包与证书信任 |
| [iOS 文件存储](network-and-storage/ios-file-storage.md) | Documents、Caches 和 tmp 的用途与备份边界 |

相关专题：[HTTP 请求方法与文件上传](network-and-storage/network-and-proxy.md#http-请求方法安全性与幂等性)。内容范围：HTTP 方法、安全性与幂等性、GET 与 POST、multipart 字段与 boundary、两层头部、服务端解析，以及 Swift URLSession 上传。

相关专题：[请求正文格式与 JSON 字段](network-and-storage/network-and-proxy.md#请求正文格式与-json-字段)；[使用 JSON 上传文件](network-and-storage/network-and-proxy.md#使用-json-上传文件)；[gzip 压缩与抓包正文显示](network-and-storage/network-and-proxy.md#gzip-压缩与抓包正文显示)。内容范围：JSON 与 multipart 字段表示、Base64 文件上传、正文格式与 gzip 压缩的区别，以及抓包预览与原始字节。

相关专题：[URLSession 请求流程](network-and-storage/network-and-proxy.md#urlsession-的职责与请求流程)、[会话配置与连接条件](network-and-storage/network-and-proxy.md#urlsessionconfiguration-与连接条件)、[代理与重定向](network-and-storage/network-and-proxy.md#urlsessiondelegate-与重定向决定)、[URLCache 与 HTTP 缓存规则](network-and-storage/network-and-proxy.md#urlcache-与-http-缓存规则)。

## 实验与统计

| 文档 | 内容范围 |
| --- | --- |
| [A/B 实验与统计](statistics/ab-testing.md) | A/B 显著性、p 值、有效样本量与结果解读 |

实验与统计补充：[使用人数少时的功能上线判断](statistics/ab-testing.md#使用人数少时的功能上线判断)。内容范围：人数与数据核对、体验价值与开发维护成本、可能损失、观察与停止条件，以及逐步开放的依据。

实验与统计补充：[客户端精准曝光实验与服务端实验](statistics/ab-testing.md#客户端精准曝光实验与服务端实验)；[实验参与范围与统计结果](statistics/ab-testing.md#实验参与范围与统计结果)。内容范围：基本术语、两种实验的区别、配置获取与曝光流程，以及统计分母、绝对提升与相对提升。

## AI 辅助开发

| 文档 | 内容范围 |
| --- | --- |
| [AI Agent 原理与开发验证](ai/agents-and-development.md) | 模型与 Agent、工具调用、Skill / Tool / MCP、代码审查与开发验证 |

相关专题：[Codex 的 AGENTS.md 加载与代码定位](ai/agents-and-development.md#codex-的-agentsmd-加载与代码定位)。内容范围：启动指令加载、按需读取、分层导航、语义匹配与源码验证。

## 数据结构

[HashMap 的结构与实现原理](data-structures/hash-map.md)：键值映射、哈希与相等规则、冲突链、插入查找删除、负载因子、扩容与复杂度。

## 操作系统基础

[操作系统基础目录](operating-systems/README.md)：1 篇文档。

[虚拟内存、按需分配与缺页异常](operating-systems/virtual-memory-and-page-faults.md)：虚拟地址与物理内存、页表、按需分配、缺页异常及非法访问。

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
