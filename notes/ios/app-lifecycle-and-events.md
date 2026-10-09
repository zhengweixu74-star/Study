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

参考：[UIApplication](https://developer.apple.com/documentation/uikit/uiapplication)

## AppDelegate 生命周期下，从启动到首帧显示

适用条件：应用采用传统 AppDelegate 生命周期，以代码创建界面，并从未运行的状态进入前台。这里不包含系统预热、状态恢复等额外路径。若应用仍在运行，从后台回到前台不会重新执行整套启动流程。

1. 系统准备并创建应用进程。dyld 是系统的动态加载器，负责加载可执行文件及依赖，并完成必要的链接工作。程序在进入 main 前还会执行必要的运行时初始化和启动初始化代码。
2. 程序入口进入 UIApplicationMain。UIApplicationMain 创建 UIApplication 和指定的 AppDelegate，建立代理关系，并设置应用主事件循环。主事件循环持续接收并分发事件。
3. UIKit 依次调用 application(\_:willFinishLaunchingWithOptions:) 和 application(\_:didFinishLaunchingWithOptions:)。应用在这些回调中完成启动所需的初始化，并按需要检查启动参数。
4. 在常见的纯代码实现中，AppDelegate 在 didFinishLaunchingWithOptions 中创建 UIWindow，设置 rootViewController，并调用 makeKeyAndVisible()。rootViewController 是管理窗口主要视图层级的根视图控制器。
5. 显示窗口会促使 UIKit 加载所需的视图。视图控制器的视图尚未加载时，UIKit 通过 loadView 创建视图，并在加载完成后调用 viewDidLoad。UIKit 随显示流程执行必要的布局和绘制。
6. 应用提交准备好的界面内容，系统完成渲染和合成后显示首帧。首帧是用户第一次看到的应用界面，不代表网络请求和所有业务数据已经完成加载。

didFinishLaunchingWithOptions 返回、makeKeyAndVisible() 返回，都不能单独证明首帧已经显示。applicationDidBecomeActive(\_:) 表示应用进入活动状态，也不是精确的首帧显示回调。采用 Storyboard 自动创建界面时，窗口与根控制器的创建方式会不同。

参考：[应用启动流程](https://developer.apple.com/documentation/uikit/responding-to-the-launch-of-your-app)

参考：[缩短启动时间与首帧度量](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)

### 进程创建时间与启动计时起点

进程创建时间是系统记录的进程开始存在的时刻。进入 `main()` 的时间是程序执行到入口的时刻，两者不同。对于新创建的 App 进程，进程创建发生在进入 `main()` 之前，因此只从 `main()` 开始记录当前时间，会遗漏这之前的启动阶段。

`sysctl` 是 Darwin 系统提供的 C 接口，用于查询系统信息，也允许具有相应权限的调用方修改部分系统信息。PID 是操作系统分配给进程的编号。查询当前进程信息时，调用方提供当前 PID，系统将允许查询的结果写入调用方提供的缓冲区。

1. 获取当前进程的 PID。
2. 通过 `sysctl` 的 `CTL_KERN → KERN_PROC → KERN_PROC_PID → PID` 查询路径请求进程信息。
3. 查询成功并获得有效结果后，从 `kinfo_proc` 结构中的 `kp_proc.p_un.__p_starttime` 读取进程启动时间。
4. 将秒数 `tv_sec` 与微秒数 `tv_usec` 合并：时间戳（秒）= `tv_sec + tv_usec / 1_000_000.0`。时间戳表示相对于约定时间原点的时间值，不是耗时。

查询时间与被查询的事件时间需要分开。例如：10:00:00.000 创建进程，10:00:00.200 进入 `main()`，10:00:00.300 查询进程创建时间；查询得到的仍是 10:00:00.000，而不是调用查询接口时的时间。

参考：[Apple：sysctl 接口](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/sysctl.3.html)；[Apple XNU：进程启动时间字段](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/sys/proc.h)。

统计前应明确起点和终点的含义。以进程创建时间为起点，可以覆盖 `main()` 之前的启动阶段，但进程创建时间不等于用户点击图标的时间。系统预热是系统在用户主动打开 App 前提前执行部分启动工作；存在预热时，需要区分预热阶段与用户实际等待阶段。用于相减的时间值还需要采用一致的时钟基准和单位。

## UIApplication 的使用场景：打开应用设置

用户拒绝照片访问后，应用可以提供“去设置”按钮。按钮被点击后，业务代码使用 UIApplication.openSettingsURLString 构造 URL，再通过 UIApplication.shared.open(...) 请求系统打开本应用的设置页面。

这里的职责顺序是：按钮接收点击 → 业务代码调用 UIApplication 的接口 → 系统决定是否打开目标设置页面。UIApplication 提供系统交互入口，页面切换由系统执行。

参考：[openSettingsURLString](https://developer.apple.com/documentation/uikit/uiapplication/opensettingsurlstring)

## 为什么一个应用可以有多个 UIWindow

UIWindow 是应用界面中的对象，不是进程，也不是必须带边框、可以单独拖动的桌面窗口。进程是应用代码运行的环境；同一个应用进程可以创建多个 UIWindow。

多数应用只需要一个主 UIWindow。额外窗口可以承载额外内容，并与主窗口叠加显示。用户看到的完整画面可能由多个窗口共同组成，因此“屏幕上只看到一个应用”不能证明应用内部只有一个 UIWindow。

例如，开发者可以让主 UIWindow 显示首页，再用背景透明的额外 UIWindow 显示悬浮调试按钮。用户看到的是首页上叠加一个按钮。这里仅说明一种实现方式；悬浮按钮也可以直接使用普通 UIView 实现。

切换首页和详情页通常通过视图控制器或视图切换完成，不需要为每个页面新建 UIWindow。

参考：[UIWindow](https://developer.apple.com/documentation/uikit/uiwindow)

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

参考：[makeKeyAndVisible()](https://developer.apple.com/documentation/uikit/uiwindow/makekeyandvisible())

参考：[关键窗口的场景范围](https://developer.apple.com/documentation/UIKit/UIWindow/becomeKey())

## UIApplication 与 UIWindow 为什么都分发事件

事件分发是把事件交给适当接收对象的过程。UIApplication 与 UIWindow 参与的层级不同，因此两者都参与分发并不矛盾。

以触摸事件为例，可以简化为：UIApplication → 对应的 UIWindow → 窗口内的目标 UIView。UIApplication 把事件分发到窗口；UIWindow 再把触摸事件交给相应视图。手势识别器等也可能参与，这条链路只表达本次讨论的主要职责。

命中测试是根据触摸位置和视图层级寻找目标视图的过程。响应链则是由响应对象之间的 next 关系构成的传递路径。命中测试负责寻找触摸目标，响应链负责描述事件继续传递的关系，二者需要区分。

参考：[UIApplication.sendEvent(\_:)](https://developer.apple.com/documentation/uikit/uiapplication/sendevent(_:))

参考：[UIWindow.sendEvent(\_:)](https://developer.apple.com/documentation/uikit/uiwindow/sendevent(_:))

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

参考：[becomeFirstResponder()](https://developer.apple.com/documentation/uikit/uiresponder/becomefirstresponder())

参考：[UIKeyInput](https://developer.apple.com/documentation/uikit/uikeyinput)
