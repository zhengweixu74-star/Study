# UIKit 布局、列表与 MVVM

[返回笔记目录](../README.md)

## 本文目录

- [页面生命周期与布局补充](#页面生命周期与布局补充)

- [viewDidLayoutSubviews 的调用时机与用途](#viewdidlayoutsubviews-的调用时机与用途)
- [frame、Auto Layout 与 SnapKit](#frameauto-layout-与-snapkit)
- [固有内容尺寸、抗压缩与约束优先级](#固有内容尺寸抗压缩与约束优先级)
- [约束对象的创建、激活与复用](#约束对象的创建激活与复用)
- [布局计算为什么可能耗时](#布局计算为什么可能耗时)
- [视图层级与父子关系](#视图层级与父子关系)
- [UIView 子视图的回调与可见性](#uiview-子视图的回调与可见性)
- [容器控制器与子控制器](#容器控制器与子控制器)
- [分栏界面与详情控制器生命周期](#分栏界面与详情控制器生命周期)
- [UITableView 的重用机制](#uitableview-的重用机制)
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

### viewDidLayoutSubviews 的调用时机与用途

`viewDidLayoutSubviews()` 是 `UIViewController` 的布局回调。UIKit 在控制器的根视图完成一次子视图布局后调用该方法。布局指根据当前尺寸、约束或手动布局代码，计算和更新子视图的位置与大小。

一次相关布局过程的顺序可以表示为：`viewWillLayoutSubviews()` → 根视图执行子视图布局 → `viewDidLayoutSubviews()`。首次页面布局、旋转或分屏导致的尺寸变化，以及约束变化引起的根视图重新布局，都可能触发回调。布局回调可以多次执行，不等于页面已经完成显示，也不保证之后尺寸不再变化。

`setNeedsLayout()` 把视图标记为需要布局，通常在后续布局周期处理。`layoutIfNeeded()` 在指定视图范围内执行当前待处理的布局；没有待处理布局时，不保证再次调用布局回调。仅有某个后代视图发生局部布局，也不能据此断言控制器一定收到根视图的布局回调。

这个方法适合根据本次布局得到的 `bounds` 更新附加图层的 `frame`、圆角半径或阴影路径。下面假设 `gradientLayer` 已在 `viewDidLoad()` 中创建并加入 `view.layer`，且保持默认锚点、没有变换：

```swift
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    gradientLayer.frame = view.bounds
}
```

`viewDidLoad()` 表示根视图完成加载，适合一次性创建和配置对象，但不保证已有本次界面展示需要的布局尺寸。`viewDidLayoutSubviews()` 中的尺寸更新应能安全重复执行；不要在每次回调中重复添加图层、创建控件或发起与布局无关的网络请求。使用 Auto Layout 管理的子视图，应优先通过约束表达布局，避免在这里反复覆盖 `frame`。

该回调只说明根视图完成了这一轮对子视图的布局，不保证所有后代视图都完成了各自的内部布局。需要读取某个后代视图内部的布局结果时，应在正确的视图范围和时机确认其布局已完成。

## frame、Auto Layout 与 SnapKit

在视图没有旋转、缩放等变换时，`frame` 表示视图在父视图坐标系中的位置和大小。Auto Layout 根据约束计算视图的位置和大小，UIKit 在布局过程中更新对应的 `frame`。SnapKit 封装的是 Auto Layout 的约束 API，不是一次 `frame` 赋值。

约束是视图几何属性之间的等式或不等式，例如“子视图左边缘距离父视图左边缘 20 点”。一组足以确定布局且没有冲突的约束，可以在给定父视图尺寸等条件后确定子视图的位置和大小。

调用 SnapKit 后，执行关系如下：

1. 代码调用 `snp.makeConstraints`，由 SnapKit 创建并激活 Auto Layout 约束。
2. UIKit 在后续布局过程中，根据约束和当前布局条件求解位置与大小。
3. UIKit 更新视图的几何信息，代码此时可以读取与当前布局一致的 `frame`。

`makeConstraints` 返回只表示约束设置操作完成，不保证新布局已经执行。立即读取 `frame` 可能得到旧值或初始值。如果当前代码需要使用布局结果，应在覆盖相关约束的父视图或祖先视图上调用 `layoutIfNeeded()`，让 UIKit 完成该范围内待处理的布局。父视图本身的尺寸也必须已经确定。

下面的示例需要引入 UIKit 和 SnapKit。容器与子视图均未设置变换，容器使用手动指定的尺寸。

```swift
import UIKit
import SnapKit

let container = UIView(
    frame: CGRect(x: 0, y: 0, width: 300, height: 100)
)
let child = UIView()
container.addSubview(child)

child.snp.makeConstraints { make in
    make.edges.equalToSuperview().inset(20)
}

container.layoutIfNeeded()
// child.frame：x = 20，y = 20，width = 260，height = 60

container.frame.size.width = 400
container.layoutIfNeeded()
// child.frame：x = 20，y = 20，width = 360，height = 60
```

两次布局使用同一组约束。父视图变宽后，Auto Layout 重新计算子视图宽度，使四边间距继续保持 20 点。直接设置一次 `frame` 只保存当时的位置和大小，不会单独建立这种持续的间距关系；后续适配需要代码重新计算，或由其他布局机制负责。

这些计算结果依赖当前约束和尺寸条件。约束不足时，布局可能不唯一；约束冲突时，系统可能打破部分约束。存在非单位 `transform` 时，也不能直接用上述规则解释或设置 `frame`。

### 固有内容尺寸、抗压缩与约束优先级

固有内容尺寸（`intrinsicContentSize`）是视图根据自身内容向 Auto Layout 报告的期望尺寸，不等于最终的 `frame.size`。对于多行 `UILabel`，所需高度还与可用文字宽度、字体和行数等排版条件有关。下面讨论的是当前布局条件下已经正确确定的固有内容高度。

当视图在某个方向上提供有效的固有内容尺寸时，Auto Layout 可以用两条不等式表达视图保持该尺寸的要求。以垂直方向为例：

```text
抗压缩（Compression Resistance）：视图高度 ≥ 固有内容高度
抗拉伸（Content Hugging）：视图高度 ≤ 固有内容高度
```

`setContentCompressionResistancePriority(_:for:)` 设置第一条不等式的优先级。`setContentHuggingPriority(_:for:)` 设置第二条不等式的优先级。`for: .vertical` 对应高度，`for: .horizontal` 对应宽度。普通 `UIView` 默认不提供固有内容尺寸；某个方向返回 `UIView.noIntrinsicMetric` 时，不能仅靠这些优先级得到该方向的内容尺寸约束。

约束优先级表示约束在布局求解中的重要程度，不表示代码执行顺序。`UILayoutPriority.required` 的值为 1000，表示必须满足；低于 1000 的约束属于可选约束。必须满足的约束彼此兼容时，Auto Layout 在满足这些约束的前提下，按优先级尽量满足可选约束。

下面用标题和上方标识之间的间距举例。假设 `titleLabel` 与 `badgeView` 已加入 `container`，标识的位置、标题的水平位置和宽度已由其他约束确定：

```swift
titleLabel.setContentCompressionResistancePriority(
    .required,
    for: .vertical
)

titleLabel.snp.makeConstraints { make in
    make.top.equalTo(badgeView.snp.bottom)
        .offset(8)
        .priority(999)

    make.bottom.equalTo(container.snp.bottom)
        .priority(.required)
}
```

这段代码中的关键关系可以表示为：

```text
标题高度 ≥ titleLabel.intrinsicContentSize.height    优先级 1000
标题顶部 = 标识底部 + 8 pt                           优先级 999
标题底部 = 容器底部                                  优先级 1000
```

`.priority(999)` 只设置标题顶部间距约束的优先级，没有修改标题的抗压缩优先级。两类约束会参与同一次布局求解。标题底部固定后，标题顶部的位置会影响标题高度，所以间距约束与抗压缩约束可能产生冲突。仅有顶部间距约束时，不能据此断言标题高度会被压缩。

例如，假设标识底部到容器底部的距离由优先级 1000 的约束固定为 47 pt，标题报告的固有内容高度为 39.333 pt，且没有其他约束改变下述结果：

```text
保持 8 pt 间距时：标题高度 = 47 − 8 = 39 pt
抗压缩要求：标题高度 ≥ 39.333 pt
保留内容高度时：实际间距 = 47 − 39.333 = 7.667 pt
```

在这个条件下，Auto Layout 让优先级 999 的间距等式不再严格成立，保留标题的内容高度。间距约束的 `constant` 仍然是 8，代码中的预设值没有改变；最终布局得到的实际间距是 7.667 pt。优先级相差 1 不表示尺寸相差 1 pt，尺寸偏差由整组约束和可用空间决定。

如果空间足够，间距约束和抗压缩约束能够同时成立，Auto Layout 会同时满足两者。例如上述距离改为 47.333 pt，就可以同时保留 8 pt 间距和 39.333 pt 标题高度。若其他视图的位置、容器高度或底部间距也能变化，最终结果还取决于这些约束的优先级，不能直接套用“顶部间距一定缩小”的结论。

抗压缩约束只要求高度不低于固有内容高度，不要求两者相等。高度是否继续增大，还受抗拉伸和其他约束影响。调整这些优先级不会直接改变字体，也不会修正错误的内容测量结果。

同样的机制也适用于水平方向。例如标题与价格并排，容器宽度、边距和间距固定，且价格本身能够放入容器时，可以把价格的水平抗压缩优先级设为 750，把标题设为 749。在没有其他更高优先级约束改变结果的条件下，空间不足时会优先缩小标题宽度。标题是否显示省略号，还需要通过 `numberOfLines` 和 `lineBreakMode` 等属性配置。

如果另有优先级 1000 的约束强制标题高度等于 39 pt，而抗压缩又要求高度至少为 39.333 pt，两条必须满足的约束就无法同时成立。系统会报告约束冲突，并打破其中某条约束来恢复布局。因此，1000 不是“无论其他条件如何都能保住内容”的保证。只有明确需要该强度并确认其他必须满足的约束兼容时，才应使用 `.required`。

### 约束对象的创建、激活与复用

NSLayoutConstraint 表示一条布局关系。变量 widthConstraint 只是保存约束对象的引用；约束针对哪个视图，由创建约束时使用的锚点决定。锚点表示视图的某个几何属性，例如宽度或左边缘。

```swift
let badge = UIView()
container.addSubview(badge)
badge.translatesAutoresizingMaskIntoConstraints = false

let widthConstraint = badge.widthAnchor.constraint(equalToConstant: 80)
widthConstraint.isActive = true

// 内容变化后修改同一个约束
widthConstraint.constant = 120
```

本例假设 container 已存在。badge.widthAnchor 创建的约束表达 badge 的宽度等于 80 点。约束创建后默认未激活；isActive = true 让系统把约束加入布局计算。这个宽度约束不会独立决定 badge 的高度和位置，完整布局仍需要相应约束。

把 constant 改成 120 是更新既有关系，不是创建第二条宽度约束。后续布局会使用新值；只有需要立即读取结果或组织布局动画时，才在覆盖相关约束的视图范围内调用 layoutIfNeeded()。需要跨方法更新时，可以把 widthConstraint 保存为拥有该视图的对象的属性。

### 布局计算为什么可能耗时

Auto Layout 根据当前约束和内容尺寸确定视图的位置与大小。文字换行、视图显隐和宽度变化可能改变布局输入，系统需要据此更新相关布局。约束之间依赖较多、文本测量较重，或者程序反复使布局失效时，相关工作可能占用较多主线程时间；不能把使用 Auto Layout 本身当作卡顿结论。

例如，Cell 内容宽度为 343 点，正文左右各留 16 点，可用文字宽度就是 311 点。系统根据正文、字体和排版设置得到正文高度，再结合其他控件计算 Cell 高度：上边距 16 + 标题高 24 + 间距 8 + 正文高 96 + 间距 12 + 按钮高 40 + 下边距 16 = 212 点。正文变长或可用宽度变窄后，正文可能增加行数，Cell 高度也要重新确定。

如果每次配置 Cell 都删除并重建全部约束，或者每修改一个约束就调用一次 layoutIfNeeded()，程序会增加约束维护或立即布局的次数。通常应一次性建立固定约束，内容变化时只更新必要的值，让 UIKit 合并可以延后执行的布局请求。

不必要的容器也会增加视图和约束维护成本。例如 contentView → A → B → C → titleLabel 中，A、B、C 都与父视图四边重合，没有背景、裁剪、交互或独立布局职责，只有 titleLabel 距 C 左边 16 点。若这些容器确实没有其他用途，可以直接把 titleLabel 放入 contentView，并保留相同的 16 点边距。这里不是让三个容器各加 16 点；后者会累计成 48 点，不能直接改成 16 点。

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

### 视图树与子视图数组

一个根视图及其后代视图构成有序多叉树：每个视图节点可以有多个子节点，同一父视图下的子节点有确定的顺序。一个视图同一时刻最多有一个直接父视图；没有父视图的节点可以作为一棵树的根。

`superview` 指向当前视图的直接父视图，没有父视图时为 `nil`。`subviews` 是当前视图的直接子视图数组，不包含孙视图等更深层的后代。递归访问每个子视图的 `subviews`，才能遍历整个视图子树。

同级视图使用默认的图层深度设置时，`subviews` 中索引较小的视图位于后方，索引较大的视图位于前方。`addSubview(_:)` 通常把视图加入数组末尾；`insertSubview`、`bringSubviewToFront(_:)` 和 `sendSubviewToBack(_:)` 可以调整顺序。这里描述的是公开接口中的组织关系，不推断 UIKit 内部使用了哪一种存储实现。

### UIStackView 与 UIScrollView 的继承和职责

`UIStackView` 不是 `UIScrollView` 的子类。`UIStackView` 和 `UIScrollView` 都直接继承自 `UIView`。类继承关系表示一个类基于另一个类定义，并继承相应的属性和方法。

`UIStackView` 负责排列视图。`arrangedSubviews` 是由 `UIStackView` 自动安排布局的子视图数组。`UIStackView` 根据 `axis` 指定的水平或垂直方向，以及间距、对齐等设置排列这些子视图。`UIStackView` 本身不提供滚动能力。

`UIScrollView` 负责内容滚动。`UIScrollView` 改变内容相对可见区域的位置，使用户能够查看超出可见区域的内容。`UIScrollView` 不会自动把普通子视图排成一行或一列。

父子视图关系表示具体视图对象之间的包含关系，与类继承关系不同。例如，`scrollView` 是一个 `UIScrollView` 对象，`stackView` 是一个 `UIStackView` 对象。调用 `scrollView.addSubview(stackView)` 后，`scrollView` 成为 `stackView` 的父视图，但 `UIStackView` 的父类仍然是 `UIView`。

需要让排列好的内容可以滚动时，可以把 `UIStackView` 放入 `UIScrollView`：由 `UIStackView` 排列内容，由 `UIScrollView` 提供滚动。还需要正确设置内容尺寸或用于确定内容尺寸的布局约束；仅调用 `addSubview` 不会自动完成滚动布局。

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

## 分栏界面与详情控制器生命周期

### 界面外观与各栏分工

典型的分栏界面会同时显示“左侧列表”和“右侧详情”。下图是苹果官方的 iPad 邮件截图，用于展示这种界面外观；下面的控制器分工是使用 UISplitViewController 实现同类界面的通用示例。

![图片展示了iPad邮件界面，左侧为邮件列表，显示“所有收件箱”，有Jenny Court、Xiaomeng Zhong等人的邮件，部分邮件被蓝色框选。右侧是选中邮件“Neighborhood garden”的正文，内容为关于邻里花园规划的讨论。图片上方标注了三个功能区域：左上角“Change mailboxes or accounts.”，右上角“Delete, move, or mark multiple messages.”，右下角“Move message to another mailbox.”，与上下文介绍的界面外观及控制器分工相呼应。](../assets/ipad-mail-split-view.png)

图示：左侧选择邮件，右侧显示选中邮件的正文。图片来源：Apple《在 iPad 上的“邮件”中查看电子邮件》。

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

## UITableView 的重用机制

UITableView 保留已经创建的 UITableViewCell，在需要显示另一行时重新配置该对象，减少 Cell 和子视图的重复创建。Cell 是展示一行内容的视图对象，不固定对应某条数据。数据模型保存内容和状态；稳定的数据 ID 标识数据项；IndexPath 由 section 和 row 表示当前位置，插入、删除或重排后可能改变。

### 注册、获取与重新配置

reuseIdentifier 是区分可重用 Cell 类别的字符串。注册 Cell 类、注册 nib 界面文件，或配置 Storyboard 原型，都是告诉表格如何在没有可重用对象时创建 Cell。注册建立创建规则，不会立即为全部数据创建 Cell。

表格按需调用数据源的 tableView(_:cellForRowAt:)，传入表格和目标 IndexPath。数据源是负责提供行数及配置后 Cell 的对象。数据源根据位置取出模型，再调用 dequeueReusableCell(withIdentifier:for:)。表格找到对应标识的可重用对象时，会调用该对象的 prepareForReuse() 后返回；没有可用对象时，根据注册信息或原型创建新对象。数据源完整配置内容，再把 Cell 返回给表格。

带 for: IndexPath 的方法要求事先提供匹配的注册信息或 Storyboard 原型，否则会发生运行时错误。调用这个方法不等于每次创建新对象。重用池是表格管理的可再次使用的 Cell 集合，应用不需要手动把 Cell 放回。不要依赖具体内部容器或取出顺序。

以下是独立单分区示例。页面需在首次获取 Cell 前调用 registerCell(in:)，并把持有此方法的对象设为数据源。contentConfiguration 适用于 iOS 14 及以上。

```swift
import UIKit

final class ItemDataSource: NSObject, UITableViewDataSource {
    struct Item {
        let title: String
        let subtitle: String?
        let isFavorite: Bool
    }

    var items: [Item] = []

    func registerCell(in tableView: UITableView) {
        tableView.register(UITableViewCell.self,
                           forCellReuseIdentifier: "ItemCell")
    }

    func tableView(_ tableView: UITableView,
                   numberOfRowsInSection section: Int) -> Int {
        items.count
    }

    func tableView(_ tableView: UITableView,
                   cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let item = items[indexPath.row]
        let cell = tableView.dequeueReusableCell(
            withIdentifier: "ItemCell", for: indexPath
        )
        var content = cell.defaultContentConfiguration()
        content.text = item.title
        content.secondaryText = item.subtitle
        cell.contentConfiguration = content
        cell.accessoryType = item.isFavorite ? .checkmark : .none
        return cell
    }
}
```

页面需要持续持有 ItemDataSource 实例，因为 UITableView 不会通过 dataSource 属性强持有数据源。模型内容改变后，还需通过合适的表格更新接口刷新受影响的行。

每次配置必须覆盖全部与数据相关的状态。若只在 isFavorite 为 true 时设置 checkmark，而 false 时不设置 none，上一行的勾选就可能残留。副标题、图片、颜色、显隐和随数据变化的约束也遵循相同规则。

### prepareForReuse 与显示回调

| 位置或方法 | 职责 |
| --- | --- |
| 初始化方法或 awakeFromNib() | 建立固定子视图、固定约束和不随数据变化的外观 |
| cellForRowAt 中调用的配置方法 | 根据当前模型完整设置内容，包括没有图片或副标题的情况 |
| prepareForReuse() | 重置与内容无关的临时状态，清理 Cell 自己持有的旧任务或动画 |
| willDisplay | 响应 Cell 即将显示，可启动只在可见期间运行的动画 |
| didEndDisplaying | 响应 Cell 已停止显示，可暂停只在可见期间运行的动画 |

prepareForReuse() 在已有 Cell 被取出准备重用时调用，不是在 Cell 刚离屏时立即调用。重写该方法必须调用 super.prepareForReuse()。新建 Cell 不通过这个方法完成初始化。内容仍应由配置方法完整赋值，不能依赖 prepareForReuse() 先清空全部内容；reconfigureRows(at:) 更新已有 Cell 时不会调用 prepareForReuse()。

收藏、展开或输入内容等需要长期保留的状态，应存到数据模型或按数据 ID 管理的状态中，不应只保存在可重用的 Cell 上。固定视图与约束的创建时机见[多元素 Cell 的配置成本](#多元素-cell-的配置成本)。

### 预取与重用数量

预取是提前准备可能即将需要的内容。UIKit 的 Cell 预取机制可能在 Cell 进入屏幕前调用数据源并配置 Cell。已配置的 Cell 可能最终没有显示；已停止显示的 Cell 也可能继续保留当前配置，随后再次显示同一行。因此，cellForRowAt、willDisplay、didEndDisplaying 和 prepareForReuse 不一定严格一一对应。

1,000 行数据不要求同时保留 1,000 个 Cell，但创建数量也不固定等于可见行数或“可见行数加一”。数量受布局、预取和系统管理影响。兼容的视图结构应共享重用标识；不同结构可使用不同标识。为每行设置唯一标识会妨碍不同数据行共享已有 Cell。

异步结果与 Cell 当前数据的匹配规则见[长列表性能与异步图片复用](#长列表性能与异步图片复用)。图片请求的共享、取消与结果校验见[长列表图片任务调度](image-loading-decoding-and-memory.md#长列表图片任务调度)。

## UICollectionView 的职责与复用

UICollectionView 展示一组可滚动内容。Cell 展示单项；DataSource 提供数量与配置好的 Cell；Delegate 处理点击等交互；Layout 提供位置、尺寸等布局信息。数据按 section／item 组织，IndexPath 表示当前位置，不是永久业务身份。

基本流程：创建并设置布局 → 注册 Cell → 设置数据源和代理 → 按需获取、配置 Cell → 根据布局信息展示。随着滚动复用 Cell，不为所有数据一次性创建视图；每次配置应覆盖完整状态，避免残留旧内容。

SnapKit 可负责 UICollectionView 在页面中的位置，以及 Cell 内控件约束。Cell 在列表中的排列由 UICollectionViewLayout 负责，例如 FlowLayout、CompositionalLayout 或自定义 Layout。

### 分区、项目与分区边距

分区（section）是 UICollectionView 对数据项的一层逻辑分组。项目（item）是分区内的一项数据，Cell 是展示该项目的可复用视图。一个分区可以显示多行 Cell，也可以不显示分区头部或尾部。

例如，同一个图片列表可以把“今天”和“昨天”各作为一个分区：

```text
同一个 UICollectionView
├─ section 0：今天
│  ├─ Header：今天（可选）
│  ├─ 第一行：[item 0] [item 1] [item 2]
│  ├─ 第二行：[item 3] [item 4]
│  └─ Footer（可选）
└─ section 1：昨天
   ├─ Header：昨天（可选）
   └─ 第一行：[item 0] [item 1]
```

每个分区的 item 索引独立从 0 开始。IndexPath(item: 1, section: 0) 表示第一个分区中的第二项，不是整个列表的第二行。

使用垂直滚动的 UICollectionViewFlowLayout 时，Cell 通常沿水平方向排列，当前行放不下时换到下一行。分区与行不是同一个概念。

sectionInset 描述分区内容的四周边距。在常见的垂直布局中，top 对应头部之后与第一行 Cell 之前的留白，bottom 对应最后一行 Cell 之后与尾部之前的留白，left 和 right 对应内容两侧的留白。未配置头部或尾部时，相应间距位于分区内容边缘。

```swift
layout.sectionInset = UIEdgeInsets(top: 10, left: 16, bottom: 10, right: 16)
```

此处 layout 是 UICollectionViewFlowLayout。具体参考边界还受 sectionInsetReference 等布局设置影响。行间距由 minimumLineSpacing 控制；同一行项目之间的最小间距由 minimumInteritemSpacing 控制。这些间距与 sectionInset 不同。

### 数据源与代理的五个重点回调

数据源负责向 UICollectionView 提供数据数量和配置好的视图；代理接收交互、显示变化等通知，也可以在约定方法中提供决策。下表针对直接实现 UICollectionViewDataSource 的常见用法。

| 方法 | 所属协议 | 使用场景 |
| --- | --- | --- |
| collectionView(_:numberOfItemsInSection:) | UICollectionViewDataSource | 返回指定分区的项目数，例如当前分区有 20 张图片就返回 20 |
| collectionView(_:cellForItemAt:) | UICollectionViewDataSource | 获取复用 Cell，根据 indexPath 对应的数据设置图片、标题及其他展示状态，再返回 Cell |
| collectionView(_:didSelectItemAt:) | UICollectionViewDelegate | 用户选中项目后，读取对应数据并执行选择动作，例如打开详情 |
| collectionView(_:didEndDisplaying:forItemAt:) | UICollectionViewDelegate | Cell 结束显示时，停止该 Cell 的动画等显示相关工作 |
| scrollViewDidScroll(_:) | UIScrollViewDelegate | 滚动位置变化时读取 contentOffset，例如按位置调整导航栏透明度 |

前两个方法是 UICollectionViewDataSource 的必需方法。多个分区还需要通过 numberOfSections(in:) 提供分区数量；未实现该可选方法时，默认按一个分区处理。

cellForItemAt 表示准备 Cell，不表示用户已经看到 Cell。UIKit 可能提前调用该方法，因此不能直接把该回调当成内容曝光。willDisplay 表示即将显示，也不能独自证明内容已满足实际可见区域或持续显示时间等曝光条件。

didEndDisplaying 回调中的 Cell 是结束显示的视图。数据可能已经发生插入、删除或重排，因此需要关联数据时，应使用稳定的数据标识，不能盲目按旧 indexPath 读取当前数组。

scrollViewDidScroll 既可能来自用户拖动，也可能来自减速滚动或程序改变滚动位置。contentOffset 表示当前内容滚动偏移。该回调可能频繁发生，适合执行轻量的界面更新。

### 代理协议的继承与布局回调

UICollectionViewDelegate 继承 UIScrollViewDelegate。因此，设置为 collectionView.delegate 的同一个对象，可以同时实现列表选择回调和滚动回调，不需要另设一个滚动代理属性。

UICollectionViewDelegateFlowLayout 又继承 UICollectionViewDelegate。使用 UICollectionViewFlowLayout 时，可以按需实现 sizeForItemAt 来提供项目尺寸，或实现 insetForSectionAt 来提供不同分区的边距。尺寸和边距统一时，直接设置布局对象的 itemSize、sectionInset 即可。

这些 FlowLayout 回调针对 UICollectionViewFlowLayout。其他布局应通过各自的布局接口配置。上述 UIKit 代理协议的方法均为可选方法，不要求把全部回调都写出来。

协议继承的通用规则见[Swift 语言基础中的协议继承](../language/swift-basics.md#协议继承与类继承的区别)。

## 页面、列表与 Cell ViewModel

MVVM 没有规定一个页面只能有一个 VM，也没有规定每个 View 都必须有 VM。划分依据是是否存在需要独立管理的展示数据、状态与交互职责。

| 范围 | 常见职责 |
|-|-|
| 页面／容器 VM | 整体展示状态、子区域组织与协调 |
| 列表 VM | 列表项数组、加载、刷新、分页、插入和删除 |
| Cell VM | 单项展示数据、样式、选中／高亮等状态及交互处理 |

列表级 VM 管理列表有哪些项、如何排序和更新。单项 VM 管理该项的数据与展示状态。容器 VM 可以组织多个子区域的 VM；是否增加这一层，取决于是否存在独立的协调职责。

复杂评论项需要管理内容、一级／二级样式、高亮、Cell 类型与高度，以及点赞／回复／删除等处理入口，拆分 Cell VM 有助于明确职责。如果只是展示固定标题或简单静态文本、没有额外状态与处理逻辑，直接配置 View 也可以，不必为了 MVVM 名称增加 VM。VM 的层级应根据界面的职责划分。

**VM 对应数据项，Cell 是可复用的视图对象。** 100 条数据各有 VM，不意味着立即创建 100 个 Cell。同一个 Cell 滚动后可以绑定另一项的 VM，绑定时需要更新展示状态。框架布局、视图创建和动画执行仍由 View 等 UI 层承担。

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

Cell 复用主要减少视图创建／销毁、重复布局对象和内存占用，不会自动取消网络请求或清空旧状态。异步赋图前需要在主线程检查当前绑定的内容 ID／请求标识，再连续完成赋值；只保留 Cell 引用并不能保证它仍代表原来的数据。 例如 Cell A 为数据 X 发起图片请求，随后改为显示数据 Y；X 的旧回调若直接赋图，就会覆盖 Y 的图片。取消旧请求不能代替结果归属检查；[weak cell] 只避免强持有 Cell，不保证 Cell 仍显示原数据。也可以按稳定数据 ID 保存结果，再刷新当前对应的行，避免旧回调直接修改已重用的 Cell。

### 多元素 Cell 的配置成本

Cell 中固定的子视图和约束通常在初始化时建立。复用配置时更新文字、图片、显隐和必要的约束值，避免每次配置都重复创建视图或重建同一套约束。减少无效的层级与重复布局，需要以实际测量结果为依据；不能只凭 Cell 中元素数量判断瓶颈。

数据发生局部变化时，优先刷新受影响的项目，避免在滚动过程中反复刷新整个列表。图片的目标尺寸、后台解码与缓存策略见[iOS 图片加载、解码与内存](image-loading-decoding-and-memory.md)。

### 不同高度 Cell 的测量与缓存

高度缓存是保存某个数据项在特定内容和布局条件下计算出的高度数值，供后续尺寸计算直接读取。高度缓存不保存 Cell 对象，也不会自动让 Auto Layout 跳过所有内部布局。

先测量文本和布局计算是否构成明显开销。如果重复测量同一内容的成本较高，再缓存结果。首次需要尺寸时测量并保存；后续需要同一条件下的尺寸时读取缓存，并把结果实际提供给列表的尺寸回调或布局逻辑。仅把高度写入字典，而布局仍然重新测量，不会获得预期收益。

缓存需要依据稳定的数据 ID，并区分内容版本、可用宽度、字体或字号、展开状态等实际影响高度的条件。IndexPath 表示当前位置，插入、删除或重排后可能变化，不适合直接作为数据项的永久身份。

例如，数据项 A 在当前条件下测得 180 点。A 离屏后，原 Cell 可以被复用来显示 B，但 A 的模型和高度缓存可以继续存在。用户滚动回来，列表再次需要 A 的高度时，不管此时取得的是哪个 Cell，都可以按 A 的数据标识查缓存。这就是滑动过程中再次遇到同一内容和布局条件的场景；并不要求两个不同数据项的正文相同。

应用通过构造包含布局输入的缓存键判断能否复用，不依赖 Cell 自己记住上次的值。下面的示例假设列表统一使用一套排版配置，并由数据层维护内容版本：

```swift
struct HeightCacheKey: Hashable {
    let itemID: UUID
    let contentVersion: Int
    let availableWidth: CGFloat
    let typographyVersion: Int
    let isExpanded: Bool
    let layoutVersion: Int
}

// 通常作为列表控制器或布局管理对象的属性
var heightCache: [HeightCacheKey: CGFloat] = [:]
```

contentVersion 在正文、图片比例等影响高度的内容改变时更新。typographyVersion 在字体、字号、行距、换行或行数设置改变时更新。layoutVersion 在边距、控件排列等布局规则改变时更新。availableWidth 使用本次实际参与测量的宽度，isExpanded 使用当前展开状态。版本号必须由对应数据或配置的修改路径维护，不会自动变化；不能只用数据 ID 就声称所有输入未变。

例如，A 的内容版本为 3、宽度为 343、排版版本为 1、未展开、布局版本为 1。第一次测量后，把该键对应的高度保存为 180。滚动回来时构造出的键完全相同，就读取 180；正文更新后内容版本变为 4，新键查不到旧高度，因此重新测量并保存。

缓存放在列表控制器或布局管理对象中，可以独立于 Cell 生命周期保存。采用手动行高时，tableView(_:heightForRowAt:) 应按当前键查缓存，未命中时测量并写入，然后返回结果。继续使用 UITableView.automaticDimension 时，系统仍负责尺寸计算，应用自己的字典不会自动接管测量。更复杂的输入应继续纳入键，或者在相关配置变化时主动清理受影响的缓存。

列表自适应高度与手动高度缓存是两种需要明确协调的尺寸来源。接入缓存时应确认当前布局实际使用哪一个结果，避免认为保存了高度就一定减少测量次数。

### 优化效果的验证

使用同一设备、同一批内容、相同缓存条件和相近的滚动操作对比优化前后表现。先观察卡顿次数、持续时间和内存峰值，再结合 CPU 调用栈与渲染阶段判断瓶颈是否减轻。更少的网络请求或更低的平均 CPU 占用，不能单独证明每次滚动都没有卡顿。

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
