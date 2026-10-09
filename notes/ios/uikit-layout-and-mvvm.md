# UIKit 布局、列表与 MVVM

[返回笔记目录](../README.md)

## 本文目录

- [页面生命周期与布局补充](#页面生命周期与布局补充)
- [视图层级与父子关系](#视图层级与父子关系)
- [UIView 子视图的回调与可见性](#uiview-子视图的回调与可见性)
- [容器控制器与子控制器](#容器控制器与子控制器)
- [分栏界面与详情控制器生命周期](#分栏界面与详情控制器生命周期)
- [UICollectionView 的职责与复用](#uicollectionview-的职责与复用)
- [页面、列表与 Cell ViewModel](#页面列表与-cell-viewmodel)
- [MVVM 的输入、状态输出与界面刷新](#mvvm-的输入状态输出与界面刷新)
- [长列表性能与异步图片复用](#长列表性能与异步图片复用)
- [布局更新与列表定位时机](#布局更新与列表定位时机)

## 页面生命周期与布局补充

UIViewController 的加载、出现和消失属于页面生命周期，应与 App、Scene 生命周期区分。布局回调可能重复执行，页面消失不等于对象释放，交互转场也可能取消。

命中测试通过 `hitTest` 寻找目标视图，`point(inside:)` 参与判断响应区域。响应链描述事件沿响应对象关系传递的机制，两者需要区分。

Cell 自动高度通常需要 automaticDimension、合理的预估行高、多行 Label、明确宽度，以及 contentView 顶部到底部的完整约束链。

局部刷新前先更新数据源。异步结果返回后按业务 ID 找到当前行，再在主线程更新界面；不能持续沿用可能已变化的旧索引。

Content Hugging 与 Compression Resistance 分别表达视图相对固有内容尺寸抵抗被拉大、被压小的优先级。空间富余时，较低 hugging 优先被拉大；空间不足时，较低 compression resistance 优先被压小。最终结果由整体约束共同决定，优先级改变的是视图尺寸倾向，不是文字字形。

例如标题与价格水平排列，边距和间距约束已建立，且价格本身能放入容器时，可令价格抗压缩优先级为 750、标题为 749，并让标题单行尾部省略。

## 视图层级与父子关系

视图层级是各个视图之间的父子关系和嵌套结构，整体像一棵树，也叫视图树。以详情页作简化示意，并非真实页面的完整结构：

```text
详情页根视图 UIView
├── 图片视图 UIImageView
└── 评论区容器 UIView
    ├── 评论文字 UILabel
    └── 回复按钮 UIButton
```

评论区容器是文字和按钮的父视图；文字和按钮是它的子视图，彼此互为兄弟视图。通过 `父视图.addSubview(子视图)` 建立关系；一个 UIView 可以有多个子视图，同一时刻最多有一个直接父视图。

移动评论区容器，里面的文字和按钮会跟着移动；隐藏容器，子视图也不会显示。评论文字的 `frame` 使用评论区容器的坐标系。UIView 管理视图层级，就是组织谁包含谁，以及这些视图的布局、显示和交互关系。

## UIView 子视图的回调与可见性

UILabel、UITextView 都是 UIView 的子类。普通子视图没有 viewWillAppear、viewDidAppear 等 UIViewController 的出现回调。UIView 提供视图层级变化回调，但“加入视图层级”和“用户实际看见内容”是不同的事件。

**控制器与视图的职责。**UIViewController 通过 view 属性持有根视图，并管理根视图及其子视图。UIKit 根据页面展示过程调用控制器的生命周期方法；存在容器控制器时，容器控制器还负责协调子控制器的出现和消失。一个界面可以由多个控制器共同组成，并非必须只对应一个控制器。

| UIView 回调 | 触发含义 |
|-|-|
| `willMove(toSuperview:)` | 父视图即将改变。参数表示新的父视图；移除时可以为 nil。 |
| `didMoveToSuperview()` | 父视图已经改变。此时可读取 superview 查看新的关系。 |
| `willMove(toWindow:)` | 所属窗口即将改变。参数表示新的窗口；离开窗口层级时可以为 nil。 |
| `didMoveToWindow()` | 所属窗口已经改变。此时可读取 window 查看当前所属窗口。 |

可以在自定义 UIView、UILabel 或 UITextView 子类中重写这些方法，响应层级关系变化。这些方法不构成“每次可见都调用”的统一回调；不要假定父视图回调与窗口回调之间存在适用于所有操作的固定交错顺序。

**示例：在已经显示的页面中加入文本视图。**先创建 label，再设置文字，最后加入控制器的根视图：

`let label = UILabel()`
`label.text = "你好"`
`view.addSubview(label)`

上述操作改变 label 的父视图，因此 label 收到父视图变化回调。如果根视图已经连接到窗口，label 的所属窗口也会从 nil 变为该窗口，并收到窗口变化回调。仅仅新增 label，不会让父控制器再次收到 viewDidAppear。

**连接到窗口不等于实际可见。**`label.window != nil`只说明 label 位于某个窗口的视图层级中。label 或祖先视图被隐藏、有效透明度为零、位于屏幕外、被裁剪或被其他内容遮挡，都可能导致用户看不到文字。didMoveToWindow 也不能证明内容已经完成首帧显示。

如果仅执行 `label.isHidden = false`，且父视图和所属窗口均未改变，就不会因这次显隐切换触发上述移动回调。业务需要在“显示文字”时启动操作，可以在更新显隐状态的代码路径中显式调用对应逻辑；业务需要判断实际曝光时，还应按约定检查可见区域、遮挡和曝光时长等条件。

参考：[Apple：didMoveToSuperview()](https://developer.apple.com/documentation/uikit/uiview/didmovetosuperview())；[Apple：didMoveToWindow()](https://developer.apple.com/documentation/uikit/uiview/didmovetowindow())。

## 容器控制器与子控制器

容器控制器是一种 UIViewController，负责管理其他 UIViewController，并组织这些子控制器的视图布局、显示和切换。被容器控制器管理的控制器称为子控制器。子控制器仍然管理自己的根视图和内部子视图。

| UIKit 标准容器控制器 | 职责 |
|-|-|
| `UINavigationController` | 管理导航栈，通过 push、pop 等操作切换页面。 |
| `UITabBarController` | 管理多个 Tab 对应的控制器，根据选中项显示对应页面。 |
| `UISplitViewController` | 管理分栏界面中的控制器，并根据显示模式组织各栏。 |

**示例：从首页切换到信息流页。**信息流页用于连续展示帖子或内容卡片，本例采用瀑布流布局。以下为通用示例：UITabBarController 管理 HomeViewController 和 FeedViewController；两个子控制器分别管理首页根视图和信息流页根视图。

假设 UITabBarController 已经显示在屏幕上，用户从首页切换到信息流页，且切换正常完成。容器控制器切换展示的子控制器视图，并协调出现与消失回调：

- HomeViewController 收到 viewWillDisappear，随后收到 viewDidDisappear。
- FeedViewController 收到 viewWillAppear，随后收到 viewDidAppear。

上述列表只表达每个控制器内部的 will → did 顺序，不表示两个控制器的回调完全串行执行。如果信息流页的根视图尚未加载，UIKit 会在展示前按需加载该视图，并调用 viewDidLoad；后续 Tab 切换不会仅因页面再次出现就重复触发 viewDidLoad。

**视图父子关系与控制器父子关系需要分别建立。**addSubview 建立 UIView 之间的父子关系。自定义容器通过 addChild 等容器 API 建立 UIViewController 之间的父子关系，并把子控制器的根视图放入容器的视图层级。只调用 addSubview(child.view)，并不等于完成控制器嵌入。

UIKit 标准容器已经实现相应的生命周期协调。自定义容器必须遵守控制器嵌入与出现回调转发规则。因此，子控制器可以收到 viewWillAppear 等控制器回调，而普通 UILabel 子视图不会因为属于某个控制器的视图层级，就获得这些回调。

参考：[Apple：UIViewController 与容器控制器](https://developer.apple.com/documentation/uikit/uiviewcontroller)。

## 分栏界面与详情控制器生命周期

### 界面外观与各栏分工

典型的分栏界面会同时显示“左侧列表”和“右侧详情”。下图是苹果官方的 iPad 邮件截图，用于展示这种界面外观；下面的控制器分工是使用 UISplitViewController 实现同类界面的通用示例。

![图片展示了iPad邮件界面，左侧为邮件列表，显示“所有收件箱”，有Jenny Court、Xiaomeng Zhong等人的邮件，部分邮件被蓝色框选。右侧是选中邮件“Neighborhood garden”的正文，内容为关于邻里花园规划的讨论。图片上方标注了三个功能区域：左上角“Change mailboxes or accounts.”，右上角“Delete, move, or mark multiple messages.”，右下角“Move message to another mailbox.”，与上下文介绍的界面外观及控制器分工相呼应。](../assets/ipad-mail-split-view.png)

图示：左侧选择邮件，右侧显示选中邮件的正文。图片来源：[Apple《在 iPad 上的“邮件”中查看电子邮件》](https://support.apple.com/guide/ipad/check-your-email-ipad99a3ef9e/ipados)。

| 对象 | 职责 |
|-|-|
| 左栏子控制器 | 展示邮件列表，响应用户选择某封邮件的操作。 |
| 右栏子控制器 | 展示选中邮件的标题、发件人和正文。 |
| `UISplitViewController` | 管理各栏的子控制器，组织各栏的尺寸、位置和显示状态。 |

用户选中邮件后，业务代码负责把选中的邮件数据传给详情控制器。UISplitViewController 不负责根据邮件数据生成正文。可用宽度不足时，分栏界面可以折叠成一次显示一栏。

### 点击邮件是否触发生命周期回调

点击事件本身不决定详情控制器是否收到出现回调。判断依据是：点击后是否加载了详情控制器的根视图，以及详情控制器的视图是否实际出现或消失。这里的出现回调指 viewWillAppear 和 viewDidAppear。

| 点击后的操作与条件 | 详情控制器的回调 |
|-|-|
| 复用右侧始终可见的详情控制器，只更新邮件内容。 | 不会仅因数据更新就再次触发 viewDidLoad、viewWillAppear 或 viewDidAppear。 |
| 右栏可见，将当前详情控制器替换为另一个控制器，并完成展示。 | 旧控制器收到消失回调，新控制器收到出现回调。新控制器的根视图若尚未加载，会在加载时先收到 viewDidLoad。 |
| 窄屏状态下，从邮件列表进入此前未显示的详情页面。 | 详情控制器收到出现回调，即使复用了已有控制器。只有根视图需要加载时，才会触发 viewDidLoad。 |

**复用并刷新内容的流程：**用户点击邮件 → 业务代码传递选中的邮件数据 → 已显示的详情控制器刷新标题、发件人和正文。控制器持续显示在右栏，因此不会仅因内容改变而重新触发出现回调。

内容更新如果引发布局，仍可能触发 viewWillLayoutSubviews、viewDidLayoutSubviews 等布局回调。布局回调与出现回调需要区分。viewDidLoad 对应根视图完成加载，也不是“每次选中一封邮件”都会调用的方法。

### 示例：用新的详情控制器替换旧对象

假设分栏界面正在显示，右栏可见，当前的 detailA 负责显示邮件 A。用户点击邮件 B 后，业务代码创建另一个详情控制器 detailB，再用 detailB 替换右栏的 detailA。

以下为示意代码：mailB 表示选中的邮件 B；MailDetailViewController 是自定义的邮件详情控制器，其初始化方法接收邮件数据；splitViewController 表示当前的 UISplitViewController。

`let detailB = MailDetailViewController(mail: mailB)`
`splitViewController.setViewController(detailB, for: .secondary)`

在这个双栏示例中，.secondary 表示承载详情的右栏。上述调用将 detailB 安装到该栏。假设替换正常完成，各对象的回调如下：

| 控制器对象 | 回调 |
|-|-|
| `detailA` | viewWillDisappear → viewDidDisappear。 |
| `detailB` | 根视图首次加载时调用 viewDidLoad；随后在显示过程中调用 viewWillAppear → viewDidAppear。 |

表格只表达每个对象内部的先后关系，不表示两个对象的回调完全串行执行。detailA 和 detailB 可以属于同一个 MailDetailViewController 类，但属于两个不同的对象。

仅创建控制器对象不代表该对象的视图已经出现。出现回调对应视图的展示过程；根视图加载则可能发生在展示前，例如业务代码提前访问了 view。这里讨论的是通用实现方式，不能仅凭邮件截图判断实际 App 是否每次都创建新的详情控制器。

参考：[Apple：UISplitViewController](https://developer.apple.com/documentation/uikit/uisplitviewcontroller)；[Apple：UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)。

## UICollectionView 的职责与复用

UICollectionView 展示一组可滚动内容。Cell 展示单项；DataSource 提供数量与配置好的 Cell；Delegate 处理点击等交互；Layout 提供位置、尺寸等布局信息。数据按 section／item 组织，IndexPath 表示当前位置，不是永久业务身份。

基本流程：创建并设置布局 → 注册 Cell → 设置数据源和代理 → 按需获取、配置 Cell → 根据布局信息展示。随着滚动复用 Cell，不为所有数据一次性创建视图；每次配置应覆盖完整状态，避免残留旧内容。

SnapKit 可负责 UICollectionView 在页面中的位置，以及 Cell 内控件约束。Cell 在列表中的排列由 UICollectionViewLayout 负责，例如 FlowLayout、CompositionalLayout 或自定义 Layout。

## 页面、列表与 Cell ViewModel

MVVM 没有规定一个页面只能有一个 VM，也没有规定每个 View 都必须有 VM。划分依据是是否存在需要独立管理的展示数据、状态与交互职责。

| 范围 | 常见职责 |
|-|-|
| 页面／容器 VM | 整体展示状态、子区域组织与协调 |
| 列表 VM | 列表项数组、加载、刷新、分页、插入和删除 |
| Cell VM | 单项展示数据、样式、选中／高亮等状态及交互处理 |

本项目评论区容器 VM 持有输入框 VM、引导 VM 和列表 VM；列表 VM 的 `cellVMs` 数组中，每条评论以及展开更多回复项都有对应 VM。列表级逻辑管理“有哪些项、如何排序和更新”，单项 VM 管理该项的数据与展示状态。

复杂评论项需要管理内容、一级／二级样式、高亮、Cell 类型与高度，以及点赞／回复／删除等处理入口，拆分 Cell VM 有助于明确职责。如果只是展示固定标题或简单静态文本、没有额外状态与处理逻辑，直接配置 View 也可以，不必为了 MVVM 名称增加 VM。上述是划分原则，不表示每种界面都必须照搬本项目层级。

**VM 对应数据项，Cell 是可复用的视图对象。** 100 条数据各有 VM，不意味着立即创建 100 个 Cell。同一个 Cell 滚动后可以绑定另一项的 VM，绑定时需要更新展示状态。框架布局、视图创建和动画执行仍由 View 等 UI 层承担。

代码核查日期：2026-10-01。

## MVVM 的输入、状态输出与界面刷新

MVVM 按职责区分三类对象：Model 表达业务数据和业务规则；View 展示界面并接收用户操作；ViewModel 处理界面输入，并把业务结果转换为 View 可展示的状态。MVVM 没有规定必须使用某一种绑定框架。

一种常见通信流程是：用户操作 → View 调用闭包或 ViewModel 方法 → ViewModel 调用业务服务 → ViewModel 根据处理结果更新展示状态 → Publisher 发布状态 → View 的订阅闭包刷新界面。业务服务可以负责网络请求或持久化，View 无需理解这些操作的具体实现。

输入和输出承担不同职责。输入表达用户希望执行的动作，可以使用方法、闭包或事件流。输出表达当前展示状态或需要执行的界面事件，可以使用 `@Published`、Subject 或其他观察机制。闭包和 Publisher 是实现通信的工具，不是 MVVM 的定义。

更新图标、文字、布局和动画由 View 执行。ViewModel 决定需要展示的数据与状态。异步结果到达后，View 应在主线程执行 UIKit 界面更新；在响应式链路中，可以在刷新界面前安排 `receive(on:)`。

View 持有协议类型的 ViewModel，可以降低 View 对某个具体实现类的依赖，但 View 仍然依赖协议约定。使用闭包或 Publisher 也不会自动消除全部依赖，还需要明确事件入口、状态来源和订阅生命周期。

语言细节见[Swift 语言基础](../language/swift-basics.md)；状态订阅与执行时序见[响应式编程与事件通知](reactive-programming.md)。

## 长列表性能与异步图片复用

卡片总数多不代表要同时创建一两千个 Cell。复用使视图数量主要受可见区域和预取范围影响；但数据模型、图片缓存、布局计算和未取消的异步任务仍可能累积。

- CPU／主线程：反复配置 Cell、文本测量、自适应布局、图片解码或缩放、大量通知和 reload，可能赶不上帧预算。60 Hz 约 16.7 ms、120 Hz 约 8.3 ms，是参考时间预算，不代表业务可以全部占满。
- 内存：大图解码成位图后明显膨胀；动画帧缓存、预取结果和模型可能不断增长，引发内存压力。文件小不代表解码后占用小。
- 网络／调度：高速滚动会启动大量马上离屏的请求，浪费带宽、解码和回调成本；旧请求完成后还可能把图片写进已复用的 Cell。
- GPU／渲染：大量混合、复杂阴影、遮罩、过度绘制或昂贵图层效果，也可能造成卡顿，不能只看主线程 CPU。

优化时先测量瓶颈，再选择：复用 Cell 并完整覆盖状态；避免滚动中全量刷新；稳定内容按 ID 缓存布局结果；按展示像素尺寸请求缩略图并下采样；在合适的后台路径解码；使用有容量限制的内存／磁盘缓存；合并重复请求、限制并发、预取即将可见内容并取消无用预取；离屏暂停动画／视频；合并同一帧内的 UI 更新。

Cell 复用主要减少视图创建／销毁、重复布局对象和内存占用，不会自动取消网络请求或清空旧状态。异步赋图前需要在主线程检查当前绑定的内容 ID／请求标识，再连续完成赋值；只保留 Cell 引用并不能保证它仍代表原来的数据。

## 布局更新与列表定位时机

| 方法 | 作用 | 常见使用方式 |
|-|-|-|
| `setNeedsLayout()` | 标记需要重新布局，允许系统合并更新，调用时通常不立即执行布局 | 内容或布局条件改变后请求后续布局 |
| `layoutIfNeeded()` | 如果有待处理布局更新，立即处理相关视图子树；没有则无需重复布局 | 修改约束后，需要获得当前几何信息或构造布局动画 |
| `layoutSubviews()` | UIKit 执行布局时调用的回调，可重写以布置子视图 | 自定义 UIView 内实现布局；通常不直接调用它来强制刷新 |

改动约束可能影响父视图和兄弟视图，调用 layoutIfNeeded 的对象应覆盖相应布局范围。这三者讨论的是布局，不能直接等同绘制完成或实际像素上屏。上屏边界另见[渲染管线与显示阶段](rendering-and-performance.md#图层动画与渲染管线)。

**reload 后定位：**

`reloadData` 会触发数据重新加载与布局更新，但调用返回不代表所有自适应高度、布局属性和后续图片尺寸都已稳定。立即按旧的 contentSize／Cell 位置滚动，就可能算错目标。

`DispatchQueue.main.async` 只保证排队执行，不能保证一定跨到下一轮 RunLoop，也不构成“布局已经完整结束”的契约。应根据数据更新方式选择合适的完成节点，在相关父／祖先视图上按需完成布局，再按最新布局定位；若异步图片改变尺寸，仍需要明确后续布局完成条件。不能把无限追加 async、固定延迟或 layoutIfNeeded 当成等待所有异步资源的通用方法。
