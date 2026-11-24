# UIKit 布局、列表与 MVVM

[返回笔记目录](../README.md)

## 本文目录

- [页面生命周期与展示方式](#页面生命周期与展示方式)
  - [控制器初始化、视图加载与页面展示](#控制器初始化视图加载与页面展示)
  - [push、present 与出现消失回调](#pushpresent-与出现消失回调)
  - [pageSheet、formSheet 与尺寸设置](#pagesheetformsheet-与尺寸设置)
  - [viewDidLayoutSubviews 的调用时机与用途](#viewdidlayoutsubviews-的调用时机与用途)
  - [UIButton 的 system 类型与 tintColor](#uibutton-的-system-类型与-tintcolor)
- [frame、Auto Layout 与 SnapKit](#frameauto-layout-与-snapkit)
  - [transform 与 bounds、center、frame](#transform-与-boundscenterframe)
  - [添加子视图但不设置约束](#添加子视图但不设置约束)
  - [autoresizingMask 与约束转换](#autoresizingmask-与约束转换)
  - [固有内容尺寸、抗压缩与约束优先级](#固有内容尺寸抗压缩与约束优先级)
  - [约束对象的创建、激活与复用](#约束对象的创建激活与复用)
  - [布局计算为什么可能耗时](#布局计算为什么可能耗时)
- [视图层级与父子关系](#视图层级与父子关系)
  - [视图树与子视图数组](#视图树与子视图数组)
  - [调整同级视图的前后顺序](#调整同级视图的前后顺序)
  - [更换父视图与移除子视图](#更换父视图与移除子视图)
  - [UIStackView 与 UIScrollView 的继承和职责](#uistackview-与-uiscrollview-的继承和职责)
- [UIView 子视图的回调与可见性](#uiview-子视图的回调与可见性)
- [容器控制器与子控制器](#容器控制器与子控制器)
- [分栏界面与详情控制器生命周期](#分栏界面与详情控制器生命周期)
  - [界面外观与各栏分工](#界面外观与各栏分工)
  - [点击邮件是否触发生命周期回调](#点击邮件是否触发生命周期回调)
  - [示例：用新的详情控制器替换旧对象](#示例用新的详情控制器替换旧对象)
- [UITableView 的重用机制](#uitableview-的重用机制)
  - [注册、获取与重新配置](#注册获取与重新配置)
  - [prepareForReuse 与显示回调](#prepareforreuse-与显示回调)
  - [Cell 中的倒计时与可见性](#cell-中的倒计时与可见性)
  - [预取与重用数量](#预取与重用数量)
- [UITableView 的行删除与批量更新](#uitableview-的行删除与批量更新)
- [UICollectionView 的职责与复用](#uicollectionview-的职责与复用)
  - [分区、项目与分区边距](#分区项目与分区边距)
  - [数据源与代理的五个重点回调](#数据源与代理的五个重点回调)
  - [代理协议的继承与布局回调](#代理协议的继承与布局回调)
- [页面、列表与 Cell ViewModel](#页面列表与-cell-viewmodel)
- [MVVM 的输入、状态输出与界面刷新](#mvvm-的输入状态输出与界面刷新)
- [长列表性能与异步图片复用](#长列表性能与异步图片复用)
  - [多元素 Cell 的配置成本](#多元素-cell-的配置成本)
  - [不同高度 Cell 的测量与缓存](#不同高度-cell-的测量与缓存)
  - [优化效果的验证](#优化效果的验证)
- [布局更新与列表定位时机](#布局更新与列表定位时机)

## 页面生命周期与展示方式

UIViewController 的视图加载、出现和消失构成页面生命周期，与 App、Scene 的生命周期是不同范围的概念。本节说明这些回调在什么时候调用。需要记住三点：布局回调可能执行多次；页面消失时控制器不一定被释放；交互式转场（例如侧滑返回）可能中途取消。

### 控制器初始化、视图加载与页面展示

控制器初始化、根视图加载和页面展示是三个不同的步骤。`init` 只设置控制器的初始状态，此时根视图还没有加载。初始化方法的分类与 `required init?(coder:)` 见[UIViewController 的初始化路径](../language/swift-properties-and-initialization.md#uiviewcontroller-的-initcoder-与纯代码初始化)。

`loadView()` 是控制器创建根视图的入口。根视图尚未加载时，访问 `view` 会触发加载；`loadViewIfNeeded()` 可以显式要求加载，根视图已经加载时不会再次加载。`isViewLoaded` 用于判断加载状态，`viewIfLoaded` 可以读取已加载的根视图而不触发加载。

根视图加载完成后，UIKit 调用 `viewDidLoad()`。根视图只加载一次，所以 `viewDidLoad()` 通常只调用一次，不会每次页面出现都调用。控件创建和约束配置一般放在这里。视图加载完成时，页面还没有显示给用户。最终布局尺寸的读取时机见[viewDidLayoutSubviews 的调用时机与用途](#viewdidlayoutsubviews-的调用时机与用途)。

`present(_:animated:completion:)` 以模态方式展示另一个控制器。一个视图还没加载的控制器，第一次被正常展示时，依次经过：

```text
初始化控制器
→ 展示流程需要根视图
→ loadView
→ viewDidLoad
→ viewWillAppear
→ 展示转场完成
→ viewDidAppear
```

转场是界面从一个展示状态切换到另一个状态的动画过程。UIKit 在展示过程中会执行布局，但布局回调执行几次不固定。如果其他代码已经提前访问过 `view`，视图已经加载，展示时就不会再调用 `loadView` 和 `viewDidLoad`。

`viewDidAppear(_:)` 在页面完全显示出来后调用，所以与页面显示相关的倒计时应在这里开始，而不是在 `viewDidLoad` 中。页面每次重新出现都会调用 `viewDidAppear(_:)`；只能启动一次的任务，要自己记录是否已经启动过。

`dismiss(animated:completion:)` 关闭模态展示的页面。调用返回时转场不一定完成，控制器也不一定立即释放：其他对象或异步闭包仍强引用控制器时，控制器会继续存活。

如果页面在异步任务 `await` 期间被关闭，任务恢复后要先检查任务是否已被取消、页面是否还需要这个结果，再修改界面。取消请求不会强制中断代码，具体规则见[Task 的上下文继承与取消](../language/swift-concurrency.md#上下文继承与取消)。

### push、present 与出现消失回调

`pushViewController(_:animated:)` 让 `UINavigationController` 把控制器压入导航栈。导航栈是导航控制器按先后顺序保存的一组控制器。A push B 后，A 仍在栈中；返回时调用 `popViewController(animated:)`，或者用侧滑手势返回。

`present(_:animated:completion:)` 以模态方式展示 B，通常用于一段独立的流程，B 不会进入 A 的导航栈，关闭时调用 `dismiss(animated:completion:)`。也可以 present 一个新的导航控制器，由它管理自己的导航栈。模态页面的外观由 `modalPresentationStyle`、设备尺寸和系统适配共同决定，不一定是从底部弹出的全屏页面。

下表假设 A 正常展示 B、转场顺利完成，且使用 UIKit 默认的回调管理。A 是否收到回调，取决于展示方式，而不是 B 遮住了 A 多少。

| 展示方式 | B 出现时 A 的回调 | B 关闭、A 恢复时 A 的回调 |
| --- | --- | --- |
| 导航控制器正常 push B | `viewWillDisappear` → `viewDidDisappear` | pop B 后：`viewWillAppear` → `viewDidAppear` |
| present B，实际样式为 `.fullScreen` | `viewWillDisappear` → `viewDidDisappear` | dismiss B 后：`viewWillAppear` → `viewDidAppear` |
| present B，实际样式为 `.overFullScreen` | 不触发 A 的消失回调 | 不触发 A 的重新出现回调 |
| present B，实际保持非全屏 `.pageSheet` 或 `.formSheet` | 通常不触发 A 的消失回调 | 通常不触发 A 的重新出现回调 |

非全屏样式被系统适配为 `.fullScreen` 时，按全屏的规则。`.overFullScreen` 即使完全盖住了 A，A 的视图仍保留在下面，所以 A 不会收到消失回调。自定义容器、转场被取消或手动管理回调时，要按实际流程判断。

B 的根视图还没加载时，B 出现的回调顺序是 `viewDidLoad` → `viewWillAppear` → `viewDidAppear`；已经加载过就没有 `viewDidLoad`。B 被 pop 或 dismiss 时，回调顺序是 `viewWillDisappear` → `viewDidDisappear`。这些只是 B 自己的回调顺序，A 和 B 的回调之间如何交错并不固定。

A 收到消失回调不等于 A 被释放。B 被 pop 或 dismiss 也不保证立即释放；只有没有其他强引用保持 B 存活时，B 才能销毁。

### pageSheet、formSheet 与尺寸设置

`.pageSheet` 和 `.formSheet` 都是模态展示样式，实际外观取决于设备和可用空间，样式本身不规定固定尺寸。

在 iPhone 上，`.pageSheet` 通常显示为从底部升起的卡片，默认几乎占满屏幕，不是半屏。iOS 15 及以上可以通过 `sheetPresentationController` 设置卡片可以停留的高度，`detents` 就是这些高度的集合。

```swift
let detail = UIViewController()
detail.modalPresentationStyle = .pageSheet
if let sheet = detail.sheetPresentationController {
    sheet.detents = [.medium(), .large()]
    sheet.selectedDetentIdentifier = .medium
    sheet.prefersGrabberVisible = true
}
present(detail, animated: true)
```

`.medium()` 约为半屏高度，`.large()` 是卡片的最大高度。上面的设置让卡片先停在半屏高度，并在顶部显示拖动条。卡片拖到最大高度后，`modalPresentationStyle` 仍是 `.pageSheet`，不会变成 `.fullScreen`。

在空间足够的 iPad 上，`.formSheet` 通常显示为屏幕中间的一个较小窗口，适合表单。`preferredContentSize` 是控制器希望的内容尺寸，最终尺寸由系统根据可用空间决定。

```swift
let form = UIViewController()
form.modalPresentationStyle = .formSheet
form.preferredContentSize = CGSize(width: 480, height: 360)
present(form, animated: true)
```

在 iPhone 上，或者横竖屏切换、窗口尺寸变化后，`.formSheet` 可能显示为卡片，也可能被适配为全屏。所以判断外观和 A 收到哪些回调时，要看系统实际使用的样式。

### viewDidLayoutSubviews 的调用时机与用途

`viewDidLayoutSubviews()` 是 `UIViewController` 的布局回调，根视图每完成一次子视图布局，UIKit 就调用一次。布局是根据当前尺寸、约束或手动布局代码，计算并设置子视图的位置和大小。

一次布局的顺序是：`viewWillLayoutSubviews()` → 根视图布局子视图 → `viewDidLayoutSubviews()`。首次显示、旋转或分屏引起的尺寸变化、约束变化，都会让根视图重新布局并调用这个回调。所以回调会执行多次；回调被调用时页面不一定已经显示，之后尺寸也可能再变。

`setNeedsLayout()` 只把视图标记为需要布局，真正的布局在下一个布局周期进行。`layoutIfNeeded()` 立即执行这个视图范围内待处理的布局；没有待处理的布局时，什么也不做，也不会调用布局回调。只有某个子孙视图内部重新布局时，控制器不一定会收到这个回调。

这个方法适合根据本次布局得到的 `bounds` 更新附加图层的 `frame`、圆角半径或阴影路径。下面假设 `gradientLayer` 已在 `viewDidLoad()` 中创建并加入 `view.layer`，且保持默认锚点、没有变换：

```swift
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    gradientLayer.frame = view.bounds
}
```

`viewDidLoad()` 适合一次性地创建和配置对象，但这时视图的尺寸还不是最终尺寸。`viewDidLayoutSubviews()` 中的代码会反复执行，只适合做可以重复执行的尺寸更新；不要在里面添加图层、创建控件或发起网络请求。由 Auto Layout 管理的子视图，应该用约束布局，不要在这里反复修改 `frame`。

这个回调只说明根视图完成了对直接子视图的布局，子视图内部的子视图不一定已经布局完。要读取更深层视图的布局结果，需要在那个视图自己的布局回调中读取，或先对它调用 `layoutIfNeeded()`。

### UIButton 的 system 类型与 tintColor

`UIButton(type: .system)` 创建系统样式的按钮。`.system` 是 `UIButton.ButtonType` 的一个值，决定按钮的默认外观和按下时的反馈效果，标题和点击动作仍需自己设置。

`tintColor` 是视图用来突出可交互元素的颜色，视图自己没有设置时，沿用父视图的 `tintColor`。`.system` 按钮的文字标题默认使用按钮的 `tintColor`；如果设置了标题颜色、带颜色的富文本标题或按钮配置，则以这些设置为准。

```swift
let button = UIButton(type: .system)
button.setTitle("关闭", for: .normal)
button.tintColor = .white
```

示例中的标题显示为白色。`setTitle(_:for:)` 设置某个状态下的标题，`.normal` 是按钮的正常状态。`tintColor` 只影响标题和图标，不会改变按钮背景色。

如果用 `setTitleColor(_:for:)`、`setAttributedTitle(_:for:)` 或 iOS 15 的 `UIButton.Configuration` 设置过颜色，标题颜色以这些设置为准，修改 `tintColor` 不一定改变标题颜色。

## frame、Auto Layout 与 SnapKit

视图没有旋转、缩放等变换时，`frame` 是视图在父视图坐标系中的位置和大小。Auto Layout 根据约束计算视图的位置和大小，UIKit 在布局时把结果写入 `frame`。SnapKit 是对 Auto Layout 约束 API 的封装，它创建的是约束，而不是直接设置 `frame`。

约束是视图几何属性之间的等式或不等式，例如“子视图左边缘距离父视图左边缘 20 点”。约束足够且没有冲突时，只要父视图的尺寸确定，子视图的位置和大小就能唯一确定。

调用 SnapKit 后，执行关系如下：

1. 代码调用 `snp.makeConstraints`，由 SnapKit 创建并激活 Auto Layout 约束。
2. UIKit 在后续布局过程中，根据约束和当前布局条件求解位置与大小。
3. UIKit 更新视图的几何信息，代码此时可以读取与当前布局一致的 `frame`。

`makeConstraints` 返回时，约束已经激活，但布局还没执行，这时读取 `frame` 得到的是旧值或零。如果代码马上要用布局结果，就对包含这些约束的父视图调用 `layoutIfNeeded()`，让 UIKit 立即完成布局；前提是父视图本身的尺寸已经确定。

下面的示例需要引入 UIKit 和 SnapKit。容器和子视图都没有变换，容器的尺寸用 frame 手动指定。

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

两次布局用的是同一组约束。父视图变宽后，Auto Layout 重新计算子视图的宽度，四边间距仍是 20 点。直接设置 `frame` 只是一次性地设置位置和大小，父视图尺寸变化后，需要代码重新计算。

约束不足时，布局结果不唯一；约束冲突时，系统会忽略部分约束。视图设置了 `transform`（不是 `.identity`）时，`frame` 的值也不再可靠，见下一节。

### transform 与 bounds、center、frame

bounds 是视图在自身坐标系中的矩形。center 是视图中心在父视图坐标系中的位置（以下示例都使用默认锚点）。transform 是对视图显示施加的缩放、旋转和平移。

只修改 transform 时，bounds 和 center 都不变，只是显示效果改变：缩放改变显示大小，平移改变显示位置，旋转改变显示方向。默认锚点下，缩放和旋转以视图中心为基准。

frame 与 bounds、center 不同：transform 不是 .identity 时，UIKit 规定 frame 的值“未定义”，既不应读取它做计算，也不应设置它。“未定义”是指这个值不可靠，读取本身不会报错。

```swift
let view = UIView(
    frame: CGRect(x: 100, y: 100, width: 100, height: 80)
)
view.transform = CGAffineTransform(scaleX: 2, y: 2)
// bounds 仍是 (0, 0, 100, 80)
// center 仍是 (150, 140)
```

在默认锚点、父视图没有其他变换的条件下，视图显示为 200 × 160，显示区域左上角在 (50, 60)。这是按变换算出来的显示范围，与此时 frame 返回什么值无关。

如果把 transform 换成 CGAffineTransform(translationX: 40, y: 0)，视图显示在原位置右侧 40 点处，center 仍是 (150, 140)。把 transform 设回 .identity，缩放、旋转和平移都被去掉，视图恢复原来的显示。

要改变布局中的位置和尺寸，修改 bounds、center 或约束；只想改变显示效果，修改 transform。修改 transform 不会影响布局，周围的视图不会因为这个视图放大而重新排列。由 Auto Layout 管理的视图，应该用约束表达布局。

以上说的是代码设置的属性值。动画期间，属性可能已经是终点值，而屏幕上显示的是动画中间的状态；两者的区别见 [图层动画与渲染管线](rendering-and-performance.md#图层动画与渲染管线)。图层锚点的细节见 [anchorPoint、position 与图层几何](rendering-and-performance.md#anchorpointposition-与图层几何)。

### 添加子视图但不设置约束

`addSubview(_:)` 只建立父子视图关系，不会改变子视图的大小，更不会让子视图铺满父视图。没有约束时，子视图的大小和位置就是它当前的 `frame`。

下面假设没有其他代码、约束或容器布局修改子视图尺寸：

```swift
let container = UIView(
    frame: CGRect(x: 0, y: 0, width: 300, height: 200)
)
let child = UIView()
child.backgroundColor = .red
container.addSubview(child)
// child.frame 为 .zero，红色背景没有可见面积。

child.frame = CGRect(x: 20, y: 30, width: 100, height: 50)
// child 有了 100 × 50 点的区域，不需要显式约束也能采用手动布局。
```

第二种情况下，红色区域能否显示出来，还要看容器是否已经显示、视图是否隐藏、透明度和是否被裁剪。另外，普通 `UIView()` 的初始尺寸为零，但有些情况下尺寸会由其他机制决定，例如 `UIStackView` 会布局它的 `arrangedSubviews`，`UILabel` 等控件会根据内容报告固有尺寸。

### autoresizingMask 与约束转换

`autoresizingMask` 规定父视图尺寸变化时，子视图的宽高和四周边距中哪些部分跟着变化。它在子视图现有 frame 的基础上调整，不会自己算出一个初始尺寸。

用代码创建的 `UIView`，`translatesAutoresizingMaskIntoConstraints` 默认为 `true`：Auto Layout 布局时，UIKit 会把视图当前的 `frame` 和 `autoresizingMask` 转换成约束。所以即使没有手写约束，视图也有系统自动生成的约束。

`child.frame` 是 `.zero` 且宽高不随父视图变化时，自动生成的约束就是“宽度 = 0、高度 = 0”。0 是一个确定的尺寸，约束完全可以确定一个面积为零的区域，所以“有约束”不代表“能看见”。

```text
已有尺寸：宽度 0，高度 0
转换所表达的尺寸要求：宽度 = 0，高度 = 0
布局结果：尺寸仍然可以是 0 × 0
```

上面只表示约束的效果，不代表系统实际生成的约束数量和日志格式。

设置 `translatesAutoresizingMaskIntoConstraints = false` 只是关闭这项自动转换，不会创建任何新约束。之后用 Auto Layout 时，位置和大小要由手写约束、固有内容尺寸以及容器（如 UIStackView）共同确定；也可以继续用代码设置 `frame`。

例如，下面先把子视图加入父视图，再关闭自动转换，激活四边约束。父视图尺寸确定、且没有冲突约束时，子视图在之后的布局中四边各保持 20 点间距：

```swift
let child = UIView()
container.addSubview(child)
child.translatesAutoresizingMaskIntoConstraints = false

NSLayoutConstraint.activate([
    child.leadingAnchor.constraint(equalTo: container.leadingAnchor, constant: 20),
    child.trailingAnchor.constraint(equalTo: container.trailingAnchor, constant: -20),
    child.topAnchor.constraint(equalTo: container.topAnchor, constant: 20),
    child.bottomAnchor.constraint(equalTo: container.bottomAnchor, constant: -20)
])
```

### 固有内容尺寸、抗压缩与约束优先级

固有内容尺寸（`intrinsicContentSize`）是视图根据自身内容告诉 Auto Layout 的期望尺寸，最终的 `frame.size` 不一定等于它。多行 `UILabel` 的固有高度还取决于可用宽度、字体和行数。下面假设固有高度已经按当前宽度确定。

视图在某个方向上有固有内容尺寸时，Auto Layout 会生成两条不等式，表示视图希望保持这个尺寸。以垂直方向为例：

```text
抗压缩（Compression Resistance）：视图高度 ≥ 固有内容高度
抗拉伸（Content Hugging）：视图高度 ≤ 固有内容高度
```

`setContentCompressionResistancePriority(_:for:)` 设置第一条不等式的优先级，`setContentHuggingPriority(_:for:)` 设置第二条不等式的优先级；`for: .vertical` 对应高度，`for: .horizontal` 对应宽度。普通 `UIView` 没有固有内容尺寸（返回 `UIView.noIntrinsicMetric`），也就没有这两条不等式，设置这两个优先级不起作用。

约束优先级表示布局计算时约束的重要程度，与代码执行顺序无关。`UILayoutPriority.required` 的值是 1000，表示必须满足；低于 1000 的是可选约束。Auto Layout 先满足所有必需约束，再按优先级从高到低尽量满足可选约束。

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

`.priority(999)` 只设置顶部间距约束的优先级，标题的抗压缩优先级仍是 1000。标题底部固定后，标题顶部的位置决定了标题高度，所以间距约束与抗压缩约束可能冲突；是否真的冲突，要看可用空间。

例如，假设标识底部到容器底部的距离由优先级 1000 的约束固定为 47 pt，标题报告的固有内容高度为 39.333 pt，且没有其他约束改变下述结果：

```text
保持 8 pt 间距时：标题高度 = 47 − 8 = 39 pt
抗压缩要求：标题高度 ≥ 39.333 pt
保留内容高度时：实际间距 = 47 − 39.333 = 7.667 pt
```

在这个条件下，Auto Layout 让优先级 999 的间距等式不再严格成立，保留标题的内容高度。间距约束的 `constant` 仍然是 8，代码中的预设值没有改变；最终布局得到的实际间距是 7.667 pt。优先级相差 1 不表示尺寸相差 1 pt，尺寸偏差由整组约束和可用空间决定。

空间足够时，两条约束可以同时满足。例如上述距离改为 47.333 pt，就能同时保留 8 pt 间距和 39.333 pt 的标题高度。如果容器高度、其他视图的位置也可以变化，结果还取决于那些约束的优先级，顶部间距不一定被压缩。

抗压缩只要求高度不小于固有内容高度，高度可以更大；会不会更大，取决于抗拉伸和其他约束。调整这些优先级不会改变字体，也修正不了错误的内容测量。

水平方向也一样。例如标题和价格并排，容器宽度、边距和间距固定，可以把价格的水平抗压缩优先级设为 750，标题设为 749。空间不够时，优先级低的标题先被压窄，价格保持完整宽度。标题被压窄后是否显示省略号，由 `numberOfLines` 和 `lineBreakMode` 决定。

如果另有一条优先级 1000 的约束要求标题高度等于 39 pt，而抗压缩要求高度至少 39.333 pt，两条必需约束不能同时满足，系统会在控制台报告约束冲突，并忽略其中一条。所以优先级 1000 也不保证内容不被压缩。只有确认与其他必需约束不冲突时，才使用 `.required`。

### 约束对象的创建、激活与复用

NSLayoutConstraint 对象表示一条布局关系。锚点表示视图的某个几何属性，例如宽度或左边缘；约束作用于哪个视图，由创建约束时使用的锚点决定。变量 widthConstraint 只保存约束对象的引用。

```swift
let badge = UIView()
container.addSubview(badge)
badge.translatesAutoresizingMaskIntoConstraints = false

let widthConstraint = badge.widthAnchor.constraint(equalToConstant: 80)
widthConstraint.isActive = true

// 内容变化后修改同一个约束
widthConstraint.constant = 120
```

本例假设 container 已经存在。用 badge.widthAnchor 创建的约束表示 badge 的宽度等于 80 点。约束创建后默认未激活，设置 isActive = true 后才参与布局计算。这条约束只决定宽度，badge 的高度和位置还需要其他约束。

把 constant 改成 120 是修改同一条约束，不会多出一条宽度约束。下一次布局时使用新值；只有需要马上读取结果或做布局动画时，才调用 layoutIfNeeded()。如果其他方法也要修改这个宽度，就把 widthConstraint 保存为属性。

### 布局计算为什么可能耗时

Auto Layout 根据约束和内容尺寸计算视图的位置与大小。文字换行、视图显隐、宽度变化都会改变计算的输入，系统要重新计算。约束之间依赖很多、文字测量量大，或者代码反复触发重新布局时，布局会占用较多主线程时间。卡顿要找到具体原因，使用 Auto Layout 本身不是原因。

例如，Cell 内容宽度为 343 点，正文左右各留 16 点，可用文字宽度就是 311 点。系统根据正文、字体和排版设置得到正文高度，再结合其他控件计算 Cell 高度：上边距 16 + 标题高 24 + 间距 8 + 正文高 96 + 间距 12 + 按钮高 40 + 下边距 16 = 212 点。正文变长或可用宽度变窄后，正文可能增加行数，Cell 高度也要重新确定。

如果每次配置 Cell 都删除并重建全部约束，或者每改一条约束就调用一次 layoutIfNeeded()，会多出很多约束维护和布局计算。应该在创建 Cell 时一次性建好固定的约束，内容变化时只修改需要变的值，让 UIKit 把多次布局请求合并成一次。

多余的容器视图也会增加视图和约束。例如 contentView → A → B → C → titleLabel，A、B、C 都与各自父视图四边重合，没有背景、裁剪、交互等作用，只有 titleLabel 距 C 的左边 16 点。这时可以去掉 A、B、C，把 titleLabel 直接放进 contentView，左边距仍是 16 点。（如果原来是 A、B、C 各有 16 点内边距，累计就是 48 点，去掉容器后边距应改为 48 点。）

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

没有修改图层 zPosition 时，`subviews` 中索引小的视图在下面，索引大的视图在上面。上面的视图能否完全遮住下面的视图，还要看两者是否重叠、是否透明。

### 调整同级视图的前后顺序

以下操作都由父视图调用。`bringSubviewToFront(_:)` 和 `sendSubviewToBack(_:)` 的参数应是该父视图的直接子视图；相对插入方法中的参照视图也必须是该父视图的直接子视图。

| 操作 | 结果 |
| --- | --- |
| `parent.addSubview(child)` | 加入子视图，位于同级视图的最前方；已是该父视图的子视图时会移到前方 |
| `parent.bringSubviewToFront(child)` | 把现有子视图移到最前方 |
| `parent.sendSubviewToBack(child)` | 把现有子视图移到最后方 |
| `parent.insertSubview(child, aboveSubview: sibling)` | 把子视图放在参照视图前方 |
| `parent.insertSubview(child, belowSubview: sibling)` | 把子视图放在参照视图后方 |
| `parent.insertSubview(child, at: index)` | 把子视图放到指定的合法索引处 |
| `parent.exchangeSubview(at: first, withSubviewAt: second)` | 交换两个合法索引处的子视图 |

例如下面两个普通视图相互重叠。使用默认图层深度时，后加入的蓝色视图先位于前方；调整顺序后，红色视图位于前方：

```swift
let red = UIView(frame: CGRect(x: 0, y: 0, width: 100, height: 100))
let blue = UIView(frame: CGRect(x: 20, y: 20, width: 100, height: 100))
red.backgroundColor = .red
blue.backgroundColor = .blue
parent.addSubview(red)
parent.addSubview(blue)
parent.bringSubviewToFront(red)
```

调整前后顺序不会改变 `frame` 和约束，而且只在同一个父视图内部有效。例如 A 和 B 都是根视图的子视图，B 在 A 上面；把 A 的某个子视图移到 A 内部的最上面，它仍然在 B 下面。要让它显示在 B 上面，只能调整 A 和 B 在根视图中的顺序，或者改变父子关系。

### 更换父视图与移除子视图

调用 `newParent.addSubview(child)` 时，如果 `child` 已有其他父视图，UIKit 会先解除旧的父子关系，再把 `child` 加入 `newParent`。不需要先手动调用一次 `removeFromSuperview()`。只想移除时，则调用 `child.removeFromSuperview()`。

更换父视图后，`frame` 使用新父视图的坐标系。若希望保持屏幕上的位置，应在移动前转换坐标。下面假设两个父视图处于同一窗口中、相关视图没有旋转或缩放，并且 `child` 使用手动布局：

```swift
let frameInNewParent = child.convert(child.bounds, to: newParent)
newParent.addSubview(child)
child.frame = frameInNewParent
```

使用 Auto Layout 时，子视图与旧父视图之间的约束在移动后会被移除，需要按新的父子关系重新添加约束；此时直接设置 `frame` 没有用，下一次布局会把它覆盖。视图层级和布局都要在主线程修改。

两个分别创建的 `UIView` 是两个独立实例，即使 `frame` 和颜色相同，也不会变成同一个视图。下面在主线程创建两个父视图和两个内容视图：

```swift
let parentA = UIView()
let parentB = UIView()
let rect = CGRect(x: 0, y: 0, width: 100, height: 100)
let first = UIView(frame: rect)
let second = UIView(frame: rect)
first.backgroundColor = .blue
second.backgroundColor = .blue
parentA.addSubview(first)
parentB.addSubview(second)

print(first === second) // false
first.removeFromSuperview()
print(first.superview == nil)     // true
print(second.superview === parentB) // true
```

移除 `first` 只解除 `first` 与 `parentA` 的父子关系，不会移除 `second`。如果写成 `let second = first`，两个变量才指向同一实例；把 `second` 加入另一个父视图会移动这一个视图。对象身份的比较规则见[相等规则与对象身份](../language/swift-memory-and-references.md#相等规则与对象身份)。

### UIStackView 与 UIScrollView 的继承和职责

`UIStackView` 不是 `UIScrollView` 的子类，两者都直接继承自 `UIView`。

`UIStackView` 负责排列视图。`arrangedSubviews` 是由 `UIStackView` 自动安排布局的子视图数组。`UIStackView` 根据 `axis` 指定的水平或垂直方向，以及间距、对齐等设置排列这些子视图。`UIStackView` 本身不提供滚动能力。

`UIScrollView` 负责内容滚动。`UIScrollView` 改变内容相对可见区域的位置，使用户能够查看超出可见区域的内容。`UIScrollView` 不会自动把普通子视图排成一行或一列。

父子视图关系表示具体视图对象之间的包含关系，与类继承关系不同。例如，`scrollView` 是一个 `UIScrollView` 对象，`stackView` 是一个 `UIStackView` 对象。调用 `scrollView.addSubview(stackView)` 后，`scrollView` 成为 `stackView` 的父视图，但 `UIStackView` 的父类仍然是 `UIView`。

要让排列好的内容可以滚动，就把 `UIStackView` 放进 `UIScrollView`：`UIStackView` 负责排列，`UIScrollView` 负责滚动。还要用约束确定 `UIStackView` 的尺寸，`UIScrollView` 才知道内容有多大；只调用 `addSubview` 不够。

## UIView 子视图的回调与可见性

UILabel、UITextView 都是 UIView 的子类。UIView 没有 viewWillAppear、viewDidAppear 这类出现回调，这些是 UIViewController 的方法。UIView 只有视图层级变化的回调，而“加入视图层级”和“用户真正看到内容”是两回事。

**控制器与视图的职责。** UIViewController 通过 view 属性持有根视图，并管理根视图及其子视图。UIKit 在页面展示过程中调用控制器的生命周期方法；有容器控制器时，由容器控制器负责通知子控制器出现和消失。一个界面可以由多个控制器共同组成。

| UIView 回调 | 触发含义 |
|-|-|
| `willMove(toSuperview:)` | 父视图即将改变。参数表示新的父视图；移除时可以为 nil。 |
| `didMoveToSuperview()` | 父视图已经改变。此时可读取 superview 查看新的关系。 |
| `willMove(toWindow:)` | 所属窗口即将改变。参数表示新的窗口；离开窗口层级时可以为 nil。 |
| `didMoveToWindow()` | 所属窗口已经改变。此时可读取 window 查看当前所属窗口。 |

可以在 UIView、UILabel 或 UITextView 的子类中重写这些方法，在层级关系变化时执行代码。但这些方法只在父视图或窗口变化时调用，不会每次变得可见都调用；父视图回调与窗口回调之间的先后顺序也不固定。

**示例：在已经显示的页面中加入文本视图。** 创建 label，设置文字，再加入控制器的根视图：

```swift
let label = UILabel()
label.text = "你好"
view.addSubview(label)
```

label 的父视图变了，所以 label 收到父视图变化回调。根视图已经在窗口中，所以 label 的 window 也从 nil 变为这个窗口，label 还会收到窗口变化回调。控制器不会因为新增了一个 label 就再次收到 viewDidAppear。

**在窗口中不等于用户看得见。** `label.window != nil` 只说明 label 在某个窗口的视图层级中。label 或它的父视图被隐藏、透明度为零、在屏幕外、被裁剪、被其他内容挡住，用户都看不到文字。调用 didMoveToWindow 时，内容也还没有显示到屏幕上。

如果只执行 `label.isHidden = false`，父视图和窗口都没变，就不会触发上述回调。需要在“文字显示出来”时执行操作，就在修改 isHidden 的代码中直接调用；需要统计用户是否真的看到（曝光），还要检查可见面积、遮挡和停留时长。

## 容器控制器与子控制器

容器控制器是管理其他 UIViewController 的 UIViewController，负责安排这些控制器的视图如何布局、显示和切换。被容器控制器管理的控制器称为子控制器，子控制器仍然管理自己的根视图和内部的子视图。

| UIKit 标准容器控制器 | 职责 |
|-|-|
| `UINavigationController` | 管理导航栈，通过 push、pop 等操作切换页面。 |
| `UITabBarController` | 管理多个 Tab 对应的控制器，根据选中项显示对应页面。 |
| `UISplitViewController` | 管理分栏界面中的控制器，并根据显示模式组织各栏。 |

**示例：从首页切换到信息流页。** UITabBarController 管理 HomeViewController（首页）和 FeedViewController（信息流页），两个子控制器各自管理自己的根视图。

UITabBarController 已经显示，用户从首页切换到信息流页，切换正常完成。UITabBarController 切换显示的视图，并通知两个子控制器：

- HomeViewController 收到 viewWillDisappear，随后收到 viewDidDisappear。
- FeedViewController 收到 viewWillAppear，随后收到 viewDidAppear。

列表只表示每个控制器自己的 will → did 顺序，两个控制器的回调之间不一定先完成一个再开始另一个。如果信息流页的根视图还没加载，UIKit 会在显示前加载并调用 viewDidLoad；之后再切回信息流页，不会再次调用 viewDidLoad。

**视图的父子关系和控制器的父子关系要分别建立。** addSubview 建立视图之间的父子关系。自定义容器要调用 addChild 等 API 建立控制器之间的父子关系，再把子控制器的根视图加入自己的视图层级。只调用 addSubview(child.view)，子控制器不算真正嵌入，收不到正确的生命周期回调。

UIKit 自带的容器控制器已经负责通知子控制器；自定义容器则要按规则自己转发出现回调。这也说明，子控制器能收到 viewWillAppear 等回调，是因为容器通知了它；而普通的 UILabel 即使在控制器的视图层级中，也收不到这些回调。

## 分栏界面与详情控制器生命周期

### 界面外观与各栏分工

典型的分栏界面同时显示左侧列表和右侧详情。下图是 iPad 邮件应用的截图，展示这种界面的外观；下面的控制器分工，是用 UISplitViewController 实现这类界面的一般做法。

![图片展示了iPad邮件界面，左侧为邮件列表，显示“所有收件箱”，有Jenny Court、Xiaomeng Zhong等人的邮件，部分邮件被蓝色框选。右侧是选中邮件“Neighborhood garden”的正文，内容为关于邻里花园规划的讨论。图片上方标注了三个功能区域：左上角“Change mailboxes or accounts.”，右上角“Delete, move, or mark multiple messages.”，右下角“Move message to another mailbox.”，与上下文介绍的界面外观及控制器分工相呼应。](../assets/ipad-mail-split-view.png)

图示：左侧选择邮件，右侧显示选中邮件的正文。图片来源：Apple《在 iPad 上的“邮件”中查看电子邮件》。

| 对象 | 职责 |
|-|-|
| 左栏子控制器 | 展示邮件列表，响应用户选择某封邮件的操作。 |
| 右栏子控制器 | 展示选中邮件的标题、发件人和正文。 |
| `UISplitViewController` | 管理各栏的子控制器，组织各栏的尺寸、位置和显示状态。 |

用户选中邮件后，由业务代码把邮件数据传给详情控制器，UISplitViewController 只负责各栏的布局。宽度不够时，分栏界面会折叠成一次只显示一栏。

### 点击邮件是否触发生命周期回调

详情控制器是否收到 viewWillAppear、viewDidAppear，与点击本身无关，只取决于点击之后详情控制器的视图有没有出现或消失；是否调用 viewDidLoad，取决于根视图有没有在这时加载。

| 点击后的操作与条件 | 详情控制器的回调 |
|-|-|
| 复用右侧始终可见的详情控制器，只更新邮件内容。 | 不会仅因数据更新就再次触发 viewDidLoad、viewWillAppear 或 viewDidAppear。 |
| 右栏可见，将当前详情控制器替换为另一个控制器，并完成展示。 | 旧控制器收到消失回调，新控制器收到出现回调。新控制器的根视图若尚未加载，会在加载时先收到 viewDidLoad。 |
| 窄屏状态下，从邮件列表进入此前未显示的详情页面。 | 详情控制器收到出现回调，即使复用了已有控制器。只有根视图需要加载时，才会触发 viewDidLoad。 |

**复用并刷新内容的流程：** 用户点击邮件 → 业务代码传入选中的邮件数据 → 已显示的详情控制器刷新标题、发件人和正文。控制器一直显示在右栏，没有出现或消失，所以不会收到出现回调。

内容更新如果引起重新布局，会触发 viewWillLayoutSubviews、viewDidLayoutSubviews 等布局回调，但这不是出现回调。viewDidLoad 只在根视图加载时调用，也不会每选中一封邮件调用一次。

### 示例：用新的详情控制器替换旧对象

假设分栏界面正在显示，右栏可见，当前的 detailA 负责显示邮件 A。用户点击邮件 B 后，业务代码创建另一个详情控制器 detailB，再用 detailB 替换右栏的 detailA。

下面是示意代码：mailB 是选中的邮件 B；MailDetailViewController 是自定义的邮件详情控制器，初始化时传入邮件数据；splitViewController 是当前的 UISplitViewController。

```swift
let detailB = MailDetailViewController(mail: mailB)
splitViewController.setViewController(detailB, for: .secondary)
```

在双栏布局中，.secondary 是显示详情的右栏，这行代码把右栏的控制器换成 detailB。替换正常完成后，两个对象收到的回调如下：

| 控制器对象 | 回调 |
|-|-|
| `detailA` | viewWillDisappear → viewDidDisappear。 |
| `detailB` | 根视图首次加载时调用 viewDidLoad；随后在显示过程中调用 viewWillAppear → viewDidAppear。 |

表格只表示每个对象自己的回调顺序，两个对象的回调之间可能交错。detailA 和 detailB 是同一个类 MailDetailViewController 的两个不同对象。

创建控制器对象时，视图既没有加载，也没有出现。出现回调只在视图显示时调用；根视图可能在显示之前就加载，例如业务代码提前访问了 view。

## UITableView 的重用机制

UITableView 会保留已经创建的 UITableViewCell，需要显示另一行时，重新配置这个 Cell 来显示新的内容，从而少创建 Cell 和子视图。Cell 是显示一行内容的视图，不固定对应某一条数据。数据模型保存内容和状态；数据 ID 是稳定标识一条数据的值；IndexPath 由 section 和 row 组成，表示当前位置，插入、删除或重排后会改变。

### 注册、获取与重新配置

reuseIdentifier 是区分不同种类 Cell 的字符串。注册 Cell 类、注册 nib 文件或在 Storyboard 中配置原型 Cell，都是告诉表格在没有可重用的 Cell 时如何创建。注册只是登记创建方式，不会立即创建 Cell。

数据源是向表格提供行数和 Cell 的对象。表格需要显示某一行时，调用数据源的 tableView(_:cellForRowAt:)，传入目标 IndexPath。数据源按位置取出模型，再调用 dequeueReusableCell(withIdentifier:for:)：表格有这个标识的可重用 Cell 时，先调用它的 prepareForReuse() 再返回；没有时，按注册信息创建一个新的。数据源配置好 Cell 的全部内容后返回给表格。

调用带 `for:` 参数的 dequeue 方法前，必须已经注册过这个标识或配置了原型，否则运行时崩溃。重用池是表格保存可重用 Cell 的集合，Cell 离开屏幕后由表格自动放回，应用不需要手动处理。

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

页面要用属性强引用 ItemDataSource，因为 UITableView 的 dataSource 属性是弱引用。模型内容改变后，还要调用表格的刷新方法更新对应的行。

每次配置都要设置所有与数据有关的属性。如果只在 isFavorite 为 true 时设置 checkmark，为 false 时不设置 none，重用的 Cell 就会残留上一行的勾选。副标题、图片、颜色、显隐和随数据变化的约束都一样。

### prepareForReuse 与显示回调

| 位置或方法 | 职责 |
| --- | --- |
| 初始化方法或 awakeFromNib() | 建立固定子视图、固定约束和不随数据变化的外观 |
| cellForRowAt 中调用的配置方法 | 根据当前模型完整设置内容，包括没有图片或副标题的情况 |
| prepareForReuse() | 重置与内容无关的临时状态，清理 Cell 自己持有的旧任务或动画 |
| willDisplay | 响应 Cell 即将显示，可启动只在可见期间运行的动画或刷新计时器 |
| didEndDisplaying | 响应 Cell 已停止显示，可停止只在可见期间运行的动画或刷新计时器 |

prepareForReuse() 在已有的 Cell 被取出重用时调用，而不是在 Cell 离开屏幕时调用；新创建的 Cell 不会调用它。重写时必须调用 super.prepareForReuse()。内容仍要在配置方法中完整设置，不能指望 prepareForReuse() 已经清空了一切；而且用 reconfigureRows(at:) 刷新已有的 Cell 时，并不会调用 prepareForReuse()。

收藏、展开、输入的文字等需要保留的状态，要保存在数据模型中，或者按数据 ID 保存，不能只放在 Cell 上，因为 Cell 会被其他行重用。固定视图与约束的创建时机见[多元素 Cell 的配置成本](#多元素-cell-的配置成本)。

### Cell 中的倒计时与可见性

Cell 离开屏幕时不会被释放，也不会立即调用 prepareForReuse()。更新倒计时文字的 Timer 应在 willDisplay 中启动、在 didEndDisplaying 中停止，prepareForReuse() 中再清理一次旧状态。Cell 再次显示时，创建一个新的 Timer。

截止时间保存在数据模型中，Cell 保存一份副本，每次刷新用当前时间计算剩余时间。离开屏幕时只停止刷新，截止时间不变；再次显示时立即重新计算，所以离屏期间过去的时间也会算进去。Timer 的停止与引用机制见 [Timer 与 CADisplayLink 的停止与引用](runloop-timer-and-displaylink.md#停止回调与引用关系)。

下面是一个只显示倒计时的 Cell。截止时间用 Date 表示，所有方法都在主线程调用。Cell 用系统的 textLabel 显示文字；数据更新时，可以对已显示的 Cell 再次调用 configure。

```swift
final class CountdownCell: UITableViewCell {
    private var deadline: Date?
    private var timer: Timer?
    private var isDisplaying = false

    func configure(deadline: Date) {
        self.deadline = deadline
        if isDisplaying {
            startRefreshing()
        } else {
            refreshText()
        }
    }

    func beginDisplaying() {
        isDisplaying = true
        startRefreshing()
    }

    func endDisplaying() {
        isDisplaying = false
        stopRefreshing()
    }

    private func startRefreshing() {
        stopRefreshing()
        refreshText()
        guard let deadline, deadline > Date() else { return }

        let timer = Timer(timeInterval: 1, repeats: true) { [weak self] _ in
            self?.refreshText()
        }
        self.timer = timer
        RunLoop.main.add(timer, forMode: .common)
    }

    private func refreshText() {
        guard let deadline else {
            textLabel?.text = nil
            return
        }
        let remaining = max(0, Int(ceil(deadline.timeIntervalSinceNow)))
        textLabel?.text = "\(remaining) 秒"
        if remaining == 0 {
            stopRefreshing()
        }
    }

    private func stopRefreshing() {
        timer?.invalidate()
        timer = nil
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        endDisplaying()
        deadline = nil
    }

    deinit {
        timer?.invalidate()
    }
}
```

控制器注册 CountdownCell，并在 cellForRowAt 中取出 Cell、传入对应模型的 deadline。控制器作为 UITableViewDelegate 时，还需要实现：

```swift
func tableView(_ tableView: UITableView,
               willDisplay cell: UITableViewCell,
               forRowAt indexPath: IndexPath) {
    (cell as? CountdownCell)?.beginDisplaying()
}

func tableView(_ tableView: UITableView,
               didEndDisplaying cell: UITableViewCell,
               forRowAt indexPath: IndexPath) {
    (cell as? CountdownCell)?.endDisplaying()
}
```

didEndDisplaying 直接处理回调传入的 Cell，不要用 indexPath 去数据源里查找，因为数据可能已经变了。例如截止时间是 12:00:10，Cell 在 12:00:03 离屏、12:00:08 再次显示，这时显示约 2 秒，而不是离屏前的 7 秒。Cell 被重用来显示另一条数据时，configure 会用新数据的截止时间覆盖旧值。

### 预取与重用数量

预取是提前准备即将用到的内容。UIKit 可能在 Cell 进入屏幕之前就调用数据源配置 Cell；配置好的 Cell 最后可能没有显示，离开屏幕的 Cell 也可能保持原样、马上又显示同一行。所以 cellForRowAt、willDisplay、didEndDisplaying 和 prepareForReuse 的调用次数不一定一一对应。

一屏显示 10 个 Cell 时，至少需要 10 个 Cell 对象，但总共创建多少个并不固定。例如 10 行正好占满屏幕时，滚动到中间位置，顶部和底部各露出半行，就需要 11 个。预取、不同的重用标识和布局变化也会影响数量。

1,000 行数据不需要创建 1,000 个 Cell。正确重用时，Cell 对象的数量接近可见行数再多几个，但 UIKit 不保证具体数量。结构相同的 Cell 应使用同一个重用标识；结构不同的使用不同标识，不同标识之间不会互相重用。如果给每一行设置不同的标识，就无法重用了。

统计创建了多少个 Cell，应在 Cell 的初始化方法中计数（代码创建走 `init(style:reuseIdentifier:)`，nib 和 Storyboard 走 `init(coder:)`）。`cellForRowAt` 的调用次数是配置次数，`prepareForReuse` 的调用次数是重用次数，两者都不是创建次数。

异步结果与 Cell 当前数据的匹配规则见[长列表性能与异步图片复用](#长列表性能与异步图片复用)。图片请求的共享、取消与结果校验见[长列表图片任务调度](image-loading-decoding-and-memory.md#长列表图片任务调度)。

## UITableView 的行删除与批量更新

`UITableView` 只负责显示，不会从数据模型中删除数据。用户左滑删除时，`UITableView` 调用数据源的 `tableView(_:commit:forRowAt:)`，传入被删除行的 `IndexPath`。数据源先从模型中删除这条数据，再调用 `deleteRows(at:with:)` 通知表格删除这一行。`IndexPath.section` 是分区，`IndexPath.row` 是分区内的行号；传给 `deleteRows` 的就是删除前的行号。

```swift
// 假设单分区数据源的 items 初始为 ["A", "B", "C"]，行数由 items.count 提供。
func tableView(_ tableView: UITableView,
               commit editingStyle: UITableViewCell.EditingStyle,
               forRowAt indexPath: IndexPath) {
    guard editingStyle == .delete else { return }
    items.remove(at: indexPath.row)
    tableView.deleteRows(at: [indexPath], with: .left)
}
```

用户删除第二行时，传入的 `indexPath.row` 是 `1`。`items.remove(at: 1)` 把数据变成 `["A", "C"]`；`deleteRows(at:)` 使用删除前的位置 `1`，让原来显示 `B` 的那一行以动画退出。一次只删除一行时，直接调用 `deleteRows(at:with:)` 即可。

`performBatchUpdates(_:completion:)` 由应用主动调用：在闭包中修改数据源并调用行的增删方法，表格把这些更新合成一次动画，动画结束后调用 `completion`。（`tableView(_:commit:forRowAt:)` 则相反，是表格在用户确认删除时调用的数据源方法。）在批量更新中，删除的位置按更新前的数据计算，插入的位置按删除之后的数据计算：

```swift
// 假设单分区 items 初始为 ["A", "B", "C"]。
tableView.performBatchUpdates({
    items.remove(at: 1)
    items.append("D")
    tableView.deleteRows(at: [IndexPath(row: 1, section: 0)], with: .left)
    tableView.insertRows(at: [IndexPath(row: 2, section: 0)], with: .fade)
}, completion: nil)
// 更新后 items 为 ["A", "C", "D"]。
```

更新后数据源的行数，必须等于更新前的行数加上插入数、减去删除数，否则会崩溃。`reloadData()` 让表格重新读取全部分区、行数和可见行，没有删除动画，也不修改模型。用 `deleteRows(at:with:)` 或批量更新删除时，不要在同一次更新中再调用 `reloadData()`。

## UICollectionView 的职责与复用

UICollectionView 显示一组可滚动的内容。各部分分工如下：Cell 显示一项内容；DataSource 提供数量和配置好的 Cell；Delegate 处理点击等交互；Layout 提供每一项的位置和尺寸。数据按 section／item 组织，IndexPath 只表示当前位置，不能当作数据的唯一标识。

使用流程：创建并设置布局 → 注册 Cell → 设置数据源和代理 → 按需取出并配置 Cell → 按布局显示。滚动时重用 Cell，不会为所有数据一次性创建视图；每次配置都要设置全部状态，避免残留旧内容。

SnapKit 负责 UICollectionView 在页面中的位置，以及 Cell 内部控件的约束；Cell 在列表中怎么排列，由 UICollectionViewLayout 负责，例如 FlowLayout、CompositionalLayout 或自定义 Layout。

### 分区、项目与分区边距

分区（section）是对数据的一层分组；项目（item）是分区中的一项数据，Cell 是显示这个项目的可重用视图。一个分区可以显示多行 Cell，可以有头部、尾部，也可以没有。

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

使用垂直滚动的 UICollectionViewFlowLayout 时，Cell 从左往右排，一行放不下就换到下一行。所以一个分区可以有多行，分区和行是两回事。

sectionInset 是分区内容四周的边距。在垂直布局中，top 是头部与第一行 Cell 之间的留白，bottom 是最后一行 Cell 与尾部之间的留白，left 和 right 是两侧的留白。没有头部或尾部时，top 和 bottom 就是分区的上下边缘留白。

```swift
layout.sectionInset = UIEdgeInsets(top: 10, left: 16, bottom: 10, right: 16)
```

这里的 layout 是 UICollectionViewFlowLayout。边距从哪里算起，还受 sectionInsetReference 影响。行与行之间的间距由 minimumLineSpacing 控制，同一行中相邻项目的最小间距由 minimumInteritemSpacing 控制，这两个与 sectionInset 是不同的设置。

### 数据源与代理的五个重点回调

数据源向 UICollectionView 提供数量和配置好的 Cell；代理接收点击、显示变化等通知，也可以通过部分方法的返回值做出决定（例如是否允许选中）。下表是直接实现 UICollectionViewDataSource 时最常用的方法。

| 方法 | 所属协议 | 使用场景 |
| --- | --- | --- |
| collectionView(_:numberOfItemsInSection:) | UICollectionViewDataSource | 返回指定分区的项目数，例如当前分区有 20 张图片就返回 20 |
| collectionView(_:cellForItemAt:) | UICollectionViewDataSource | 获取复用 Cell，根据 indexPath 对应的数据设置图片、标题及其他展示状态，再返回 Cell |
| collectionView(_:didSelectItemAt:) | UICollectionViewDelegate | 用户选中项目后，读取对应数据并执行选择动作，例如打开详情 |
| collectionView(_:didEndDisplaying:forItemAt:) | UICollectionViewDelegate | Cell 结束显示时，停止该 Cell 的动画等显示相关工作 |
| scrollViewDidScroll(_:) | UIScrollViewDelegate | 滚动位置变化时读取 contentOffset，例如按位置调整导航栏透明度 |

前两个方法是 UICollectionViewDataSource 的必需方法。多个分区还需要通过 numberOfSections(in:) 提供分区数量；未实现该可选方法时，默认按一个分区处理。

cellForItemAt 只是准备 Cell，用户不一定看到了；UIKit 可能为了预取提前调用它，所以不能在这里统计曝光。willDisplay 表示 Cell 即将显示，也不能说明可见面积和停留时间已经达到曝光标准。

didEndDisplaying 传入的是结束显示的 Cell。此时数据可能已经插入、删除或重排，要找到对应的数据，应使用 Cell 上保存的数据 ID，而不是按 indexPath 读取当前数组。

用户拖动、松手后的减速滚动、代码修改滚动位置，都会触发 scrollViewDidScroll。contentOffset 是当前的滚动偏移。这个回调调用非常频繁，里面只能做轻量的界面更新。

### 代理协议的继承与布局回调

UICollectionViewDelegate 继承 UIScrollViewDelegate，所以 collectionView.delegate 指向的对象，可以同时实现选中回调和滚动回调，不需要另外设置滚动代理。

UICollectionViewDelegateFlowLayout 又继承 UICollectionViewDelegate。使用 UICollectionViewFlowLayout 时，可以实现 sizeForItemAt 为每一项提供不同的尺寸，或实现 insetForSectionAt 为每个分区提供不同的边距。所有项目尺寸和边距都一样时，直接设置 layout 的 itemSize 和 sectionInset 即可。

这些回调只对 UICollectionViewFlowLayout 有效，其他布局用各自的接口配置。以上代理协议中的方法都是可选的，按需实现即可。

协议继承的通用规则见[Swift 基础语法与类型系统中的协议继承](../language/swift-basics.md#协议继承与类继承的区别)。

## 页面、列表与 Cell ViewModel

MVVM 没有规定一个页面只能有一个 VM（ViewModel），也没有规定每个 View 都要有一个 VM。是否单独建一个 VM，看这部分界面有没有需要独立管理的展示数据、状态和交互逻辑。

| 范围 | 常见职责 |
|-|-|
| 页面／容器 VM | 整体展示状态、子区域组织与协调 |
| 列表 VM | 列表项数组、加载、刷新、分页、插入和删除 |
| Cell VM | 单项展示数据、样式、选中／高亮等状态及交互处理 |

列表 VM 管理列表中有哪些项、怎样排序和更新；单项 VM 管理一项的数据和展示状态；容器 VM 协调多个子区域的 VM，只有确实需要协调时才加这一层。

例如一条复杂的评论要管理内容、一级／二级评论样式、高亮、Cell 类型和高度，以及点赞、回复、删除的处理，为它单独建一个 Cell VM，职责会更清楚。如果只是显示一个固定标题或一段静态文字，没有状态和处理逻辑，直接配置 View 就行，不必为了套用 MVVM 而加一个 VM。

**VM 对应数据，Cell 是可重用的视图。** 100 条数据各有一个 VM，并不需要创建 100 个 Cell。Cell 滚动后会绑定另一条数据的 VM，绑定时要更新全部展示状态。布局、创建视图和执行动画仍由 View 负责。

## MVVM 的输入、状态输出与界面刷新

MVVM 把对象按职责分成三类：Model 表示业务数据和业务规则；View 显示界面、接收用户操作；ViewModel 处理用户输入，并把业务结果转换成 View 可以直接显示的状态。MVVM 不要求使用特定的绑定框架。

一种常见的流程是：用户操作 → View 调用 ViewModel 的方法 → ViewModel 调用业务服务（网络请求、持久化等）→ ViewModel 根据结果更新状态 → Publisher 发布新状态 → View 的订阅闭包刷新界面。View 不需要知道业务服务是怎么实现的。

输入和输出的职责不同。输入是用户想执行的动作，可以用方法、闭包或事件流表示；输出是当前的展示状态或需要执行的界面事件，可以用 `@Published`、Subject 等方式表示。闭包和 Publisher 只是通信工具，并不是 MVVM 本身。

ViewModel 决定显示什么数据和状态，View 负责实际更新图标、文字、布局和动画。异步结果返回后，界面更新要在主线程执行；用 Combine 时，可以在订阅前加 `receive(on: DispatchQueue.main)`。

View 持有协议类型的 ViewModel，可以不依赖具体的实现类，但仍依赖这个协议。用闭包或 Publisher 通信也不会消除依赖，仍要明确事件从哪里进入、状态从哪里来、订阅什么时候取消。

语言细节见[Swift 基础语法与类型系统](../language/swift-basics.md)；状态订阅与执行时序见[响应式编程与事件通知](reactive-programming.md)。

## 长列表性能与异步图片复用

数据有一两千条，Cell 也不会有一两千个：重用让 Cell 数量只取决于可见区域和预取范围。但数据模型、图片缓存、布局计算结果和没有取消的异步任务仍可能不断累积。

- CPU／主线程：反复配置 Cell、测量文字、自适应布局、解码或缩放图片、大量通知和 reload，都可能超出一帧的时间。60 Hz 屏幕一帧约 16.7 ms，120 Hz 约 8.3 ms，而且这段时间还要留给系统的工作。
- 内存：图片解码成位图后比文件大得多，文件小不代表解码后小；动画帧缓存、预取结果和模型也可能越积越多。
- 网络和调度：快速滚动时会发起大量请求，很多 Cell 马上就离开了屏幕，白白消耗带宽、解码和回调；旧请求完成后还可能把图片设置到已经重用的 Cell 上。
- GPU／渲染：大量半透明混合、复杂阴影、遮罩、过度绘制等图层效果也会造成卡顿，不能只看主线程 CPU。

优化前先测量找到瓶颈，再选择对应措施：重用 Cell 并每次设置全部状态；滚动中不要刷新整个列表；按数据 ID 缓存布局结果；按显示尺寸请求缩略图并下采样；在后台线程解码；使用有容量上限的内存和磁盘缓存；合并重复请求、限制并发数、预取即将显示的内容并取消不再需要的预取；Cell 离屏时暂停动画和视频；把同一帧内的多次界面更新合并。

Cell 重用只减少了视图的创建和销毁，不会取消网络请求，也不会清空旧状态。例如 Cell A 为数据 X 发起图片请求，之后 Cell A 被重用来显示数据 Y；X 的请求完成后如果直接设置图片，就会把 Y 的图片换成 X 的。所以设置图片前，要在主线程检查 Cell 当前绑定的数据 ID 是否还是 X，是才设置。取消旧请求也不能代替这项检查；`[weak cell]` 只是不强引用 Cell，并不能保证 Cell 还在显示 X。另一种做法是请求完成后把图片按数据 ID 保存，再刷新这个 ID 当前所在的行。

### 多元素 Cell 的配置成本

Cell 中固定的子视图和约束在初始化时创建；每次配置只更新文字、图片、显隐和需要变化的约束值，不要每次都重新创建视图或约束。是否需要精简层级、减少布局，要看实际测量结果，不能只看 Cell 中元素多不多。

局部数据变化时，只刷新受影响的项，不要在滚动中反复刷新整个列表。图片的目标尺寸、后台解码与缓存策略见[iOS 图片加载、解码与内存](image-loading-decoding-and-memory.md)。

### 不同高度 Cell 的测量与缓存

高度缓存是把某条数据在特定内容和布局条件下算出的高度保存下来，下次直接读取。缓存保存的是高度数值，不是 Cell 对象，也不会让 Auto Layout 跳过 Cell 内部的布局。

先测量确认文字测量和布局计算确实耗时，再考虑缓存。第一次需要高度时计算并保存，之后同样条件下直接读取，并且要真正把读到的值返回给列表的高度回调。如果只是把高度存进字典，列表仍按原方式重新计算，缓存就没有作用。

缓存的键要用稳定的数据 ID，并包含内容版本、可用宽度、字体字号、展开状态等所有影响高度的条件。IndexPath 会因插入、删除、重排而变化，不能当作键。

例如，数据 A 测得高度 180 点。A 离屏后，原来的 Cell 被重用来显示 B，但 A 的高度缓存还在。用户滚动回来时，无论拿到的是哪个 Cell，都按 A 的数据 ID 查到 180，不用再算。

能否使用缓存，由缓存键中的条件是否完全相同决定，而不是靠 Cell 记住上次的值。下面的示例假设整个列表使用同一套排版配置，内容版本由数据层维护：

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

contentVersion 在正文、图片比例等影响高度的内容改变时加 1；typographyVersion 在字体、字号、行距、换行或行数设置改变时加 1；layoutVersion 在边距、控件排列等布局规则改变时加 1。availableWidth 是这次计算实际使用的宽度，isExpanded 是当前的展开状态。版本号要由修改数据或配置的代码负责更新，不会自己变化；只用数据 ID 作键，内容变了也查到旧高度。

例如，A 的内容版本为 3、宽度 343、排版版本 1、未展开、布局版本 1，第一次计算后保存高度 180。滚动回来时生成的键完全相同，直接读到 180；正文更新后内容版本变为 4，新的键查不到，于是重新计算并保存。

缓存放在列表控制器或布局管理对象中，不受 Cell 重用影响。使用手动行高时，在 tableView(_:heightForRowAt:) 中用当前条件生成键查缓存，查不到就计算并保存，再返回高度。如果仍使用 UITableView.automaticDimension，高度由系统计算，应用自己的字典不起作用。

总之，自动行高和手动缓存是两种不同的高度来源，接入缓存时要确认列表实际用的是哪一个。

### 优化效果的验证

在同一台设备上，用同一批内容、相同的缓存状态和相似的滚动操作，对比优化前后的表现。先看卡顿次数、卡顿时长和内存峰值，再结合 CPU 调用栈和渲染阶段判断瓶颈是否减轻。网络请求少了、平均 CPU 占用低了，并不能说明滚动不卡了。

## 布局更新与列表定位时机

| 方法 | 作用 | 常见使用方式 |
|-|-|-|
| `setNeedsLayout()` | 标记需要重新布局，允许系统合并更新，调用时通常不立即执行布局 | 内容或布局条件改变后请求后续布局 |
| `layoutIfNeeded()` | 如果有待处理布局更新，立即处理相关视图子树；没有则无需重复布局 | 修改约束后，需要获得当前几何信息或构造布局动画 |
| `layoutSubviews()` | UIKit 执行布局时调用的回调，可重写以布置子视图 | 自定义 UIView 内实现布局；通常不直接调用它来强制刷新 |

修改一条约束可能影响父视图和兄弟视图，所以要对包含这些视图的父视图调用 layoutIfNeeded。这三个方法只与布局有关，布局完成后还要经过绘制和渲染，画面才会显示到屏幕上，渲染过程见[渲染管线与显示阶段](rendering-and-performance.md#图层动画与渲染管线)。

**reload 后定位：**

`reloadData` 返回时，表格只是标记了需要重新加载，自适应高度、Cell 位置和之后加载的图片尺寸都还没确定。这时立刻按 contentSize 或 Cell 位置滚动，目标位置可能算错。

用 `DispatchQueue.main.async` 推迟一下，也不能保证布局已经完成。应根据更新方式选择可靠的完成时机，例如在 `performBatchUpdates` 的 completion 中，或者对表格调用 `layoutIfNeeded()` 后再按最新布局定位。如果异步加载的图片还会改变 Cell 高度，就要在图片加载完成、重新布局后再定位一次。多套几层 async、固定延迟，都不是可靠的做法。
