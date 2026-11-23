# iOS 应用启动、窗口与输入事件

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
  - [LaunchScreen、自定义启动页与异步加载](#launchscreen自定义启动页与异步加载)
- [启动耗时的测量与优化](#启动耗时的测量与优化)
  - [进程创建时间与启动计时起点](#进程创建时间与启动计时起点)
  - [启动优化的三个方向](#启动优化的三个方向)
  - [动态库数量与启动开销](#动态库数量与启动开销)
  - [删除无用代码与地址修正成本](#删除无用代码与地址修正成本)
- [为什么一个应用可以有多个 UIWindow](#为什么一个应用可以有多个-uiwindow)
- [关键窗口与 makeKeyAndVisible()](#关键窗口与-makekeyandvisible)
- [触摸事件分发：命中测试、响应链与手势](#触摸事件分发命中测试响应链与手势)
  - [UIView 的局部坐标与 bounds](#uiview-的局部坐标与-bounds)
  - [point(inside:with:) 与 hitTest(\_:with:)](#pointinsidewith-与-hittest_with)
  - [事件穿透与重叠视图的触摸目标](#事件穿透与重叠视图的触摸目标)
  - [扩大按钮的命中区域](#扩大按钮的命中区域)
  - [触摸回调与响应链转发](#触摸回调与响应链转发)
  - [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)
  - [同一按钮的点击与长按协调](#同一按钮的点击与长按协调)
  - [长按状态与取消后的恢复](#长按状态与取消后的恢复)
- [屏幕键盘：触摸位置与文字接收对象](#屏幕键盘触摸位置与文字接收对象)

## UIApplication、AppDelegate、UIWindow 与 UIView 的职责

UIKit 应用运行时，主要由四类对象分工协作：UIApplication 代表整个应用，AppDelegate 接收应用级通知，UIWindow 承载界面，UIView 显示内容并接收触摸。

UIApplication 是 UIKit 创建的应用对象，每个应用进程只有一个。业务代码通过 `UIApplication.shared` 访问 UIApplication。AppDelegate 是开发者编写的类，遵循 `UIApplicationDelegate` 协议。UIKit 在启动完成、进入后台等时机调用 AppDelegate 的方法，开发者在这些方法里写自己的初始化、保存等代码。

| 对象 | 由谁实现 | 主要职责 | 例子 |
|-|-|-|-|
| UIApplication | UIKit | 记录应用的运行状态，把事件分发给窗口，提供与系统交互的接口 | 调用 `UIApplication.shared.open(url)` 请求系统打开某个 URL |
| AppDelegate | 开发者 | 接收 UIKit 在应用级时机发出的回调，执行自定义代码 | 在 `application(_:didFinishLaunchingWithOptions:)` 中初始化服务、创建主窗口 |
| UIWindow | UIKit 提供类，开发者创建实例 | 作为视图层级的根容器，把事件交给窗口内的视图 | 设置 `rootViewController`，调用 `makeKeyAndVisible()` 显示窗口 |
| UIView | UIKit 提供类，开发者创建实例或子类 | 表示屏幕上的一块矩形区域，绘制内容并接收触摸 | `UIButton` 接收点击；`UITextField` 接收文字输入 |

UIApplication 和 AppDelegate 是两个不同的对象。UIApplication 由 UIKit 实现，负责管理应用和分发事件；AppDelegate 由开发者实现，负责在 UIKit 发出通知时执行开发者的代码。按钮点击等普通触摸事件由 UIApplication 交给窗口和视图，不经过 AppDelegate。

**例子：打开本应用的系统设置页。** 用户拒绝照片权限后，页面显示“去设置”按钮。用户点击按钮后，按钮的动作方法用 `UIApplication.openSettingsURLString` 生成 URL，再调用 `UIApplication.shared.open(url)`。UIApplication 把请求转给系统，系统打开本应用的设置页面。UIApplication 只负责发出请求，设置页面由系统显示。

### UIApplication 与主线程 RunLoop

RunLoop 是在一个线程上反复执行的循环：没有事件时，RunLoop 让线程休眠；触摸、定时器等事件到达时，RunLoop 唤醒线程，并调用对应的处理代码。主线程 RunLoop 是运行在主线程上的 RunLoop，UIKit 也把主线程 RunLoop 称为主事件循环。

启动时，`UIApplicationMain` 创建 UIApplication 和 AppDelegate，然后启动主线程 RunLoop。主线程 RunLoop 由 UIKit 启动，业务代码不需要调用 `RunLoop.main.run()`。

以点击按钮为例，RunLoop 和 UIKit 对象的分工如下：

1. 系统把触摸事件发给应用进程。
2. 主线程 RunLoop 被唤醒，开始处理这次触摸。RunLoop 决定“什么时候处理事件”。
3. UIApplication 把事件交给触摸所在的 UIWindow，UIWindow 再把事件交给按钮。UIApplication 和 UIWindow 决定“把事件交给谁”。
4. 按钮执行动作方法。

主线程同一时刻只执行一段代码，RunLoop 等当前代码执行完，才处理下一个事件。如果按钮的动作方法在主线程上执行了 2 秒，新的触摸和界面刷新都要等这 2 秒结束才执行，用户就会感到卡顿。运行模式和输入源见 [iOS 线程与任务调度](threads-and-scheduling.md#runloop-与运行模式)。

## 应用运行状态与生命周期回调

应用生命周期是应用进程从启动、在前台和后台之间切换，到最终终止的整个过程。生命周期回调是 UIKit 在应用状态变化时调用的代理方法。应用在回调中完成初始化、暂停、恢复、保存等工作。

本节先按传统 AppDelegate 生命周期说明。AppDelegate 中以 `application` 开头的回调方法，参数是当前的 UIApplication 对象。启用 Scene 生命周期后，界面状态的变化改为通知 SceneDelegate，区别见 [Scene 生命周期与应用生命周期](#scene-生命周期与应用生命周期)。

### 五种运行状态

| 状态 | 含义 | 容易误解的地方 |
|-|-|-|
| Not Running（未运行） | 应用进程还没启动，或已经终止 | 再次打开应用时，系统要重新创建进程 |
| Inactive（未激活） | 应用在前台，但暂时不接收触摸等用户交互 | 应用代码仍在执行，网络请求完成后的回调照常执行 |
| Active（已激活） | 应用在前台，并接收用户交互 | Active 只表示应用可以接收交互；数据加载和首帧显示各有自己的完成时机 |
| Background（后台） | 应用不在前台，但在系统允许的时间内还能执行代码 | 普通应用做完收尾工作后会被挂起；只有声明了音频播放、定位等后台能力的应用，才能让对应工作继续运行 |
| Suspended（挂起） | 进程还在内存中，但不执行任何代码 | 系统需要内存时可以直接终止挂起的进程，内存中未保存的数据随之丢失 |

前台包括 Inactive 和 Active 两种状态。应用回到前台时先处于 Inactive，之后才变为 Active。后台状态下代码还能运行，挂起后代码完全停止。音频、定位等后台能力需要在工程中配置，系统只允许对应类型的工作在后台持续运行。

这五种状态是为了说明生命周期而做的划分。代码能读到的 `UIApplication.shared.applicationState` 只有 `.active`、`.inactive`、`.background` 三个值。原因是应用处于 Not Running 或 Suspended 时，应用代码根本不执行，也就没有机会读取状态。

### AppDelegate 的主要回调

| 回调 | UIKit 调用时机 | 常见用途 |
|-|-|-|
| `application(_:didFinishLaunchingWithOptions:)` | 启动流程即将完成，应用即将开始运行 | 初始化启动必需的服务；用代码创建界面时，在这里创建主窗口 |
| `applicationWillResignActive(_:)` | 应用即将从 Active 变为 Inactive | 暂停依赖用户交互的工作，例如暂停游戏、停止计时器 |
| `applicationDidEnterBackground(_:)` | 应用已经进入 Background | 保存还没保存的数据，释放可以重新创建的资源，结束不需要继续的工作 |
| `applicationWillEnterForeground(_:)` | 应用即将从 Background 回到前台，先进入 Inactive | 检查后台期间发生的数据变化，按需刷新 |
| `applicationDidBecomeActive(_:)` | 应用已经从 Inactive 变为 Active | 恢复因失去激活而暂停的工作 |
| `applicationWillTerminate(_:)` | 应用即将终止，并且系统有机会通知应用 | 执行简短的清理。系统直接终止挂起进程时不调用这个方法，所以数据要在更早的时机保存 |

UIKit 只负责在对应时机调用这些方法。计时器是否暂停、数据是否刷新，取决于开发者在方法里写了什么代码。

`applicationWillResignActive(_:)` 被调用后，应用可能进入后台，也可能很快恢复激活。例如用户下拉控制中心又收起，应用会从 Inactive 直接回到 Active。

### 常见状态变化与调用顺序

下面三个流程都假设状态切换正常完成。如果期间发生系统中断、后台启动或进程被终止，回调顺序会不同。

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

进入后台后，应用可能被挂起；挂起后，系统也可能终止进程。如果系统在后台期间终止了进程，用户再次打开应用时走流程 1。

**流程 3：前台短暂中断。** 例如用户下拉控制中心后又收起，应用只进入 Inactive，没有进入 Background。

```text
状态：Active → Inactive → Active
回调：applicationWillResignActive(_:) → applicationDidBecomeActive(_:)
```

来电、系统弹窗出现时，应用可能只进入 Inactive，也可能进入 Background。UIKit 按实际发生的状态变化调用对应的回调。

### 进入前台与进入激活状态的区别

`applicationWillEnterForeground(_:)` 表示“应用即将从后台回到前台”。`applicationDidBecomeActive(_:)` 表示“应用已经可以接收用户交互”。两个回调不总是成对出现，对照上一节的三个流程：

| 流程 | `applicationWillEnterForeground(_:)` | `applicationDidBecomeActive(_:)` |
|-|-|-|
| 流程 1：从未运行直接启动 | 不调用 | 调用 |
| 流程 2：从后台返回前台 | 先调用 | 后调用 |
| 流程 3：短暂中断后恢复 | 不调用 | 调用 |

因此，代码放在哪个回调，取决于代码需要在什么条件下执行：

- “每次从后台回来都刷新数据”放在 `applicationWillEnterForeground(_:)`。这次刷新只在流程 2 中执行。
- “每次恢复交互时继续暂停的任务”放在 `applicationDidBecomeActive(_:)`。三个流程都会触发。

如果在 `applicationWillEnterForeground(_:)` 中发起异步网络请求，UIKit 照常继续切换状态，并调用 `applicationDidBecomeActive(_:)`，网络请求在此期间继续进行。所以 `applicationDidBecomeActive(_:)` 执行时，数据可能还没返回。应用可以先显示已有数据，请求完成后再更新界面。

### 挂起、终止与数据保存

挂起的应用不执行代码。系统可以直接终止挂起的进程。终止发生时应用没有代码在运行，所以 `applicationWillTerminate(_:)` 不会被调用，数据也来不及保存。用户在应用切换器中上滑关闭应用时，`applicationWillTerminate(_:)` 同样可能不被调用。

持久保存是把数据写入文件、数据库等存储，进程结束后数据仍然存在。只保存在内存中的数据，会随进程终止一起丢失。

因此，重要数据应在产生或修改后及时保存。`applicationDidEnterBackground(_:)` 适合再补存一次尚未保存的状态，作为及时保存之外的补充。具体存储方式见 [iOS 数据持久化](../network-and-storage/ios-data-persistence.md)。

例如，用户在编辑器中写草稿，应用每隔几秒把草稿写入文件。之后应用进入后台、被挂起，又被系统终止。用户重新打开应用时，应用从文件读出最近一次保存的草稿；最近一次保存之后、还没写入文件的修改会丢失。

### Scene 生命周期与应用生命周期

Scene 是应用界面的一个实例，有自己的前台、后台和激活状态。例如在 iPad 上把同一个应用打开成两个窗口，就有两个 Scene。`UIWindowScene` 是管理窗口的 Scene 类。SceneDelegate 是开发者编写的场景代理类，遵循 `UIWindowSceneDelegate` 协议（继承自 `UISceneDelegate`），接收某一个 Scene 的状态回调。一个应用进程可以同时有多个 Scene。

Scene 生命周期从 iOS 13 开始提供。应用启用 Scene 生命周期后，UIKit 把下面四种界面状态变化通知给对应 Scene 的 SceneDelegate，不再调用 AppDelegate 中对应的方法。SceneDelegate 回调的参数是发生状态变化的那个 Scene。

| AppDelegate 回调 | SceneDelegate 对应回调 |
|-|-|
| `applicationWillResignActive(_:)` | `sceneWillResignActive(_:)` |
| `applicationDidEnterBackground(_:)` | `sceneDidEnterBackground(_:)` |
| `applicationWillEnterForeground(_:)` | `sceneWillEnterForeground(_:)` |
| `applicationDidBecomeActive(_:)` | `sceneDidBecomeActive(_:)` |

AppDelegate 仍负责进程级的工作，例如在 `application(_:didFinishLaunchingWithOptions:)` 中完成启动初始化；SceneDelegate 负责某一个 Scene 的界面状态。一个 Scene 进入后台时，同一应用的另一个 Scene 可能仍在前台。

**首次进入前台时的区别。** 新创建的 Scene 第一次进入前台时，UIKit 也会调用 `sceneWillEnterForeground(_:)`。而在传统生命周期中，应用首次启动时不调用 `applicationWillEnterForeground(_:)`。所以把“从后台回来刷新数据”的代码移到 `sceneWillEnterForeground(_:)` 后，Scene 首次显示时也会执行一次这段代码。

应用、Scene 和 UIViewController 各有自己的生命周期。页面出现或消失（例如 `viewDidAppear(_:)` 被调用）由 push、present 等页面切换触发；应用进入前台或后台由用户切换应用触发。页面层面的回调见 [UIKit 布局、列表与 MVVM](uikit-layout-and-mvvm.md#页面生命周期与展示方式)。

## AppDelegate 生命周期下，从启动到首帧显示

适用条件：应用使用传统 AppDelegate 生命周期，用代码创建界面，从未运行状态启动到前台。系统预热、状态恢复等情况另有流程。应用从后台回到前台时，进程已经存在，UIKit 只调用生命周期回调，见 [常见状态变化与调用顺序](#常见状态变化与调用顺序)。

首帧是应用启动后，用户在屏幕上看到的第一个应用界面。从进程创建到首帧显示，按顺序分为六步：

1. 系统创建进程并加载代码。系统创建应用进程，dyld 加载可执行文件和依赖的动态库，再执行运行时初始化等代码。这一步发生在 `main` 函数之前，详见 [main 之前的启动工作与 dyld](#main-之前的启动工作与-dyld)。
2. 程序进入 `UIApplicationMain`。`UIApplicationMain` 创建 UIApplication 和 AppDelegate，把 AppDelegate 设为 UIApplication 的代理，并启动主线程 RunLoop。
3. UIKit 调用启动回调。UIKit 依次调用 `application(_:willFinishLaunchingWithOptions:)` 和 `application(_:didFinishLaunchingWithOptions:)`。应用在这两个方法里完成启动必需的初始化，并检查启动参数，例如应用是否由某个 URL 打开。
4. AppDelegate 创建窗口。用代码创建界面时，AppDelegate 在 `application(_:didFinishLaunchingWithOptions:)` 中创建 UIWindow，设置 `rootViewController`，再调用 `makeKeyAndVisible()`。`rootViewController` 是窗口的根视图控制器，窗口显示的主要内容由根视图控制器管理。
5. UIKit 加载根视图控制器的视图。窗口要显示时，UIKit 调用根视图控制器的 `loadView()` 创建视图，再调用 `viewDidLoad()`，然后执行布局和绘制。
6. 系统显示首帧。应用把准备好的界面提交给系统，系统完成渲染与合成后，屏幕显示首帧。

首帧在第 6 步才显示，所以下面两个时间点都早于首帧：

- `application(_:didFinishLaunchingWithOptions:)` 返回时。
- `makeKeyAndVisible()` 返回时。`makeKeyAndVisible()` 只把窗口设为可见的关键窗口；界面还要完成布局和绘制，再由系统渲染，才会出现在屏幕上。

`applicationDidBecomeActive(_:)` 只表示应用可以接收交互，判断首帧要看第 6 步。首帧显示时，网络请求和业务数据可能还在加载。

使用 Storyboard 作为主界面时，第 4 步由 UIKit 完成：UIKit 根据工程配置的主 Storyboard 创建窗口和根视图控制器。

### main 之前的启动工作与 dyld

`main` 是程序的入口函数。pre-main 指进入 `main` 之前的启动阶段。Swift 的 UIKit 应用用 `@main` 标记 AppDelegate，编译器据此生成入口代码，所以源码中通常看不到 `main` 函数。

dyld 是 Apple 平台的动态链接器，负责在程序运行时把动态库装入进程。动态库是单独存放的代码文件，程序运行时才被装入。pre-main 阶段按以下顺序执行：

1. 系统创建进程，把主可执行文件映射到内存。
2. dyld 读取可执行文件中记录的依赖列表，加载所需的动态库，再加载这些动态库自己依赖的库。
3. dyld 修正地址并绑定符号。符号是函数、全局变量等在代码中的名字。符号绑定是把代码对某个符号的引用，改成这个符号在内存中的实际地址。例如，应用代码调用 UIKit 中的函数，加载前只记录了函数名，绑定后才知道函数在内存中的位置。
4. 运行时完成初始化，并执行启动阶段的初始化代码。例如，Objective-C 运行时调用类和分类的 `+load` 方法；C++ 全局对象执行构造函数；标记为初始化函数（例如 `__attribute__((constructor))`）的函数被执行。

以上工作都发生在 `main` 之前，所以都计入启动时间。

需要区分两种情况：

- C++ 静态对象的构造时机分两种。全局对象在 `main` 之前构造；函数内的静态局部对象，如果需要运行代码才能初始化（例如函数里的 `static Logger logger;`，`Logger` 的构造函数需要执行代码），会在程序第一次执行到这行声明时才构造。存储期与构造时机见 [C++ 静态存储期对象与初始化](../language/cpp-static-storage-and-initialization.md)。
- 静态库只在构建时使用。静态库（`.a` 文件）的代码在构建时已经复制进可执行文件，所以运行时 dyld 只加载动态库。链接、嵌入和装载的区别见 [编译、链接、嵌入与装载](../engineering/xcode-and-testing.md#编译链接嵌入与装载)。

### LaunchScreen、自定义启动页与异步加载

`LaunchScreen` 是应用界面准备好之前，系统先显示的启动画面。用 Launch Screen Storyboard 配置的 `LaunchScreen` 是一份静态布局，由系统直接显示，应用代码无法控制 `LaunchScreen` 的内容。所以 `LaunchScreen` 只能展示固定的图片、文字和布局；网络请求、倒计时和按钮响应要放到自定义启动页中。

自定义启动页（例如开屏广告页）是应用自己创建的普通视图控制器，可以运行代码、加载素材、响应点击。这里要区分两个首帧：应用首帧是应用显示的第一个画面；首页首帧是首页第一次显示的画面。如果窗口先显示自定义启动页，应用首帧就是启动页，首页首帧要等启动页关闭后才出现。

自定义启动页涉及三个对象：启动页控制器负责显示素材和倒计时；加载组件负责获取素材；管理者负责决定显示启动页还是首页，例如 AppDelegate 或一个专门的协调对象。按执行顺序分为四步：

1. 管理者提供显示位置。首次启动时，启动页要在首页之前显示，管理者就把启动页的容器控制器设为窗口的 `rootViewController`，启动页结束后再换成首页。启用 Scene 生命周期时，窗口由对应的 Scene 管理。
2. 加载组件异步获取素材。素材请求结束后，加载组件调用管理者传入的完成回调，并传入素材或错误。等待期间，窗口要先显示一个可以立即绘制的界面，例如占位图。素材必须异步加载：主线程同步等待素材时会被阻塞，首帧随之推迟。
3. 启动页显示素材，并开始倒计时。启动页在 `viewDidAppear(_:)` 中开始倒计时，因为 `viewDidLoad()` 执行时页面还没出现在屏幕上。回调顺序见 [控制器初始化、视图加载与页面展示](uikit-layout-and-mvvm.md#控制器初始化视图加载与页面展示)。
4. 启动页关闭。启动页停止倒计时，取消还没完成的任务，再调用管理者传入的关闭回调。管理者收到关闭回调后显示首页。

实现时注意三点：

- 设置等待上限。素材加载失败或超过等待上限时，管理者直接显示首页，首页最多等待这个上限时长。
- 区分两个回调。“素材加载完成”由加载组件调用完成回调通知管理者；“启动页已关闭”由启动页调用关闭回调通知管理者。两件事各自发生，例如素材加载超时后启动页已经关闭，之后素材才返回。
- 丢弃过期结果。被取消的任务有时仍会返回结果。完成回调执行时，要先检查启动页是否已经关闭；启动页已关闭时直接丢弃结果，不刷新已关闭的页面。

应用已经有可见页面时（例如从后台回到前台），也可以用 `present(_:animated:completion:)` 弹出启动页。前提有两个：发起 `present` 的控制器正在屏幕上显示；当前没有其他正在进行的转场。启动回调执行时，窗口和控制器可能还没显示，所以要等发起 `present` 的控制器出现在屏幕上（例如在 `viewDidAppear(_:)` 中）再调用。

如果启动页每次进程启动只显示一次，显示记录要存放在与进程同生命周期的对象中，例如管理者的一个属性。`applicationDidBecomeActive(_:)`、`sceneDidBecomeActive(_:)` 和 `viewDidAppear(_:)` 在一次进程运行中会被调用多次，所以要用显示记录判断：记录为“未显示”时才显示启动页，显示后把记录改为“已显示”。进程终止后再打开，算新的一次启动；前后台切换不算，进程还是原来的进程。

倒计时的剩余时间要用“截止时刻 − 当前时刻”计算，截止时刻等于开始时刻加展示时长。不要每次定时器回调减 1 秒，因为定时器回调可能延迟，逐次减 1 秒会让显示值和实际时间越差越多。时间要取自单调时钟。单调时钟是只向前走、不受用户修改系统时间影响的时钟，例如 Swift 的 `ContinuousClock`。例如展示时长为 5 秒，开始时刻为 100.0 秒，截止时刻就是 105.0 秒；某次定时器回调在 102.3 秒执行，剩余时间是 105.0 − 102.3 = 2.7 秒。任务的结果类型和取消规则见 [任务结果类型与 Void、Never](../language/swift-basics.md#任务结果类型与-voidnever) 与 [上下文继承与取消](../language/swift-basics.md#上下文继承与取消)。

## 启动耗时的测量与优化

启动耗时是从选定的起点到选定的终点之间经过的时间。优化启动耗时分三步：先确定测量的起点和终点，再找出用户等待期间耗时最多的工作，最后减少或推迟这些工作。

### 进程创建时间与启动计时起点

进程创建时间是系统记录的、进程开始存在的时刻，比进入 `main()` 更早。只从 `main()` 开始计时，会漏掉 pre-main 阶段，所以启动计时常用进程创建时间作为起点。

读取进程创建时间要用 `sysctl`。`sysctl` 是 Darwin 系统提供的 C 函数，用来查询系统和进程的信息。PID 是系统分配给进程的编号。读取步骤如下：

1. 调用 `getpid()` 获取当前进程的 PID。
2. 准备查询路径 `[CTL_KERN, KERN_PROC, KERN_PROC_PID, pid]`，含义是“查询内核中、指定 PID 的进程信息”。
3. 准备一个 `kinfo_proc` 结构体作为缓冲区，调用 `sysctl`。系统把进程信息写入缓冲区。
4. 确认调用成功后，从 `kp_proc.p_un.__p_starttime` 读取启动时间。启动时间分为秒 `tv_sec` 和微秒 `tv_usec` 两部分，合并后为 `tv_sec + tv_usec / 1_000_000.0` 秒。

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

`processStartTime()` 返回的是时间戳。时间戳是从固定时间原点（1970 年 1 月 1 日 00:00:00 UTC）算起的时刻；耗时要用两个时间戳相减得到。时间戳记录的是进程创建的时刻，与何时查询无关。例如：10:00:00.000 系统创建进程，10:00:00.200 进入 `main()`，10:00:00.300 调用 `processStartTime()`，得到的仍是 10:00:00.000。

用进程创建时间作起点时，还要注意两点：

- 进程创建时间可能早于用户点击图标的时间。系统预热是指系统在用户打开应用之前，提前创建进程并执行部分启动工作。发生预热时，进程创建时间可能比用户点击早很多，而预热期间用户还没开始等待，所以统计时要把预热阶段和用户实际等待的时间分开。
- 起点和终点必须来自同一种时钟、使用相同的单位。`processStartTime()` 返回的是从 1970 年起算的秒数，终点也要用同一种时钟读取，例如 `Date().timeIntervalSince1970`，两者相减才是耗时。

### 启动优化的三个方向

启动优化先确定测量的起点和终点，再把启动过程分为三段：进入 `main` 之前、`main` 到首屏显示、首屏显示之后。只有减少用户等待期间执行的工作，启动耗时才会下降。

**方向一：减少首屏不需要的初始化。** 初始化是为某个功能提前创建对象、读取配置、准备运行条件。如果首屏用不到某个功能，就把这个功能的初始化推迟到功能第一次被使用时。这样首屏更快显示，代价是第一次使用该功能时要多等这部分时间。例如分享 SDK 只在用户第一次点击“分享”时初始化。`+load`、`+initialize` 和功能入口的执行时机、同步等待及注册顺序见 [Objective-C 类初始化与按需准备](../language/swift-objc-runtime.md#initialize-的触发时机与执行限制)。

**方向二：把耗时工作移出主线程。** 主线程负责显示界面和处理触摸。在主线程上同步扫描文件、解析大量数据或解码大图，都会推迟界面显示。例如，启动时先读取一份很小的首页摘要并显示界面，再在后台线程读取和解析完整的历史记录，完成后回到主线程刷新对应区域。这种做法有两个限制：

- 只有不操作 UIKit 的工作才能移到后台线程，UIKit 对象要在主线程上使用。
- 工作移到后台线程后，主线程继续执行其他工作，后台工作完成后再回到主线程更新界面。如果主线程同步等待后台工作，主线程仍被占用，启动耗时保持原样。

**方向三：减少首屏的视图和布局工作。** 只创建当前要显示的界面；按显示尺寸准备图片；避免重复创建约束。例如首页有多个标签页时，先创建默认标签页的内容，其他标签页在用户切换过去时再创建；列表优先配置当前屏幕上显示的项目。布局优化见 [UIKit 布局、列表与 MVVM](uikit-layout-and-mvvm.md)。

`main` 之前的耗时来自上文 [main 之前的启动工作与 dyld](#main-之前的启动工作与-dyld) 中的各项工作。已经确认耗时发生在 `main` 之前时，可以减少 `+load` 等启动阶段的初始化代码，去掉不需要的依赖，并减少动态库数量和需要修正的地址，见下面两节。

验证优化效果时注意三点：

- 根据测量出的各阶段耗时判断瓶颈。
- 每次测量保持设备、启动类型、数据和缓存条件一致。启动类型例如设备重启后第一次启动应用，或应用刚被关闭后再次启动。
- 同时检查首屏可以交互的时间，确认耗时没有只是转移到用户接下来的操作上。

### 动态库数量与启动开销

启动时每多加载一个自定义动态库，系统就要多做一份固定的工作：打开文件、建立内存映射、处理依赖关系、修正地址。内存映射是把文件内容对应到进程的地址空间，代码被访问到时，系统才从文件中读取对应的部分。系统会缓存部分工作结果，但剩下的开销仍会随动态库的数量、大小和实际访问的数据增加。

例如，把 10 个各约 1 MB 的动态库合并成 1 个约 10 MB 的动态库。代码总量基本不变，但要加载和管理的库从 10 个变成 1 个；原来跨库的部分符号引用，在构建时就能确定地址。合并减少的只是每个库各自的固定开销，代码本身的加载和初始化工作保持不变，所以启动耗时的下降比例小于库数量的下降比例。

合并动态库有三个局限：

- `+load` 等初始化代码合并后照样执行，执行时间保持不变。
- 系统框架（例如 UIKit）有共享缓存等专门优化。共享缓存是系统预先合并、预先链接好的系统库集合，所有进程共用。所以上面的开销规律只适用于自定义动态库。
- “自定义动态库控制在 6 个以内”是经验值，iOS 没有规定数量上限。启动开销随库的数量逐渐增加，在 6 个处没有突变。

静态链接也能减少独立的动态库。静态链接在构建时把库的代码复制进可执行文件，运行时不再单独加载这个库。构建速度、代码共享和可合并库的取舍见 [静态链接与动态链接的取舍](../engineering/xcode-and-testing.md#静态链接与动态链接的取舍)。

### 删除无用代码与地址修正成本

删除无用代码能否加快启动，取决于删掉的内容是否减少了加载时需要修正的指针。

dyld 加载可执行文件时要修正两类指针。Rebase 修正指向本模块内部的指针：系统每次把可执行文件加载到随机的地址，模块内部的指针要按实际加载地址调整。Binding 修正指向其他动态库中符号的指针：dyld 查到符号的实际地址后，把地址写入指针。两种操作的机制与示例见 [Rebase 与 Binding：地址修正](../engineering/xcode-and-testing.md#rebase-与-binding地址修正)。

类的元数据是描述类的数据，记录类有哪些方法、父类是谁等信息。元数据中保存着指向方法实现、父类等的指针，这些指针在加载时都需要修正。所以删除一个确实留在最终产物中的无用类后，元数据中的指针随之消失，Rebase 和 Binding 的工作就会减少。

删除代码对修正工作的影响，取决于删掉的内容：

- 只保存数值的全局变量（例如 `int count = 0;`）在加载地址变化时保持原值，无需修正。只有保存地址的指针才需要修正。
- 修正次数取决于元数据中的指针数量，与方法数量没有固定比例。删除 10 个方法后减少了多少次修正，要实际测量。
- 已经被链接器裁剪掉的代码没有进入最终产物，运行时本来就没有对应的修正工作，再删除这部分源码对启动耗时没有影响。裁剪条件见 [静态库的目标文件提取与代码裁剪](../engineering/xcode-and-testing.md#静态库的目标文件提取与代码裁剪)。

删除前要确认三点：代码没有被运行时按名称查找（例如通过 `NSClassFromString` 查找类）；代码没有被配置文件引用；代码没有作为 `+load` 等初始化入口被执行。优化后分别测量代码体积、地址修正次数和首屏耗时三项指标。

## 为什么一个应用可以有多个 UIWindow

UIWindow 是 UIView 的子类，作为视图层级的根容器承载界面，并把事件交给窗口内的视图。UIWindow 是应用进程中的一个视图对象，与桌面系统中带标题栏、可以拖动的窗口是不同的概念。同一个应用进程可以创建多个 UIWindow。

多数应用只需要一个主 UIWindow。额外的 UIWindow 会叠在主 UIWindow 上面显示，所以用户看到的一屏画面可能由多个 UIWindow 叠加而成。

例如，主 UIWindow 显示首页；另一个背景透明的 UIWindow 只放一个悬浮的调试按钮。用户看到的是首页上浮着一个按钮，应用内部其实有两个 UIWindow。这只是一种做法，悬浮按钮也可以用主 UIWindow 中的普通 UIView 实现。

首页和详情页之间的切换由视图控制器完成，例如 push 或 present，不需要为每个页面新建 UIWindow。

## 关键窗口与 makeKeyAndVisible()

关键窗口（key window）是 UIKit 指定的、接收键盘输入等没有触摸坐标的事件的窗口。这类事件没有坐标，UIKit 无法按位置判断交给哪个窗口，所以交给关键窗口。关键窗口身份和可见状态由不同的操作设置：

| 操作 | 作用 |
|-|-|
| `makeKey()` | 请求窗口成为关键窗口，窗口保持原来的显示状态 |
| `isHidden = false` | 显示窗口，关键窗口保持不变 |
| `makeKeyAndVisible()` | 显示窗口，并把窗口设为关键窗口 |

`makeKeyAndVisible()` 还会把窗口放到 `windowLevel` 相同或更低的其他窗口前面。`windowLevel` 是窗口的层级值，层级值大的窗口显示在上面。窗口叠放的前后顺序由 `windowLevel` 和显示的先后决定，与关键窗口身份各自独立。

非关键窗口照样显示，也照样接收落在自己范围内的触摸，因为触摸事件按触摸位置分发。

“关键窗口只有一个”是针对一个 Scene 而言的。不使用 Scene 的传统应用只有一个关键窗口；使用多个 Scene 时，每个 Scene 各有一个关键窗口。

## 触摸事件分发：命中测试、响应链与手势

事件分发是把一个事件交给合适的对象处理。UIApplication 和 UIWindow 都参与分发，因为两者负责的层级不同：UIApplication 在应用内选择窗口，UIWindow 在窗口内选择视图。

触摸事件的路径可以简化为：

```text
系统 → UIApplication → 触摸所在的 UIWindow → 窗口内的目标 UIView
```

这条路径省略了手势识别器，手势识别器的参与方式见 [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)。

找到目标视图，以及找到之后继续传递事件，分别由两套机制负责：

- 命中测试：UIKit 根据触摸位置和视图层级，找出触摸落在哪个视图上。命中测试找到的视图是第一个接收这次触摸的视图。
- 响应链：由每个响应对象的 `next` 属性串起来的一条链。目标视图不继续处理的事件，可以沿响应链传给下一个对象。

下面先说明命中测试，包括局部坐标、`hitTest(_:with:)`、事件穿透和扩大命中区域；再说明响应链和手势识别。

### UIView 的局部坐标与 bounds

命中测试使用的坐标都是视图自己的局部坐标。局部坐标系是以视图自身为参照的坐标系。默认情况下，`bounds.origin` 为 (0, 0)，视图可见区域的左上角就是局部坐标 (0, 0)，x 轴向右，y 轴向下，单位是点（pt）。

例如，一个按钮的 `bounds` 是 `CGRect(x: 0, y: 0, width: 50, height: 50)`。在按钮的局部坐标系中，中心点是 (25, 25)，右下角是 (50, 50)。按钮放在父视图的任何位置，这些局部坐标都不变。

`bounds` 和 `frame` 的区别如下：

- `bounds` 是视图在自身坐标系中的矩形。
- `frame` 是视图在父视图坐标系中的矩形。例如按钮放在父视图的 (100, 200) 处，按钮的 `frame.origin` 是 (100, 200)，`bounds.origin` 仍是 (0, 0)。

修改 `bounds.origin` 后，可见区域左上角的局部坐标就不再是 (0, 0)，`UIScrollView` 滚动时改变的就是 `bounds.origin`。`UIView.center` 使用父视图坐标系。例如按钮的 `frame` 是 `CGRect(x: 100, y: 200, width: 50, height: 50)` 时，`center` 是 (125, 225)，局部坐标中的中心点是 (25, 25)。

### point(inside:with:) 与 hitTest(\_:with:)

两个方法的分工不同：

- `point(inside:with:)` 判断一个局部坐标点是否在当前视图的命中区域内，返回 `Bool`。命中区域是能让当前视图成为触摸目标的范围，默认就是 `bounds`。
- `hitTest(_:with:)` 在当前视图及其所有子视图中找出触摸目标，返回 `UIView?`；找不到目标时返回 `nil`。

UIWindow 收到触摸后，从自己开始调用 `hitTest(_:with:)`。UIView 默认的 `hitTest(_:with:)` 按以下顺序执行：

1. 当前视图被隐藏（`isHidden` 为 true）、关闭了用户交互（`isUserInteractionEnabled` 为 false），或 `alpha` 小于 0.01 时，直接返回 `nil`。
2. 调用当前视图的 `point(inside:with:)`。结果为 false 时，直接返回 `nil`，当前视图的所有子视图都不再检查。
3. 从最上层的子视图开始，依次检查子视图：先把触摸点转换成子视图的局部坐标，再调用子视图的 `hitTest(_:with:)`。最上层的子视图是 `subviews` 数组中的最后一个，所以检查顺序是数组的倒序。
4. 某个子视图返回非 `nil` 的目标时，当前视图返回这个目标，不再检查其余子视图。所有子视图都返回 `nil` 时，当前视图返回自己。

例如，窗口中有一个容器视图，容器视图中有一个按钮，用户点击按钮。窗口检查容器视图，容器视图的 `point(inside:with:)` 返回 true，容器视图接着检查按钮；按钮的 `point(inside:with:)` 返回 true，按钮没有子视图，于是按钮返回自己。容器视图和窗口依次把按钮作为结果返回，按钮成为这次触摸的目标。

两个方法接收的 `point` 都是当前视图局部坐标系中的坐标。所以重写命中区域时，要把 `point` 与同样使用局部坐标的 `bounds` 比较。

命中测试在触摸开始时确定目标视图。之后这次触摸从开始到结束都交给这个目标视图；手指移动到其他视图上，触摸仍交给原来的目标视图。

**让容器接管子视图的触摸。** 重写容器的 `hitTest(_:with:)`，把原本返回子视图的结果改为返回容器自己：

```swift
final class InterceptingContainer: UIView {
    var interceptsTouches = false

    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        guard let target = super.hitTest(point, with: event) else {
            return nil
        }
        return interceptsTouches ? self : target
    }
}
```

`InterceptingContainer` 的执行过程如下：

1. `super.hitTest(point, with: event)` 按默认规则查找目标。先调用 `super`，是为了保留隐藏、交互关闭、命中区域等默认判断。
2. `super` 返回 `nil` 时，说明触摸不在容器范围内或容器不可交互，`InterceptingContainer` 也返回 `nil`。
3. `super` 返回非 `nil`，且 `interceptsTouches` 为 true 时，容器返回 `self`。这次触摸的目标变成容器，子控件收不到触摸。

容器成为目标后，要自己通过触摸方法或手势识别器执行动作。另外，改变 `hitTest(_:with:)` 的返回值挡不住祖先视图上的手势识别器，原因见 [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)。

### 事件穿透与重叠视图的触摸目标

事件穿透是让上层视图在某些位置不成为触摸目标，于是命中测试继续检查下层的视图。穿透发生在命中测试选择目标的阶段：命中测试直接选中下层视图，上层视图从头到尾收不到这次触摸。

例如，父视图中有按钮 B，视图 A 盖在 B 上面，A 在 `subviews` 中排在 B 后面。父视图做命中测试时先检查 A：

- A 的 `hitTest(_:with:)` 返回 A 或 A 的子视图时，目标已经确定，父视图停止检查，B 收不到触摸。
- 只有 A 的 `hitTest(_:with:)` 返回 `nil` 时，父视图才接着检查 B。

命中测试只看 `hitTest(_:with:)` 的返回值，与背景色无关，所以背景透明的 A 仍会成为目标并挡住触摸。

**方案一：空白处穿透，子按钮照常响应。**

```swift
final class PassthroughOverlay: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let target = super.hitTest(point, with: event)
        return target === self ? nil : target
    }
}
```

`PassthroughOverlay` 的执行结果如下：

- 用户点击覆盖层中的按钮时，`super` 返回按钮，覆盖层原样返回按钮，按钮收到触摸。
- 用户点击覆盖层上没有子视图的位置时，`super` 返回覆盖层自己，覆盖层改为返回 `nil`，父视图接着检查下层的视图。

覆盖层中只用于展示、不需要交互的子视图（例如一个装饰用的 UIView 色块）仍会被命中，并挡住触摸。要让这些位置也穿透，需要把这些子视图的 `isUserInteractionEnabled` 设为 false。

**方案二：指定一个完全穿透的矩形区域。**

```swift
final class HoleOverlay: UIView {
    var passthroughRect: CGRect = .zero

    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        if passthroughRect.contains(point) {
            return false
        }
        return super.point(inside: point, with: event)
    }
}

let overlay = HoleOverlay()
overlay.passthroughRect = CGRect(x: 40, y: 100, width: 120, height: 60)
```

`HoleOverlay` 的规则如下：

- `passthroughRect` 的初始值 `CGRect.zero` 是原点为 (0, 0)、宽高都为 0 的矩形，`contains(_:)` 对任何点都返回 false，所以初始状态下没有穿透区域。
- `passthroughRect` 使用 `overlay` 自己的坐标系。例如局部点 (60, 120) 落在 `CGRect(x: 40, y: 100, width: 120, height: 60)` 内，`point(inside:with:)` 返回 false，这次触摸穿透到下层。
- `point(inside:with:)` 返回 false 时，命中测试跳过整个覆盖层及其子视图，所以孔洞范围内覆盖层的子按钮也收不到触摸。孔洞内的子按钮需要响应时，改用方案一。
- 孔洞要跟随某个视图移动时，在布局变化后重新计算矩形。例如 `overlay.convert(button.bounds, from: button)` 把 `button` 的区域转换成 `overlay` 坐标系中的矩形。

### 扩大按钮的命中区域

重写 `point(inside:with:)` 可以扩大按钮的命中区域，按钮的显示尺寸和布局尺寸保持不变。下面的按钮向四周各扩大 10 pt：当 `bounds` 为 50×50 pt 时，命中区域为 70×70 pt。

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

`bounds.insetBy(dx: -10, dy: -10)` 中的负数表示向外扩展，50×50 的 `bounds` 扩展后为 `CGRect(x: -10, y: -10, width: 70, height: 70)`。

扩大的区域只有落在父视图范围内的部分才有效。原因是命中测试从父视图向子视图逐层进行：触摸点在父视图范围之外时，父视图的 `point(inside:with:)` 返回 false，父视图直接返回 `nil`，命中测试到父视图就结束，按钮的 `point(inside:with:)` 没有被调用。这时还需要扩大父视图的命中区域，或者重写父视图的 `hitTest(_:with:)`，把触摸转给按钮。

`clipsToBounds` 只控制是否裁剪超出视图边界的绘制内容。父视图的 `clipsToBounds` 设为 false 后，子视图超出父视图的部分能显示出来，但落在超出部分的触摸会被父视图的 `point(inside:with:)` 排除，原因同上。

### 触摸回调与响应链转发

触摸序列是一次触摸从按下、移动到抬起或取消的完整过程。触摸回调是 UIKit 在触摸序列各阶段调用的方法：`touchesBegan(_:with:)`、`touchesMoved(_:with:)`、`touchesEnded(_:with:)`、`touchesCancelled(_:with:)` 分别对应按下、移动、抬起和取消。

`UIResponder` 是所有能接收事件的对象的基类，UIView、UIViewController、UIWindow 和 UIApplication 都继承自 `UIResponder`。`next` 属性指向响应链中的下一个响应对象：

- 普通视图的 `next` 是父视图。
- 视图控制器的根视图的 `next` 是这个视图控制器。

所以响应链经过视图控制器的根视图时，会先到视图控制器，再继续向上传递。

触摸回调只交给目标视图。子视图在 `touchesBegan(_:with:)` 中处理触摸后，父视图要收到这个回调，必须由子视图转发。触摸方法没有返回值，UIKit 无法根据返回值判断子视图是否“已经处理”。子视图转发有两种方式：调用会转发的父类实现；显式把事件转给下一个响应对象，例如调用 `next?.touchesBegan(touches, with: event)`。

`super` 指继承关系上的父类，`next` 指响应链上的下一个对象。`super.touchesBegan(touches, with: event)` 执行的是当前类的父类实现。`UIResponder` 的默认实现会把事件转给 `next`；如果父类是 `UIButton` 等控件或自定义类，父类实现可能自己处理触摸并停止转发。所以调用 `super` 后，事件能否到达父视图由父类实现决定。

| 子视图的实现方式 | 对这个触摸回调的影响 |
|-|-|
| 重写触摸方法，执行自定义逻辑，不调用 `super`，也不显式转发 | 回调停在子视图，不再沿响应链传递 |
| 执行自定义逻辑后调用 `super` | 是否继续传递由父类实现决定；执行过自定义逻辑不影响之后的转发 |
| 不重写触摸方法 | 使用继承来的实现；继承的是默认转发行为时，回调会继续传递 |

四个触摸方法要成对重写。如果重写 `touchesBegan(_:with:)` 且不调用 `super`，由子视图自己处理触摸，那么 `touchesMoved(_:with:)`、`touchesEnded(_:with:)`、`touchesCancelled(_:with:)` 也要重写且不调用 `super`，即使方法体为空。只在 `touchesBegan(_:with:)` 中不转发时，后续三个阶段仍会通过继承来的默认实现转发出去，下一个响应对象会收到没有开始阶段的触摸序列。

### 父子视图的手势识别与同时识别

`UIGestureRecognizer` 是手势识别器，根据触摸序列判断用户做了点击、拖动、长按等哪种手势。一个手势识别器同一时刻只能添加在一个视图上；把识别器添加到另一个视图时，识别器会从原来的视图上移除。

命中测试确定目标视图后，目标视图上的识别器，以及目标视图所有祖先视图上的识别器，都会收到这次触摸。祖先视图是沿 `superview` 逐层向上的各层父视图。与目标视图在画面上重叠的兄弟视图，位于另一条视图分支上，兄弟视图上的识别器收不到这次触摸。

例如，`parentView` 包含 `childView`，两者各有一个拖动识别器 `parentPan` 和 `childPan`。手指落在 `childView` 上时，`parentPan` 和 `childPan` 都会收到触摸。两个识别器收到触摸后各自判断手势；默认情况下，其中只有一个能识别成功。

同时识别是指允许两个识别器在同一次触摸中都识别成功。UIKit 会调用识别器代理的 `gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)`，询问两个识别器能否同时识别。下面的方法写在视图控制器中：视图控制器遵循 `UIGestureRecognizerDelegate`，用属性保存 `parentPan` 和 `childPan`，并把两个识别器的 `delegate` 都设为视图控制器。

```swift
func gestureRecognizer(
    _ gestureRecognizer: UIGestureRecognizer,
    shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
) -> Bool {
    (gestureRecognizer === parentPan && otherGestureRecognizer === childPan)
        || (gestureRecognizer === childPan && otherGestureRecognizer === parentPan)
}
```

返回 true 只表示允许 `parentPan` 和 `childPan` 同时识别，不保证两者都识别成功，每个识别器仍要满足自己的识别条件，例如拖动距离。如果还用 `require(toFail:)` 设置了“等另一个识别器失败后才能成功”的依赖，依赖关系同样生效。两个识别器都识别成功后，各自调用自己的动作方法，分别移动父视图和子视图。

UIKit 先把触摸交给目标视图及其祖先上的手势识别器，再把触摸回调交给视图。所以识别器收到触摸早于视图的触摸回调，子视图在 `touchesBegan(_:with:)` 中是否调用 `super`，对父视图的识别器没有影响。识别器识别成功后如何取消视图收到的触摸，见 [同一按钮的点击与长按协调](#同一按钮的点击与长按协调)。

### 同一按钮的点击与长按协调

按钮的点击用 `.touchUpInside` 事件处理，`.touchUpInside` 表示手指在按钮范围内抬起。长按用 `UILongPressGestureRecognizer` 识别。长按识别器直接添加到按钮上时，只有从按钮范围内开始的按压才会被识别为长按。

以下代码写在 `UIViewController` 子类中。`button` 是已经创建好的 `UIButton` 属性，`setupInteractions()` 在界面初始化时调用一次。

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

用户的操作有两种结果：

- 按住不到 0.5 秒就在按钮上松手：长按识别失败，按钮触发 `.touchUpInside`，执行 `handleTap()`。
- 按住达到 0.5 秒，且手指移动没有超过允许距离：长按识别成功，进入 `.began` 状态，执行 `handleLongPress(_:)`。

`cancelsTouchesInView` 为 true 时，长按识别成功后，UIKit 取消按钮收到的触摸，按钮收到 `touchesCancelled(_:with:)`。所以这次松手不会再触发 `.touchUpInside`。UIKit 取消的是按钮的触摸，长按识别器继续跟踪触摸，松手时进入 `.ended`。

如果按钮的动作绑定在 `.touchDown`（手指按下就触发）上，按下时动作已经执行。长按识别成功后再取消触摸，也撤销不了已经执行的动作。

长按识别器添加在哪个视图上，决定了响应范围。长按识别器添加到页面的根视图上时，从根视图任何位置开始的按压都可能被识别为长按。只希望按钮响应长按时，就把识别器添加到按钮上；按钮扩大后的命中区域也算按钮范围。

### 长按状态与取消后的恢复

`UILongPressGestureRecognizer` 是连续手势识别器：识别成功后，识别器会继续跟踪触摸，在手指移动、抬起或触摸被取消时再次调用动作方法。动作方法通过 `gesture.state` 区分当前阶段。

| 状态 | 含义 | 适合执行的操作 |
|-|-|-|
| `.began` | 达到长按条件，长按识别成功 | 保存原状态，开启临时效果 |
| `.changed` | 长按已经开始，手指位置等触摸信息发生变化 | 按需要更新连续交互，例如跟随手指移动 |
| `.ended` | 已识别的长按正常结束，例如用户松手 | 恢复原状态，结束临时效果 |
| `.cancelled` | 已识别的长按被中断 | 同样恢复原状态并清理 |
| `.failed` | 识别成功之前条件就不满足，例如按住时间不够就松手 | 不执行只属于长按成功的操作 |

例如，长按开始后来电界面弹出并接管了交互，UIKit 取消当前触摸，识别器进入 `.cancelled`。识别器只在触摸被系统实际中断时进入 `.cancelled`；来电横幅这类不接管交互的界面出现时，触摸可能继续保留。

长按开始后，手指移出按钮，长按仍会继续，识别器保持在 `.changed` 状态。需要“移出按钮就结束”时，要在 `.changed` 状态中用 `gesture.location(in: button)` 读取手指位置，自己判断手指是否已经离开 `button.bounds`。

临时效果（例如长按时把按钮放大）要在 `.began` 时保存原状态，在 `.ended` 和 `.cancelled` 时都恢复。只在 `.ended` 中恢复的话，长按被取消时临时效果会一直留在界面上。

## 屏幕键盘：触摸位置与文字接收对象

第一响应者是当前优先接收键盘输入等事件的响应对象。在文字输入中，成为第一响应者的输入框就是文字的输入目标。触摸事件仍按触摸位置分发。

假设页面同时显示用户名框和密码框，用户使用屏幕键盘输入：

1. 用户点击密码框。UIKit 根据触摸位置找到密码框，密码框成为第一响应者，系统显示屏幕键盘。
2. 用户点击屏幕键盘上的 A 键。这次触摸有坐标，键盘用这个坐标判断用户按了哪个键。
3. 系统把字母 A 交给当前的第一响应者，也就是密码框。A 键的位置只用来判断按的是哪个键，与文字写进哪个输入框无关。
4. 用户点击用户名框，用户名框成为第一响应者。用户再按同一个 A 键，字母 A 就输入到用户名框。键盘位置没变，变的是第一响应者。

所以，点击屏幕键盘时有触摸坐标，但触摸坐标只用来确定按键；文字写到哪个输入框，由第一响应者决定。

关键窗口和第一响应者分工不同：关键窗口决定键盘输入交给哪个窗口，第一响应者决定交给窗口中的哪个对象。标准文本输入控件只有在所在窗口是关键窗口、并且自己成为第一响应者时，才能正常弹出屏幕键盘。

外接键盘的按键事件完全没有触摸坐标，文字同样交给第一响应者。
