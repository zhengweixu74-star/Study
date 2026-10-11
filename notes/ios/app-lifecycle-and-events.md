# iOS 应用启动、窗口与输入事件

[返回笔记目录](../README.md)

## 本文目录

- [UIApplication、AppDelegate、UIWindow 与 UIView 的职责](#uiapplicationappdelegateuiwindow-与-uiview-的职责)
- [AppDelegate 生命周期下，从启动到首帧显示](#appdelegate-生命周期下从启动到首帧显示)
- [UIApplication 的使用场景：打开应用设置](#uiapplication-的使用场景打开应用设置)
- [为什么一个应用可以有多个 UIWindow](#为什么一个应用可以有多个-uiwindow)
- [关键窗口与 makeKeyAndVisible()](#关键窗口与-makekeyandvisible)
- [UIApplication 与 UIWindow 为什么都分发事件](#uiapplication-与-uiwindow-为什么都分发事件)
- [屏幕键盘：触摸位置与文字接收对象](#屏幕键盘触摸位置与文字接收对象)

分类：iOS 界面、事件与调度。本文整理应用启动、窗口职责、事件分发和文本输入；以传统 AppDelegate 生命周期为主要背景。

## UIApplication、AppDelegate、UIWindow 与 UIView 的职责

UIApplication 是 UIKit 提供的应用对象，负责应用级事件分发、应用状态以及与系统交互的接口。AppDelegate 是开发者提供的应用代理对象，通过 UIApplicationDelegate 协议接收应用级回调，并执行自定义初始化等逻辑。

| 对象 | 主要职责 | 例子 |
|-|-|-|
| UIApplication | 管理应用级行为，并将事件分发给适当的对象 | 通过 UIApplication.shared.open(...) 请求系统打开设置页面 |
| AppDelegate | 响应应用生命周期和应用级回调 | 在 application(\_:didFinishLaunchingWithOptions:) 中初始化必要服务和配置主窗口 |
| UIWindow | 承载视图层级，并参与向视图分发事件 | 设置 rootViewController、显示窗口、成为关键窗口 |
| UIView | 表示界面中的一块区域，展示内容并参与交互 | 按钮接收点击；UITextField 作为 UIView 的子类接收文字输入 |

UIApplication 与 AppDelegate 是不同对象。UIKit 调用 AppDelegate 的回调方法，让开发者在指定时机执行代码；普通按钮触摸并不需要先交给 AppDelegate 处理。

## AppDelegate 生命周期下，从启动到首帧显示

适用条件：应用采用传统 AppDelegate 生命周期，以代码创建界面，并从未运行的状态进入前台。这里不包含系统预热、状态恢复等额外路径。若应用仍在运行，从后台回到前台不会重新执行整套启动流程。

1. 系统准备并创建应用进程。dyld 是系统的动态加载器，负责加载可执行文件及依赖，并完成必要的链接工作。程序在进入 main 前还会执行必要的运行时初始化和启动初始化代码。
2. 程序入口进入 UIApplicationMain。UIApplicationMain 创建 UIApplication 和指定的 AppDelegate，建立代理关系，并设置应用主事件循环。主事件循环持续接收并分发事件。
3. UIKit 依次调用 application(\_:willFinishLaunchingWithOptions:) 和 application(\_:didFinishLaunchingWithOptions:)。应用在这些回调中完成启动所需的初始化，并按需要检查启动参数。
4. 在常见的纯代码实现中，AppDelegate 在 didFinishLaunchingWithOptions 中创建 UIWindow，设置 rootViewController，并调用 makeKeyAndVisible()。rootViewController 是管理窗口主要视图层级的根视图控制器。
5. 显示窗口会促使 UIKit 加载所需的视图。视图控制器的视图尚未加载时，UIKit 通过 loadView 创建视图，并在加载完成后调用 viewDidLoad。UIKit 随显示流程执行必要的布局和绘制。
6. 应用提交准备好的界面内容，系统完成渲染和合成后显示首帧。首帧是用户第一次看到的应用界面，不代表网络请求和所有业务数据已经完成加载。

didFinishLaunchingWithOptions 返回、makeKeyAndVisible() 返回，都不能单独证明首帧已经显示。applicationDidBecomeActive(\_:) 表示应用进入活动状态，也不是精确的首帧显示回调。采用 Storyboard 自动创建界面时，窗口与根控制器的创建方式会不同。

### 进程创建时间与启动计时起点

进程创建时间是系统记录的进程开始存在的时刻。进入 `main()` 的时间是程序执行到入口的时刻，两者不同。对于新创建的 App 进程，进程创建发生在进入 `main()` 之前，因此只从 `main()` 开始记录当前时间，会遗漏这之前的启动阶段。

`sysctl` 是 Darwin 系统提供的 C 接口，用于查询系统信息，也允许具有相应权限的调用方修改部分系统信息。PID 是操作系统分配给进程的编号。查询当前进程信息时，调用方提供当前 PID，系统将允许查询的结果写入调用方提供的缓冲区。

1. 获取当前进程的 PID。
2. 通过 `sysctl` 的 `CTL_KERN → KERN_PROC → KERN_PROC_PID → PID` 查询路径请求进程信息。
3. 查询成功并获得有效结果后，从 `kinfo_proc` 结构中的 `kp_proc.p_un.__p_starttime` 读取进程启动时间。
4. 将秒数 `tv_sec` 与微秒数 `tv_usec` 合并：时间戳（秒）= `tv_sec + tv_usec / 1_000_000.0`。时间戳表示相对于约定时间原点的时间值，不是耗时。

查询时间与被查询的事件时间需要分开。例如：10:00:00.000 创建进程，10:00:00.200 进入 `main()`，10:00:00.300 查询进程创建时间；查询得到的仍是 10:00:00.000，而不是调用查询接口时的时间。

统计前应明确起点和终点的含义。以进程创建时间为起点，可以覆盖 `main()` 之前的启动阶段，但进程创建时间不等于用户点击图标的时间。系统预热是系统在用户主动打开 App 前提前执行部分启动工作；存在预热时，需要区分预热阶段与用户实际等待阶段。用于相减的时间值还需要采用一致的时钟基准和单位。

## UIApplication 的使用场景：打开应用设置

用户拒绝照片访问后，应用可以提供“去设置”按钮。按钮被点击后，业务代码使用 UIApplication.openSettingsURLString 构造 URL，再通过 UIApplication.shared.open(...) 请求系统打开本应用的设置页面。

这里的职责顺序是：按钮接收点击 → 业务代码调用 UIApplication 的接口 → 系统决定是否打开目标设置页面。UIApplication 提供系统交互入口，页面切换由系统执行。

## 为什么一个应用可以有多个 UIWindow

UIWindow 是应用界面中的对象，不是进程，也不是必须带边框、可以单独拖动的桌面窗口。进程是应用代码运行的环境；同一个应用进程可以创建多个 UIWindow。

多数应用只需要一个主 UIWindow。额外窗口可以承载额外内容，并与主窗口叠加显示。用户看到的完整画面可能由多个窗口共同组成，因此“屏幕上只看到一个应用”不能证明应用内部只有一个 UIWindow。

例如，开发者可以让主 UIWindow 显示首页，再用背景透明的额外 UIWindow 显示悬浮调试按钮。用户看到的是首页上叠加一个按钮。这里仅说明一种实现方式；悬浮按钮也可以直接使用普通 UIView 实现。

切换首页和详情页通常通过视图控制器或视图切换完成，不需要为每个页面新建 UIWindow。

## 关键窗口与 makeKeyAndVisible()

关键窗口是 UIKit 指定的、接收没有相关触摸坐标的键盘等事件的窗口。关键窗口状态用于确定事件接收范围；窗口是否可见则由显示状态决定，两者不是同一个概念。

| 操作 | 作用 |
|-|-|
| makeKey() | 请求窗口成为关键窗口，不负责显示隐藏的窗口 |
| isHidden = false | 显示窗口，不等同于将窗口设为关键窗口 |
| makeKeyAndVisible() | 显示窗口，并将窗口设为关键窗口 |

makeKeyAndVisible() 还会把窗口放到同级或更低级别窗口的前面。窗口级别由 windowLevel 表示。关键窗口状态与窗口叠放次序需要分别理解。

非关键窗口仍可能显示，也仍可能接收命中该窗口的触摸。不能把“非关键窗口”理解为“不可见”或“不能点击”。

“只有一个关键窗口”需要限定范围。传统应用的讨论通常以应用范围说明；使用多场景机制时，应按场景理解关键窗口，不能推广成整个设备只有一个。

## UIApplication 与 UIWindow 为什么都分发事件

事件分发是把事件交给适当接收对象的过程。UIApplication 与 UIWindow 参与的层级不同，因此两者都参与分发并不矛盾。

以触摸事件为例，可以简化为：UIApplication → 对应的 UIWindow → 窗口内的目标 UIView。UIApplication 把事件分发到窗口；UIWindow 再把触摸事件交给相应视图。手势识别器等也可能参与，这条链路只表达本次讨论的主要职责。

命中测试是根据触摸位置和视图层级寻找目标视图的过程。响应链则是由响应对象之间的 next 关系构成的传递路径。命中测试负责寻找触摸目标，响应链负责描述事件继续传递的关系，二者需要区分。

### UIView 的局部坐标与 bounds

UIView 的局部坐标系用于描述视图内部的位置。默认情况下，bounds.origin 为 (0, 0)，可见矩形的左上角对应局部坐标 (0, 0)，x 轴向右，y 轴向下。

对于 bounds 为 CGRect(x: 0, y: 0, width: 50, height: 50) 的按钮，局部中心点是 (25, 25)，右下角的边界坐标是 (50, 50)。这里的尺寸单位是点（pt）。按钮放在父视图中的哪个位置，不会自动改变这些局部坐标。

bounds 描述视图在自身坐标系中的矩形；frame 描述视图在父视图坐标系中的矩形。修改 bounds.origin 后，可见矩形左上角对应的局部坐标会随之改变，不能始终把左上角认定为 (0, 0)。UIView.center 属性使用父视图坐标系，也不同于局部中心点。

### point(inside:with:) 与 hitTest(_:with:)

point(inside:with:) 判断一个局部坐标点是否属于当前视图的命中区域，返回 Bool。hitTest(_:with:) 在当前视图及其子视图中寻找触摸目标，返回 UIView?；找不到目标时返回 nil。

UIView 默认命中测试可以按以下顺序理解：

1. 当前视图隐藏、关闭用户交互或透明度低于 UIKit 的命中阈值时，返回 nil。
2. 调用当前视图的 point(inside:with:)。结果为 false 时，返回 nil，并跳过当前视图的子树。
3. 按从前向后的顺序检查子视图。先把触摸点转换到对应子视图的坐标系，再调用子视图的 hitTest(_:with:)。
4. 某个子视图找到目标时，返回该目标。所有子视图都没有找到目标时，返回当前视图。

两个方法接收的 point 都采用方法接收者自身的坐标系。重写命中区域时，通常应把 point 与 bounds 比较，不能直接与父视图坐标系中的 frame 比较。

### 扩大按钮的命中区域

重写 point(inside:with:) 可以扩大命中区域，同时保持按钮的显示尺寸和布局尺寸。下面的按钮向四周各扩大 10 pt；当 bounds 为 50×50 pt 时，命中矩形为 70×70 pt。

```swift
final class ExpandedHitButton: UIButton {
    override func point(
        inside point: CGPoint,
        with event: UIEvent?
    ) -> Bool {
        let hitBounds = bounds.insetBy(dx: -10, dy: -10)
        return hitBounds.contains(point)
    }
}
```

扩大后的区域只有在祖先视图允许命中测试继续进入该子树时才有效。如果触摸位置位于父视图的命中区域之外，父视图默认会直接返回 nil，按钮就没有机会判断该位置。需要根据布局调整父视图的命中范围或命中测试转发。

clipsToBounds 控制是否裁剪超出边界的绘制内容。把 clipsToBounds 设为 false，不会自动扩大父视图或按钮的命中区域。

### 同一按钮的点击与长按协调

按钮的短按动作可以使用 touchUpInside。长按可以使用 UILongPressGestureRecognizer。把长按识别器直接添加到按钮上，可以让长按从按钮的命中区域内开始识别；仅与按钮发生视觉重叠的兄弟视图，不会因此收到这次触摸。

以下代码位于 UIViewController 子类中。button 是已经创建的 UIButton 属性；setupInteractions() 在界面初始化时调用一次。示例仅展示事件协调。

```swift
private func setupInteractions() {
    button.addTarget(self, action: #selector(handleTap), for: .touchUpInside)

    let longPress = UILongPressGestureRecognizer(
        target: self,
        action: #selector(handleLongPress(_:))
    )
    longPress.minimumPressDuration = 0.5
    longPress.cancelsTouchesInView = true
    button.addGestureRecognizer(longPress)
}

@objc private func handleTap() {
    print("短按完成")
}

@objc private func handleLongPress(_ gesture: UILongPressGestureRecognizer) {
    switch gesture.state {
    case .began:
        print("长按开始")
    case .ended, .cancelled:
        print("长按结束，恢复临时状态")
    default:
        break
    }
}
```

按住时间不足 0.5 秒、且按钮仍满足正常点击条件时，用户松手会触发 touchUpInside，不需要等待剩余时间。达到最短时长且满足触点移动距离等识别条件时，长按进入 began。

cancelsTouchesInView 为 true 时，已经识别成功的长按会取消发送给按钮的触摸。按钮因此不会在这次长按松手时再触发 touchUpInside。该属性取消的是按钮的触摸，不是把长按识别器自身改成 cancelled。

如果按下时就执行动作，例如使用 touchDown，长按识别成功时该动作已经发生。后续取消触摸不能撤销已经执行的动作。

手势识别器也会观察命中其附属视图的子视图的触摸。因此，若将识别器添加到整个页面的根视图上，长按的起始范围就可能扩大到页面内的其他视图。按钮已经扩大的命中区域也会参与起始位置判断。

### 长按状态与取消后的恢复

UILongPressGestureRecognizer 是连续手势识别器。连续手势在识别开始后还会跟踪触摸变化，并在正常结束或取消时通知目标对象。

| 状态 | 含义 | 适合执行的操作 |
| --- | --- | --- |
| .began | 达到长按识别条件 | 保存原状态，开启临时效果 |
| .changed | 长按已经开始，触摸信息发生变化 | 按需要更新连续交互 |
| .ended | 已识别的长按正常结束，例如用户松手 | 恢复原状态，结束临时效果 |
| .cancelled | 已识别的长按被中断 | 同样恢复原状态并清理 |
| .failed | 在识别开始前未满足条件 | 不执行仅属于长按成功路径的操作 |

例如，长按开始后，如果来电界面接管交互并导致 UIKit 取消当前触摸，识别器可能进入 cancelled。不是所有来电横幅都会取消触摸，判断依据是系统是否实际中断了这次触摸。

长按已经开始后，手指移出按钮不会自动表示手势取消。若要求移出时结束效果，需要显式检查位置并定义结束规则；这与限制“长按从哪里开始”是两个条件。

临时效果应在 began 时保存原状态，并在 ended 和 cancelled 时恢复该状态。不能只处理正常松手，否则发生取消时可能残留临时状态。

## 屏幕键盘：触摸位置与文字接收对象

第一响应者是当前优先接收特定类型事件的响应对象。在文本输入场景中，成为第一响应者的输入框是当前文字输入目标；这不意味着所有触摸事件都交给该输入框。

假设页面同时显示用户名框和密码框，使用手机自带的屏幕键盘输入：

1. 用户点击密码框。UIKit 根据触摸位置找到密码框，密码框成为第一响应者，系统显示屏幕键盘。
2. 用户点击屏幕键盘上的 A 键。这次触摸有坐标，键盘利用该坐标判断用户点击了哪个键。
3. 系统把输入的文字交给当前接收文字输入的密码框。A 键所在的位置不负责判断文字应写入用户名框还是密码框。
4. 用户切换到用户名框后，再点击同一个 A 键，文字会输入到用户名框。键盘位置可以不变，但第一响应者已经改变。

因此，准确表述是：点击屏幕键盘有触摸坐标，但文字接收对象不由本次点击键盘的坐标决定。选择键盘上的按键与选择文字输入目标是两个不同步骤。

关键窗口解决窗口范围的问题，第一响应者解决具体接收对象的问题。对于标准文本输入控件，控件所在窗口需要是关键窗口，控件成为第一响应者后才能正常唤起屏幕键盘。因此，关键窗口也与日常手机输入直接相关。

外接键盘的按键事件没有对应的屏幕触摸位置，是另一种输入情况。判断是否有可用位置，应看事件包含的信息，而不是看设备是否位于 App 外部。
