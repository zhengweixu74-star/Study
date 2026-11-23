# iOS 应用生命周期、启动与窗口

[返回笔记目录](../README.md)

## 本文目录

- [UIApplication、AppDelegate、UIWindow 与 UIView 的职责](#uiapplicationappdelegateuiwindow-与-uiview-的职责)
  - [UIApplication 与主线程 RunLoop](#uiapplication-与主线程-runloop)
- [应用运行状态与生命周期回调](#应用运行状态与生命周期回调)
  - [五种运行状态](#五种运行状态)
  - [AppDelegate 的主要回调](#appdelegate-的主要回调)
  - [常见状态变化与调用顺序](#常见状态变化与调用顺序)
  - [进入前台与进入激活状态的区别](#进入前台与进入激活状态的区别)
  - [挂起、终止与数据保存](#挂起终止与数据保存)
  - [Scene 生命周期与应用生命周期](#scene-生命周期与应用生命周期)
- [AppDelegate 生命周期下，从启动到首帧显示](#appdelegate-生命周期下从启动到首帧显示)
  - [main 之前的启动工作与 dyld](#main-之前的启动工作与-dyld)
  - [LaunchScreen 与自定义启动页](#launchscreen-与自定义启动页)
- [启动耗时的测量与优化](#启动耗时的测量与优化)
  - [进程创建时间与启动计时起点](#进程创建时间与启动计时起点)
  - [启动优化的三个方向](#启动优化的三个方向)
  - [动态库数量与启动开销](#动态库数量与启动开销)
  - [删除无用代码与地址修正成本](#删除无用代码与地址修正成本)
- [为什么一个应用可以有多个 UIWindow](#为什么一个应用可以有多个-uiwindow)
- [关键窗口与 makeKeyAndVisible()](#关键窗口与-makekeyandvisible)

## UIApplication、AppDelegate、UIWindow 与 UIView 的职责

UIKit 应用运行时，主要由四类对象分工协作：

| 对象 | 由谁实现 | 主要职责 | 例子 |
|-|-|-|-|
| UIApplication | UIKit 创建，每个进程只有一个，通过 `UIApplication.shared` 访问 | 记录应用的运行状态，把事件分发给窗口，提供与系统交互的接口 | 调用 `UIApplication.shared.open(url)` 请求系统打开某个 URL |
| AppDelegate | 开发者编写，遵循 `UIApplicationDelegate` 协议 | 接收 UIKit 在启动完成、进入后台等时机发出的回调，执行开发者的代码 | 在 `application(_:didFinishLaunchingWithOptions:)` 中初始化服务、创建主窗口 |
| UIWindow | UIKit 提供类，开发者创建实例 | 作为视图层级的根容器，把事件交给窗口内的视图 | 设置 `rootViewController`，调用 `makeKeyAndVisible()` 显示窗口 |
| UIView | UIKit 提供类，开发者创建实例或子类 | 表示屏幕上的一块矩形区域，绘制内容并接收触摸 | `UIButton` 接收点击；`UITextField` 接收文字输入 |

UIApplication 和 AppDelegate 是两个对象：UIApplication 负责管理应用和分发事件，AppDelegate 负责在 UIKit 发出通知时执行开发者的代码。按钮点击等触摸事件由 UIApplication 交给窗口和视图，不经过 AppDelegate。

例如，用户点击“去设置”按钮后，按钮的动作方法用 `UIApplication.openSettingsURLString` 生成 URL，再调用 `UIApplication.shared.open(url)`。UIApplication 把请求转给系统，系统打开本应用的设置页面。

### UIApplication 与主线程 RunLoop

RunLoop 是在一个线程上反复执行的循环：没有事件时，RunLoop 让线程休眠；触摸、定时器等事件到达时，RunLoop 唤醒线程，并调用对应的处理代码。UIKit 把主线程上的 RunLoop 称为主事件循环。主事件循环由 `UIApplicationMain` 启动，业务代码不需要调用 `RunLoop.main.run()`。

以点击按钮为例：

1. 系统把触摸事件发给应用进程。
2. 主线程 RunLoop 被唤醒，开始处理这次触摸。RunLoop 决定“什么时候处理事件”。
3. UIApplication 把事件交给触摸所在的 UIWindow，UIWindow 再把事件交给按钮。UIApplication 和 UIWindow 决定“把事件交给谁”，具体过程见 [UIKit 触摸事件、手势与键盘输入](touch-events-and-gestures.md)。
4. 按钮执行动作方法。

主线程同一时刻只执行一段代码，RunLoop 等当前代码执行完，才处理下一个事件。如果按钮的动作方法在主线程上执行了 2 秒，新的触摸和界面刷新都要等这 2 秒结束才执行，用户就会感到卡顿。运行模式和输入源见 [iOS 线程与任务调度](threads-and-scheduling.md#runloop-与运行模式)。

## 应用运行状态与生命周期回调

应用生命周期是应用进程从启动、在前台和后台之间切换，到最终终止的整个过程。生命周期回调是 UIKit 在应用状态变化时调用的代理方法。代理方法是 UIKit 定义在协议中、由开发者的对象实现、在指定时机由 UIKit 调用的方法，例如 AppDelegate 实现的 `UIApplicationDelegate` 方法。

本节按传统 AppDelegate 生命周期说明。启用 Scene 生命周期后，界面状态的变化改为通知 SceneDelegate，见 [Scene 生命周期与应用生命周期](#scene-生命周期与应用生命周期)。

### 五种运行状态

| 状态 | 含义 | 需要注意的地方 |
|-|-|-|
| Not Running（未运行） | 应用进程还没启动，或已经终止 | 再次打开应用时，系统要重新创建进程 |
| Inactive（未激活） | 应用在前台，但暂时不接收触摸等用户交互 | 应用代码仍在执行，网络请求完成后的回调照常执行 |
| Active（已激活） | 应用在前台，并接收用户交互 | Active 只表示应用可以接收交互；数据加载和首帧显示各有自己的完成时机 |
| Background（后台） | 应用不在前台，但在系统允许的时间内还能执行代码 | 普通应用做完收尾工作后会被挂起；声明了音频播放、定位等后台能力的应用，对应工作可以继续运行 |
| Suspended（挂起） | 进程还在内存中，但不执行任何代码 | 系统需要内存时可以直接终止挂起的进程，内存中未保存的数据随之丢失 |

前台包括 Inactive 和 Active 两种状态，应用回到前台时先处于 Inactive，之后才变为 Active。代码能读到的 `UIApplication.shared.applicationState` 只有 `.active`、`.inactive`、`.background` 三个值，因为应用处于 Not Running 或 Suspended 时，应用代码根本不执行。

### AppDelegate 的主要回调

| 回调 | UIKit 调用时机 | 常见用途 |
|-|-|-|
| `application(_:didFinishLaunchingWithOptions:)` | 启动流程即将完成，应用即将开始运行 | 初始化启动必需的服务；用代码创建界面时，在这里创建主窗口 |
| `applicationWillResignActive(_:)` | 应用即将从 Active 变为 Inactive | 暂停依赖用户交互的工作，例如暂停游戏、停止计时器 |
| `applicationDidEnterBackground(_:)` | 应用已经进入 Background | 保存还没保存的数据，释放可以重新创建的资源 |
| `applicationWillEnterForeground(_:)` | 应用即将从 Background 回到前台，先进入 Inactive | 检查后台期间发生的数据变化，按需刷新 |
| `applicationDidBecomeActive(_:)` | 应用已经从 Inactive 变为 Active | 恢复因失去激活而暂停的工作 |
| `applicationWillTerminate(_:)` | 应用即将终止，并且系统有机会通知应用 | 执行简短的清理。系统直接终止挂起进程时不调用这个方法，所以数据要在更早的时机保存 |

UIKit 只负责在对应时机调用这些方法。计时器是否暂停、数据是否刷新，取决于开发者在方法里写了什么代码。

### 常见状态变化与调用顺序

下面三个流程都假设状态切换正常完成。

**流程 1：从未运行直接启动到前台。** 应用没有经过后台，所以 UIKit 不调用 `applicationWillEnterForeground(_:)`。

```text
状态：Not Running → 启动初始化 → Inactive → Active
回调：application(_:willFinishLaunchingWithOptions:)
   → application(_:didFinishLaunchingWithOptions:)
   → applicationDidBecomeActive(_:)
```

**流程 2：切到后台，再切回前台。** 应用离开时先失去激活，再进入后台；回来时先回到前台，再恢复激活。

```text
离开：Active → Inactive → Background →（可能）Suspended
回调：applicationWillResignActive(_:) → applicationDidEnterBackground(_:)

返回：（如果已挂起，先恢复执行）Background → Inactive → Active
回调：applicationWillEnterForeground(_:) → applicationDidBecomeActive(_:)
```

如果系统在后台期间终止了进程，用户再次打开应用时走流程 1。

**流程 3：前台短暂中断。** 例如用户下拉控制中心后又收起，应用只进入 Inactive，没有进入 Background。

```text
状态：Active → Inactive → Active
回调：applicationWillResignActive(_:) → applicationDidBecomeActive(_:)
```

来电、系统弹窗出现时，应用可能只进入 Inactive，也可能进入 Background。UIKit 按实际发生的状态变化调用回调。

### 进入前台与进入激活状态的区别

`applicationWillEnterForeground(_:)` 只在流程 2 中调用，表示“应用即将从后台回到前台”。`applicationDidBecomeActive(_:)` 在三个流程中都会调用，表示“应用已经可以接收用户交互”。所以代码放在哪个回调，取决于代码的执行条件：

- “每次从后台回来都刷新数据”放在 `applicationWillEnterForeground(_:)`。
- “每次恢复交互时继续暂停的任务”放在 `applicationDidBecomeActive(_:)`。

在 `applicationWillEnterForeground(_:)` 中发起异步网络请求后，UIKit 照常继续切换状态，并调用 `applicationDidBecomeActive(_:)`。所以 `applicationDidBecomeActive(_:)` 执行时，数据可能还没返回。应用可以先显示已有数据，请求完成后再更新界面。

### 挂起、终止与数据保存

挂起的应用不执行代码。系统直接终止挂起的进程时，应用没有代码在运行，所以 `applicationWillTerminate(_:)` 不会被调用，数据也来不及保存。用户在应用切换器中上滑关闭应用时，`applicationWillTerminate(_:)` 同样可能不被调用。

持久保存是把数据写入文件、数据库等存储，进程结束后数据仍然存在。重要数据应在产生或修改后及时保存；`applicationDidEnterBackground(_:)` 适合再补存一次尚未保存的状态。例如，编辑器每隔几秒把草稿写入文件，应用被挂起后又被终止；用户重新打开应用时，应用读出最近一次保存的草稿，之后的修改会丢失。存储方式见 [iOS 数据持久化](../network-and-storage/ios-data-persistence.md)。

### Scene 生命周期与应用生命周期

Scene 是应用界面的一个实例，有自己的前台、后台和激活状态。例如在 iPad 上把同一个应用打开成两个窗口，就有两个 Scene。`UIWindowScene` 是管理窗口的 Scene 类。SceneDelegate 是开发者编写的场景代理类，遵循 `UIWindowSceneDelegate` 协议，接收某一个 Scene 的状态回调。

Scene 生命周期从 iOS 13 开始提供。应用启用 Scene 生命周期后，UIKit 把下面四种界面状态变化通知给对应 Scene 的 SceneDelegate，不再调用 AppDelegate 中对应的方法：

| AppDelegate 回调 | SceneDelegate 对应回调 |
|-|-|
| `applicationWillResignActive(_:)` | `sceneWillResignActive(_:)` |
| `applicationDidEnterBackground(_:)` | `sceneDidEnterBackground(_:)` |
| `applicationWillEnterForeground(_:)` | `sceneWillEnterForeground(_:)` |
| `applicationDidBecomeActive(_:)` | `sceneDidBecomeActive(_:)` |

AppDelegate 仍负责进程级的工作，例如启动初始化；SceneDelegate 负责某一个 Scene 的界面状态。一个 Scene 进入后台时，同一应用的另一个 Scene 可能仍在前台。

新创建的 Scene 第一次进入前台时，UIKit 也会调用 `sceneWillEnterForeground(_:)`；传统生命周期中，应用首次启动不调用 `applicationWillEnterForeground(_:)`。所以把“从后台回来刷新数据”的代码移到 `sceneWillEnterForeground(_:)` 后，Scene 首次显示时也会执行一次这段代码。

## AppDelegate 生命周期下，从启动到首帧显示

适用条件：应用使用传统 AppDelegate 生命周期，用代码创建界面，从未运行状态启动到前台。首帧是应用启动后，用户在屏幕上看到的第一个应用界面。从进程创建到首帧显示，按顺序分为六步：

1. 系统创建进程并加载代码。dyld 加载可执行文件和依赖的动态库，再执行运行时初始化等代码。这一步发生在 `main` 函数之前，见 [main 之前的启动工作与 dyld](#main-之前的启动工作与-dyld)。
2. 程序进入 `UIApplicationMain`。`UIApplicationMain` 创建 UIApplication 和 AppDelegate，把 AppDelegate 设为 UIApplication 的代理，并启动主线程 RunLoop。
3. UIKit 依次调用 `application(_:willFinishLaunchingWithOptions:)` 和 `application(_:didFinishLaunchingWithOptions:)`。应用在这两个方法里完成启动必需的初始化，并检查启动参数，例如应用是否由某个 URL 打开。
4. AppDelegate 在 `application(_:didFinishLaunchingWithOptions:)` 中创建 UIWindow，设置 `rootViewController`，再调用 `makeKeyAndVisible()`。`rootViewController` 是窗口的根视图控制器，窗口显示的主要内容由根视图控制器管理。使用 Storyboard 作为主界面时，UIKit 根据工程配置的主 Storyboard 完成这一步。
5. UIKit 加载根视图控制器的视图：调用 `loadView()` 创建视图，再调用 `viewDidLoad()`，然后执行布局和绘制。
6. 应用把准备好的界面提交给系统，系统完成渲染与合成后，屏幕显示首帧。

首帧在第 6 步才显示。`application(_:didFinishLaunchingWithOptions:)` 和 `makeKeyAndVisible()` 返回时，界面还在准备中；`applicationDidBecomeActive(_:)` 只表示应用可以接收交互。首帧显示时，网络请求和业务数据可能还在加载。

### main 之前的启动工作与 dyld

`main` 是程序的入口函数。pre-main 指进入 `main` 之前的启动阶段。Swift 的 UIKit 应用用 `@main` 标记 AppDelegate，编译器据此生成入口代码，所以源码中通常看不到 `main` 函数。

dyld 是 Apple 平台的动态链接器，负责在程序运行时把动态库装入进程。动态库是单独存放、程序运行时才装入的代码文件。pre-main 阶段按以下顺序执行：

1. 系统创建进程，把主可执行文件映射到内存。
2. dyld 读取可执行文件中记录的依赖列表，加载所需的动态库，再加载这些动态库自己依赖的库。
3. dyld 修正地址并绑定符号。符号是函数、全局变量等在代码中的名字。符号绑定是把代码对某个符号的引用，改成这个符号在内存中的实际地址。例如，应用代码调用 UIKit 中的函数，加载前只记录了函数名，绑定后才知道函数在内存中的位置。
4. 运行时完成初始化，并执行启动阶段的初始化代码。Objective-C 运行时是在程序运行期间注册类、管理方法和发送消息的系统库，初始化时调用类和分类的 `+load` 方法；C++ 全局对象执行构造函数；标记为初始化函数（例如 `__attribute__((constructor))`）的函数被执行。

以上工作都发生在 `main` 之前，都计入启动时间。函数内的 C++ 静态局部对象在第一次执行到声明时才构造，构造时机见 [C++ 静态存储期对象与初始化](../language/cpp-static-storage-and-initialization.md)。静态库（`.a` 文件）的代码在构建时已经复制进可执行文件，所以运行时 dyld 只加载动态库，链接与装载的区别见 [编译、链接、嵌入与装载](../engineering/xcode-and-testing.md#编译链接嵌入与装载)。

### LaunchScreen 与自定义启动页

`LaunchScreen` 是应用界面准备好之前，系统先显示的启动画面。用 Launch Screen Storyboard 配置的 `LaunchScreen` 是一份静态布局，由系统直接显示，所以只能展示固定的图片、文字和布局。

自定义启动页（例如开屏广告页）是应用自己创建的普通视图控制器，可以运行代码、加载素材、响应点击。窗口先显示自定义启动页时，应用首帧就是启动页，首页要等启动页关闭后才第一次显示。

自定义启动页影响首帧的关键有两点：

- 素材异步加载，并设置等待上限。等待期间，窗口先显示可以立即绘制的占位界面。主线程同步等待素材时会被阻塞，首帧随之推迟；素材加载失败或超过上限时，应用直接显示首页。
- 倒计时在 `viewDidAppear(_:)` 中开始。`viewDidLoad()` 执行时页面还没出现在屏幕上，回调顺序见 [控制器初始化、视图加载与页面展示](uikit-layout-and-mvvm.md#控制器初始化视图加载与页面展示)。

## 启动耗时的测量与优化

启动耗时是从选定的起点到选定的终点之间经过的时间。优化启动耗时分三步：先确定测量的起点和终点，再找出用户等待期间耗时最多的工作，最后减少或推迟这些工作。

### 进程创建时间与启动计时起点

进程创建时间是系统记录的、进程开始存在的时刻，比进入 `main()` 更早。只从 `main()` 开始计时，会漏掉 pre-main 阶段，所以启动计时常用进程创建时间作为起点。

读取进程创建时间要用 `sysctl`。`sysctl` 是 Darwin 系统提供的 C 函数，调用方传入查询路径和一块缓冲区，系统把查询结果写入缓冲区。下面的代码用当前进程的编号（PID，由 `getpid()` 返回）查询进程信息，再从 `kinfo_proc` 结构中读出启动时间：

```swift
import Darwin

func processStartTime() -> TimeInterval? {
    var mib: [Int32] = [CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()]
    var info = kinfo_proc()
    var size = MemoryLayout<kinfo_proc>.stride
    guard sysctl(&mib, UInt32(mib.count), &info, &size, nil, 0) == 0 else {
        return nil
    }
    let start = info.kp_proc.p_un.__p_starttime
    return TimeInterval(start.tv_sec) + TimeInterval(start.tv_usec) / 1_000_000
}
```

`processStartTime()` 返回时间戳，即从 1970 年 1 月 1 日 00:00:00 UTC 起算的秒数，记录的是进程创建的时刻。例如：10:00:00.000 系统创建进程，10:00:00.300 调用 `processStartTime()`，得到的仍是 10:00:00.000。耗时要用两个时间戳相减得到，所以终点也要用同一种时钟读取，例如 `Date().timeIntervalSince1970`。

系统预热是指系统在用户打开应用之前，提前创建进程并执行部分启动工作。发生预热时，进程创建时间可能比用户点击图标早很多，而预热期间用户还没开始等待，所以统计时要把预热阶段和用户实际等待的时间分开。

### 启动优化的三个方向

启动过程可以分为三段：进入 `main` 之前、`main` 到首屏显示、首屏显示之后。只有减少用户等待期间执行的工作，启动耗时才会下降。

**方向一：减少首屏不需要的初始化。** 初始化是为某个功能提前创建对象、读取配置、准备运行条件。首屏用不到的功能，把初始化推迟到功能第一次被使用时，例如分享 SDK 只在用户第一次点击“分享”时初始化。首屏因此更快显示，代价是第一次使用该功能时要多等这部分时间。`+load`、`+initialize` 的执行时机见 [Objective-C 类初始化与按需准备](../language/swift-objc-runtime.md#initialize-的触发时机与执行限制)。

**方向二：把耗时工作移出主线程。** 主线程负责显示界面和处理触摸。例如，启动时先读取一份很小的首页摘要并显示界面，再在后台线程读取和解析完整的历史记录，完成后回到主线程刷新对应区域。移到后台线程的工作只能是不操作 UIKit 的工作；主线程同步等待后台工作时，主线程仍被占用，启动耗时保持原样。

**方向三：减少首屏的视图和布局工作。** 只创建当前要显示的界面，按显示尺寸准备图片，避免重复创建约束。例如首页有多个标签页时，先创建默认标签页，其他标签页在用户切换过去时再创建。布局优化见 [UIKit 布局、列表与 MVVM](uikit-layout-and-mvvm.md)。

`main` 之前的耗时，可以通过减少 `+load` 等启动阶段的初始化代码、减少动态库数量和需要修正的地址来降低，见下面两节。

验证优化效果时，每次测量保持设备、启动类型（例如设备重启后第一次启动，或应用刚被关闭后再次启动）、数据和缓存条件一致，并同时检查首屏可以交互的时间，确认耗时没有只是转移到用户接下来的操作上。

### 动态库数量与启动开销

启动时每多加载一个自定义动态库，系统就要多做一份固定的工作：打开文件、建立内存映射、处理依赖关系、修正地址。内存映射是把文件内容对应到进程的地址空间，代码被访问到时，系统才从文件中读取对应的部分。

例如，把 10 个各约 1 MB 的动态库合并成 1 个约 10 MB 的动态库。代码总量基本不变，要加载和管理的库从 10 个变成 1 个。合并减少的只是每个库各自的固定开销，代码本身的加载和 `+load` 等初始化代码照样执行，所以启动耗时的下降比例小于库数量的下降比例。

系统框架（例如 UIKit）存放在共享缓存中。共享缓存是系统预先合并、预先链接好的系统库集合，所有进程共用，所以上面的开销规律只适用于自定义动态库。“自定义动态库控制在 6 个以内”是经验值，iOS 没有规定数量上限，启动开销随库的数量逐渐增加。

静态链接在构建时把库的代码复制进可执行文件，也能减少独立的动态库，取舍见 [静态链接与动态链接的取舍](../engineering/xcode-and-testing.md#静态链接与动态链接的取舍)。

### 删除无用代码与地址修正成本

删除无用代码能否加快启动，取决于删掉的内容是否减少了加载时需要修正的指针。dyld 加载时要修正两类指针：Rebase 按实际加载地址调整指向本模块内部的指针，Binding 把其他动态库中符号的实际地址写入指针，机制见 [Rebase 与 Binding：地址修正](../engineering/xcode-and-testing.md#rebase-与-binding地址修正)。

类的元数据是记录类有哪些方法、父类是谁等信息的数据，其中保存着指向方法实现、父类的指针。删除一个留在最终产物中的无用类后，元数据中的这些指针随之消失，修正工作就会减少。删除代码对修正工作的影响分三种情况：

- 只保存数值的全局变量（例如 `int count = 0;`）在加载地址变化时保持原值，无需修正。
- 修正次数取决于元数据中的指针数量，与方法数量没有固定比例，删除后减少多少要实际测量。
- 链接器在构建时只把被引用到的代码放进最终产物，其余代码被排除，这一步称为裁剪。已经被裁剪的代码在运行时本来就没有修正工作，删除这部分源码对启动耗时没有影响，裁剪条件见 [静态库的目标文件提取与代码裁剪](../engineering/xcode-and-testing.md#静态库的目标文件提取与代码裁剪)。

删除前要确认代码没有被运行时按名称查找（例如通过 `NSClassFromString` 查找类），没有被配置文件引用，也没有作为 `+load` 等初始化入口被执行。优化后分别测量代码体积、地址修正次数和首屏耗时。

## 为什么一个应用可以有多个 UIWindow

UIWindow 是 UIView 的子类，是应用进程中承载视图层级的根容器。同一个应用进程可以创建多个 UIWindow，额外的 UIWindow 叠在主 UIWindow 上面显示，所以用户看到的一屏画面可能由多个 UIWindow 叠加而成。

例如，主 UIWindow 显示首页；另一个背景透明的 UIWindow 只放一个悬浮的调试按钮。用户看到首页上浮着一个按钮，应用内部有两个 UIWindow。悬浮按钮也可以用主 UIWindow 中的普通 UIView 实现。

多数应用只需要一个主 UIWindow。首页和详情页之间的切换由视图控制器完成，例如 push 或 present。

## 关键窗口与 makeKeyAndVisible()

关键窗口（key window）是 UIKit 指定的、接收键盘输入等没有触摸坐标的事件的窗口。这类事件没有坐标，UIKit 无法按位置判断交给哪个窗口，所以交给关键窗口。关键窗口身份和可见状态由不同的操作设置：

| 操作 | 作用 |
|-|-|
| `makeKey()` | 请求窗口成为关键窗口，窗口保持原来的显示状态 |
| `isHidden = false` | 显示窗口，关键窗口保持不变 |
| `makeKeyAndVisible()` | 显示窗口，并把窗口设为关键窗口 |

`makeKeyAndVisible()` 还会把窗口放到 `windowLevel` 相同或更低的其他窗口前面。`windowLevel` 是窗口的层级值，层级值大的窗口显示在上面。

非关键窗口照样显示，也照样接收落在自己范围内的触摸，因为触摸事件按触摸位置分发。不使用 Scene 的传统应用只有一个关键窗口；使用多个 Scene 时，每个 Scene 各有一个关键窗口。键盘输入如何从关键窗口交给输入框，见 [屏幕键盘：触摸位置与文字接收对象](touch-events-and-gestures.md#屏幕键盘触摸位置与文字接收对象)。
