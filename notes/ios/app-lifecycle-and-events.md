# iOS 应用启动、窗口与输入事件

[返回笔记目录](../README.md)

## 本文目录

- [UIApplication、AppDelegate、UIWindow 与 UIView 的职责](#uiapplicationappdelegateuiwindow-与-uiview-的职责)
- [应用运行状态与生命周期回调](#应用运行状态与生命周期回调)
  - [五种运行状态](#五种运行状态)
  - [AppDelegate 的主要回调](#appdelegate-的主要回调)
  - [进入前台与进入激活状态的区别](#进入前台与进入激活状态的区别)
  - [常见状态变化与调用顺序](#常见状态变化与调用顺序)
  - [Scene 生命周期与应用生命周期](#scene-生命周期与应用生命周期)
  - [挂起、终止与数据保存](#挂起终止与数据保存)
- [AppDelegate 生命周期下，从启动到首帧显示](#appdelegate-生命周期下从启动到首帧显示)
  - [LaunchScreen、自定义启动页与异步加载](#launchscreen自定义启动页与异步加载)
  - [main 之前的启动工作与 dyld](#main-之前的启动工作与-dyld)
  - [动态库数量与启动开销](#动态库数量与启动开销)
  - [删除无用代码与地址修正成本](#删除无用代码与地址修正成本)
  - [UIApplication 与主线程 RunLoop](#uiapplication-与主线程-runloop)
  - [进程创建时间与启动计时起点](#进程创建时间与启动计时起点)
  - [启动优化的三个方向](#启动优化的三个方向)
- [UIApplication 的使用场景：打开应用设置](#uiapplication-的使用场景打开应用设置)
- [为什么一个应用可以有多个 UIWindow](#为什么一个应用可以有多个-uiwindow)
- [关键窗口与 makeKeyAndVisible()](#关键窗口与-makekeyandvisible)
- [UIApplication 与 UIWindow 为什么都分发事件](#uiapplication-与-uiwindow-为什么都分发事件)
  - [触摸回调与响应链转发](#触摸回调与响应链转发)
  - [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)
  - [UIView 的局部坐标与 bounds](#uiview-的局部坐标与-bounds)
  - [point(inside:with:) 与 hitTest(_:with:)](#pointinsidewith-与-hittest_with)
  - [事件穿透与重叠视图的触摸目标](#事件穿透与重叠视图的触摸目标)
  - [扩大按钮的命中区域](#扩大按钮的命中区域)
  - [同一按钮的点击与长按协调](#同一按钮的点击与长按协调)
  - [长按状态与取消后的恢复](#长按状态与取消后的恢复)
- [屏幕键盘：触摸位置与文字接收对象](#屏幕键盘触摸位置与文字接收对象)

## UIApplication、AppDelegate、UIWindow 与 UIView 的职责

UIApplication 是 UIKit 提供的应用对象，负责应用级事件分发、应用状态以及与系统交互的接口。AppDelegate 是开发者提供的应用代理对象，通过 UIApplicationDelegate 协议接收应用级回调，并执行自定义初始化等逻辑。

| 对象 | 主要职责 | 例子 |
|-|-|-|
| UIApplication | 管理应用级行为，并将事件分发给适当的对象 | 通过 UIApplication.shared.open(...) 请求系统打开设置页面 |
| AppDelegate | 响应应用生命周期和应用级回调 | 在 application(\_:didFinishLaunchingWithOptions:) 中初始化必要服务和配置主窗口 |
| UIWindow | 承载视图层级，并参与向视图分发事件 | 设置 rootViewController、显示窗口、成为关键窗口 |
| UIView | 表示界面中的一块区域，展示内容并参与交互 | 按钮接收点击；UITextField 作为 UIView 的子类接收文字输入 |

UIApplication 与 AppDelegate 是两个不同的对象。UIKit 在特定时机调用 AppDelegate 的回调方法，让开发者执行自己的代码；按钮触摸等普通事件不经过 AppDelegate。

## 应用运行状态与生命周期回调

应用生命周期是应用从启动、进入不同运行状态，到进程终止的过程。生命周期回调是 UIKit 在指定阶段调用的代理方法；应用在回调中执行初始化、暂停、恢复或保存等工作。

下面先按传统 AppDelegate 生命周期说明应用状态与回调顺序。AppDelegate 中以 application 开头的回调方法，参数是当前的 UIApplication 对象。启用 Scene 生命周期后，界面状态变化改由 SceneDelegate 接收，见下文 [Scene 生命周期与应用生命周期](#scene-生命周期与应用生命周期)。

### 五种运行状态

| 状态 | 含义 | 关键边界 |
| --- | --- | --- |
| Not Running（未运行） | 应用进程尚未启动，或应用进程已经终止 | 再次打开应用需要重新启动进程 |
| Inactive（未激活） | 应用在前台运行，但不接收触摸等用户交互事件 | 应用仍在执行代码，网络请求完成等其他事件照常处理 |
| Active（已激活） | 应用在前台运行，并接收用户交互事件 | 只说明可以交互，网络数据和首帧不一定已经准备好 |
| Background（后台） | 应用不在前台，但在系统允许的时间内仍可执行代码 | 普通应用做完收尾工作后会被挂起；只有开启了音频、定位等后台能力的工作可以继续执行 |
| Suspended（挂起） | 进程还在内存中，但不执行任何代码 | 系统可以随时终止挂起的进程回收内存，内存中的数据随之丢失 |

前台包括 Inactive 和 Active 两种状态，所以“进入前台”不等于“已经激活”。后台和挂起也不同：后台还能运行代码，挂起后不能。音频、定位等后台能力需要配置，并受系统规则限制，不能让任意任务一直运行。

这五种状态是为了讲清生命周期而做的划分。UIApplication.applicationState 只有 active、inactive、background 三个值；应用未运行或挂起时，自己的代码不执行，也就无从查询状态。

### AppDelegate 的主要回调

| 回调 | UIKit 调用的阶段 | 常见用途 |
| --- | --- | --- |
| application(_:didFinishLaunchingWithOptions:) | 启动初始化接近完成，应用即将开始运行 | 初始化启动必需的服务；纯代码界面在这里创建窗口 |
| applicationWillResignActive(_:) | 应用即将从 Active 变为 Inactive | 暂停依赖用户交互的任务，按需停止计时器或降低画面更新频率 |
| applicationDidEnterBackground(_:) | 应用已经进入 Background | 保存尚未保存的必要状态，释放可重建的资源，并结束不需要继续的工作 |
| applicationWillEnterForeground(_:) | 应用即将从后台返回前台，进入 Inactive 阶段 | 检查后台期间的数据变化，按需重新加载资源或刷新数据 |
| applicationDidBecomeActive(_:) | 应用已经从 Inactive 进入 Active | 恢复因失去激活状态而暂停的任务或计时器 |
| applicationWillTerminate(_:) | 应用即将终止，且系统有机会通知应用时 | 执行简短的清理；很多终止情况不会调用这个方法，不能依赖它保存数据 |

UIKit 只负责调用回调；计时器是否暂停、数据是否刷新，由应用在回调中写的代码决定。调用 applicationWillResignActive 后，应用不一定会进入后台：短暂中断结束后，应用会直接恢复为 Active。

启动初始化与首帧显示的具体过程见[AppDelegate 生命周期下，从启动到首帧显示](#appdelegate-生命周期下从启动到首帧显示)。

### 进入前台与进入激活状态的区别

applicationWillEnterForeground 表示“应用即将从后台回到前台”，applicationDidBecomeActive 表示“应用已经可以接收用户交互”。应用可能没有进入后台，只是暂时失去激活状态后又恢复，所以两个回调不总是成对出现。

| 场景：采用传统 AppDelegate 生命周期 | applicationWillEnterForeground | applicationDidBecomeActive |
| --- | --- | --- |
| 应用从未运行状态直接启动到前台，并正常激活 | 不调用 | 调用 |
| 应用进程仍存在，从后台返回前台，并正常激活 | 先调用 | 后调用 |
| 应用仅暂时变为 Inactive，没有进入后台，随后恢复 Active | 不调用 | 调用 |

“每次从后台回来都刷新数据”适合放在 applicationWillEnterForeground；“每次恢复交互时恢复暂停的任务”适合放在 applicationDidBecomeActive。放在哪个回调，取决于任务需要在什么条件下触发。

如果在 applicationWillEnterForeground 中发起异步网络请求，UIKit 不会等请求完成再调用 applicationDidBecomeActive。所以 applicationDidBecomeActive 被调用时，数据可能还没回来；应用可以先显示已有数据，请求完成后再更新界面。

### 常见状态变化与调用顺序

以下流程都假设状态转换正常完成；遇到系统中断、后台启动或进程被终止，回调顺序会不同。

**从未运行状态直接启动到前台。** UIKit 执行启动回调，然后在应用进入 Active 时调用 applicationDidBecomeActive。由于应用没有从后台返回，UIKit 不调用 applicationWillEnterForeground。

```text
Not Running → 启动初始化 → Inactive → Active
主要回调：willFinishLaunchingWithOptions
       → didFinishLaunchingWithOptions
       → applicationDidBecomeActive
```

**从前台进入后台，再返回前台。** 应用先失去激活状态，再进入后台。应用未被终止且正常返回时，UIKit 先通知进入前台，再通知已经激活。

```text
离开前台：Active → Inactive → Background → 可能进入 Suspended
主要回调：applicationWillResignActive → applicationDidEnterBackground

返回前台：若已挂起，先恢复执行；随后 Background → Inactive → Active
主要回调：applicationWillEnterForeground → applicationDidBecomeActive
```

进入后台后不一定会被挂起。如果系统已经终止了后台的进程，用户再次打开应用时会重新启动进程，走的是启动流程，而不是“从后台返回”的流程。

**只发生前台短暂中断。** 例如，打开后关闭控制中心；如果应用期间只进入 Inactive，没有进入 Background，UIKit 只通知失去激活状态和恢复激活状态。

```text
Active → Inactive → Active
主要回调：applicationWillResignActive → applicationDidBecomeActive
```

来电或弹出系统界面时，应用不一定进入后台；调用哪些回调，取决于实际发生了哪些状态转换。

### Scene 生命周期与应用生命周期

Scene 是应用的一个界面实例，有自己的前后台状态和激活状态；例如 iPad 上同一个应用开两个窗口，就是两个 Scene。UIWindowScene 是管理窗口的 Scene。SceneDelegate 是开发者提供的场景代理对象，遵循 UISceneDelegate 协议，接收 Scene 的状态回调。一个应用进程可以同时有多个 Scene。

Scene 生命周期从 iOS 13 开始提供。应用启用后，UIKit 把下列界面状态变化通知给对应的 SceneDelegate，不再调用 AppDelegate 中对应的四个方法。SceneDelegate 回调的参数是发生状态变化的那个 Scene。

| AppDelegate 回调 | SceneDelegate 对应回调 |
| --- | --- |
| applicationWillResignActive(_:) | sceneWillResignActive(_:) |
| applicationDidEnterBackground(_:) | sceneDidEnterBackground(_:) |
| applicationWillEnterForeground(_:) | sceneWillEnterForeground(_:) |
| applicationDidBecomeActive(_:) | sceneDidBecomeActive(_:) |

AppDelegate 仍负责应用启动等应用级工作，SceneDelegate 负责某一个 Scene 的界面状态。一个 Scene 进入后台时，其他 Scene 可能仍在前台。

**首次进入前台时的区别。** 新创建的 Scene 第一次进入前台时，也会调用 sceneWillEnterForeground；而应用首次启动时不调用 applicationWillEnterForeground。

应用、Scene 和 UIViewController 各有自己的生命周期。页面出现或消失，与应用进入前台或后台是两回事；页面层面的说明见[UIKit 布局、列表与 MVVM](uikit-layout-and-mvvm.md#页面生命周期与展示方式)。

### 挂起、终止与数据保存

应用被挂起后不再执行代码。系统可以直接终止挂起的进程，这时不会调用 applicationWillTerminate，应用也没有机会保存数据。用户在应用切换器中划掉应用时，也不一定调用 applicationWillTerminate。

持久保存是把数据写入文件、数据库等存储，进程结束后数据仍然保留。重要数据应在产生或修改后及时保存；进入后台时可以再保存一次尚未保存的状态，但不能只依赖某一个生命周期回调。具体存储方式见[iOS 数据持久化](../network-and-storage/ios-data-persistence.md)。

例如，用户编辑文本时，应用定期把草稿写入文件。即使系统之后在挂起期间终止了应用，重新启动后仍能读到草稿；只在内存中、还没写入文件的修改则会丢失。

## AppDelegate 生命周期下，从启动到首帧显示

适用条件：应用使用传统 AppDelegate 生命周期，用代码创建界面，从未运行状态启动到前台。不包括系统预热、状态恢复等情况。应用已经在运行时，从后台回到前台不会重新走这套启动流程。

1. 系统创建应用进程。dyld 是系统的动态加载器，负责加载可执行文件和依赖的动态库，并完成链接。进入 main 之前，还要执行运行时初始化和启动阶段的初始化代码。
2. 程序进入 UIApplicationMain。UIApplicationMain 创建 UIApplication 和 AppDelegate，把 AppDelegate 设为 UIApplication 的代理，并启动主事件循环。主事件循环不断接收并分发事件。
3. UIKit 依次调用 application(\_:willFinishLaunchingWithOptions:) 和 application(\_:didFinishLaunchingWithOptions:)，应用在其中完成启动所需的初始化，并检查启动参数。
4. 在常见的纯代码实现中，AppDelegate 在 didFinishLaunchingWithOptions 中创建 UIWindow，设置 rootViewController，并调用 makeKeyAndVisible()。rootViewController 是管理窗口主要视图层级的根视图控制器。
5. 窗口要显示时，UIKit 加载根视图控制器的视图：调用 loadView 创建视图，加载完成后调用 viewDidLoad，然后执行布局和绘制。
6. 应用把准备好的界面提交给系统，系统渲染、合成后显示首帧。首帧是用户第一次看到的应用界面，此时网络请求和业务数据不一定已经加载完成。

didFinishLaunchingWithOptions 返回、makeKeyAndVisible() 返回，都不代表首帧已经显示；applicationDidBecomeActive(\_:) 也不是首帧显示的回调。用 Storyboard 自动创建界面时，窗口和根控制器由 UIKit 根据 Storyboard 创建。

### LaunchScreen、自定义启动页与异步加载

`LaunchScreen` 是应用界面准备好之前，系统显示的启动画面。用 Launch Screen Storyboard 配置的启动画面只能是静态布局，不能运行自定义控制器的代码，所以不能做网络请求、倒计时或响应按钮。

需要动态内容或交互的启动页，是应用自己创建的普通视图控制器。注意区分应用首帧和首页首帧：如果窗口先显示启动页，应用首帧就是启动页，首页之后才显示。

动态启动页的职责可以按执行顺序划分：

1. 窗口提供显示位置。首次启动时，如果启动页必须在首页之前显示，就把启动页的容器控制器设为 `rootViewController`，结束后再切换到首页。启用 Scene 生命周期时，窗口由对应的 Scene 管理。
2. 加载组件异步获取素材，完成后返回素材或错误。等待期间要先显示一个可以绘制的界面，不能在主线程同步等待素材，否则首帧会被推迟。
3. 启动页显示已准备好的素材，在页面出现（`viewDidAppear`）后开始倒计时。视图加载完成时页面还没出现，回调顺序见[控制器初始化、视图加载与页面展示](uikit-layout-and-mvvm.md#控制器初始化视图加载与页面展示)。
4. 关闭时，停止倒计时，取消未完成的任务，并通知管理者显示首页。已取消的任务如果之后仍返回结果，要先检查启动页是否已关闭，避免刷新已关闭的页面。

素材加载要设置等待上限：加载失败或超时，管理者都直接进入首页，不能让首页一直等网络。这里有两个不同的回调：素材请求结束时，加载组件调用管理者传入的完成回调，传入素材或错误；启动页关闭时，启动页调用管理者传入的关闭回调。“素材加载完成”和“启动页已关闭”是两件事。

应用已经有可见页面时（例如从后台回到前台），也可以用 `present` 展示动态启动页。前提是发起 `present` 的控制器正在显示，并且当前没有正在进行的其他转场；把 `present` 写在启动回调中，并不能保证这些前提成立。

如果每次进程启动只展示一次，展示记录要保存在与进程同生命周期的管理者中，不能只靠 `applicationDidBecomeActive`、`sceneDidBecomeActive` 或 `viewDidAppear` 判断，因为这些回调会反复调用。进程终止后再打开，算新的一次启动；前后台切换则仍是原来的进程。

倒计时的显示值要用“截止时刻 − 当前时刻”计算，不能假设每次回调正好间隔一秒。截止时刻等于开始时刻加展示时长，时间要取自单调时钟（不受系统时间修改影响）。回调可能延迟，每次都重新计算剩余时间，显示的才是实际剩余时间。任务类型和取消规则见[任务结果类型与 Void、Never](../language/swift-basics.md#任务结果类型与-voidnever)与[上下文继承与取消](../language/swift-basics.md#上下文继承与取消)。

### main 之前的启动工作与 dyld

`main` 是程序的入口函数，pre-main 是进入 `main` 之前的启动阶段。Swift 的 UIKit 应用用 `@main` 让编译器生成入口代码，所以源码中通常看不到 `main` 函数。

`dyld` 是 Apple 平台的动态链接器。系统创建进程、映射主可执行文件后，dyld 按可执行文件中记录的依赖加载所需的动态库，再加载这些库自己依赖的库。动态库加载后，dyld 修正地址并绑定符号。符号是函数、变量等的名称；符号绑定是把代码中对某个符号的引用，指向这个符号在内存中的实际地址。

进入 `main` 前，还要完成运行时初始化，并执行启动阶段的初始化代码。例如，Objective-C 运行时调用类和分类的 `+load`；启动时加载的代码还会执行 C++ 全局对象的构造函数，以及标记为初始化函数的代码。这些工作都计入启动时间。

但并非所有 C++ 静态对象都在 main 之前构造：需要动态初始化的函数内静态局部对象，在程序第一次执行到声明时才构造。对象的存储期与构造时机见 [C++ 静态存储期对象与初始化](../language/cpp-static-storage-and-initialization.md)。

静态库的代码在构建时已经合并到可执行文件中，dyld 在运行时不会加载 `.a` 文件。动态库的链接、嵌入和装载见 [编译、链接、嵌入与装载](../engineering/xcode-and-testing.md#编译链接嵌入与装载)。

### 动态库数量与启动开销

启动时要加载的自定义动态库越多，系统要打开的文件、建立的内存映射、处理的依赖和地址修正就越多。内存映射是把文件内容对应到进程的地址空间，访问到时才真正读取，不会一次读入整个文件。系统会缓存部分工作结果，但剩下的开销仍随库的数量、大小和实际访问的数据增加。

例如，把 10 个各约 1 MB 的动态库合并成 1 个约 10 MB 的动态库，代码总量基本不变，但要加载和管理的库从 10 个变成 1 个，原来跨库的部分引用也可以在构建时解决。这只说明减少库的数量能降低额外开销，启动耗时并不会按库的数量成比例下降。

合并动态库不会去掉代码中的 `+load` 等初始化工作。系统框架有共享缓存等专门优化，所以自定义动态库的开销不能套用到 UIKit 等系统框架上。

“自定义动态库控制在 6 个以内”只是经验值，iOS 没有这个硬性上限，超过 6 个也不会突然变慢。静态链接也是减少独立动态库的方式，构建速度、代码共享及可合并库的取舍见[静态链接与动态链接的取舍](../engineering/xcode-and-testing.md#静态链接与动态链接的取舍)。

### 删除无用代码与地址修正成本

删除无用的类、方法和全局变量，可能同时删掉一些需要修正的地址。类的元数据是描述类、方法和继承关系的数据，其中保存着指向方法实现、父类等的指针，这些指针在加载时都需要修正。只有当删除的内容原本确实留在最终产物中，删除后需要修正的指针变少了，Rebase 和 Binding 的耗时才会减少。两种操作的机制与示例见[Rebase 与 Binding：地址修正](../engineering/xcode-and-testing.md#rebase-与-binding地址修正)。

整数类型的全局变量保存的是数值，加载地址变化时不需要修正；只有保存地址的指针才需要修正。方法数量与修正次数也不是一一对应的。如果某段代码在构建时已经被裁剪掉，再删除对应的源码不会减少运行时的工作；裁剪条件见[静态库的目标文件提取与代码裁剪](../engineering/xcode-and-testing.md#静态库的目标文件提取与代码裁剪)。

删除前要确认代码没有被运行时按名称查找、没有被配置文件引用，也不是初始化入口。优化后要分别检查代码体积、地址修正次数和首屏耗时，不能只看删了多少代码。

### UIApplication 与主线程 RunLoop

`UIApplicationMain` 创建 `UIApplication` 单例和 AppDelegate，并启动主事件循环。业务代码通过 `UIApplication.shared` 访问这个应用对象。UIApplication 管理应用级状态、分发事件，还提供打开 URL、注册远程通知等接口；具体页面由窗口、视图控制器和视图负责。

主事件循环就是 UIKit 在主线程上运行的 RunLoop，负责等待和分发事件。RunLoop 是在线程上等待输入源、定时器等事件，并调用对应处理函数的循环。UIKit 应用的主 RunLoop 由框架启动，业务代码不需要调用 `RunLoop.main.run()`。

以按钮点击为例：系统把触摸事件发给应用，主线程 RunLoop 被唤醒并开始处理；UIKit 通过 UIApplication、UIWindow 等对象把事件交给按钮，最终执行按钮的动作。RunLoop 决定什么时候处理事件，UIApplication 等 UIKit 对象决定事件交给谁。

没有事件时，RunLoop 让主线程休眠；事件到来后，主线程被唤醒，等系统调度后继续执行。如果按钮回调长时间占用主线程，后续的触摸处理和界面更新都会延迟，RunLoop 无法打断正在执行的回调。运行模式和输入源见 [iOS 线程与任务调度](threads-and-scheduling.md#runloop-与运行模式)。

### 进程创建时间与启动计时起点

进程创建时间是系统记录的进程开始存在的时刻，比进入 `main()` 早。只从 `main()` 开始计时，会漏掉 `main()` 之前的启动阶段。

`sysctl` 是 Darwin 系统提供的 C 接口，用来查询系统信息，有权限的调用方还可以修改部分信息。PID 是系统分配给进程的编号。查询当前进程的信息时，调用方传入当前 PID 和一块缓冲区，系统把查询结果写入缓冲区。

1. 获取当前进程的 PID。
2. 通过 `sysctl` 的 `CTL_KERN → KERN_PROC → KERN_PROC_PID → PID` 查询路径请求进程信息。
3. 查询成功并获得有效结果后，从 `kinfo_proc` 结构中的 `kp_proc.p_un.__p_starttime` 读取进程启动时间。
4. 把秒数 `tv_sec` 和微秒数 `tv_usec` 合并：时间戳（秒）= `tv_sec + tv_usec / 1_000_000.0`。时间戳是从某个固定时间原点算起的时刻，不是耗时。

查到的是进程创建的时刻，不是查询的时刻。例如：10:00:00.000 创建进程，10:00:00.200 进入 `main()`，10:00:00.300 查询，得到的是 10:00:00.000。

统计前要先确定起点和终点。用进程创建时间作起点，可以算上 `main()` 之前的阶段，但进程创建时间也不等于用户点击图标的时间。系统预热是指系统在用户打开 App 之前，提前执行部分启动工作；有预热时，进程创建时间可能比用户点击早很多，要把预热阶段和用户实际等待的时间分开。相减的两个时间还必须来自同一种时钟、使用相同的单位。

### 启动优化的三个方向

启动优化先要确定测量的起点和终点，并把工作分为三段：进入 main 之前、首屏显示之前、首屏显示之后。只有减少用户等待期间执行的工作，启动耗时才会下降。

**减少首屏不需要的初始化。** 初始化是为组件创建对象、读取配置、准备运行条件。把首屏用不到的初始化推迟到功能第一次使用时，首屏会更快，但第一次使用功能时要承担这部分耗时。`+load`、`+initialize` 和功能入口的时机、同步等待及注册顺序见[Objective-C 类初始化与按需准备](../language/swift-objc-runtime.md#initialize-的触发时机与执行限制)。

**减少主线程上的同步耗时工作。** 主线程负责界面和输入事件。在主线程同步扫描文件、解析大量数据或解码大图，都会推迟界面显示。例如，启动时先读取一份很小的首页摘要并显示界面，再在后台读取和解析完整的历史记录，完成后回到主线程刷新对应区域。只有不涉及 UIKit 的工作才能移到后台；而且把任务放到后台后，主线程如果立即等待它完成，耗时并没有减少。

**减少首屏的视图和布局工作。** 只创建当前需要显示的界面，按展示尺寸准备图片，并避免重复创建约束。例如，首页有多个标签页时，先创建默认标签页的内容，其他页面在需要时加载；列表优先配置当前显示需要的项目。具体布局优化见 [UIKit 布局、列表与 MVVM](uikit-layout-and-mvvm.md)。

对已经确认发生在 main 之前的耗时初始化，按上文的启动阶段调整执行时机，并去掉不需要的依赖。判断瓶颈要看实际测量结果，不能只看动态库或视图的数量。验证时保持设备、启动类型、数据和缓存条件一致，并同时检查首屏可交互的时间，避免只是把耗时转移到了用户接下来的操作上。

## UIApplication 的使用场景：打开应用设置

用户拒绝照片访问后，应用可以提供“去设置”按钮。按钮被点击后，业务代码使用 UIApplication.openSettingsURLString 构造 URL，再通过 UIApplication.shared.open(...) 请求系统打开本应用的设置页面。

执行顺序是：按钮接收点击 → 业务代码调用 UIApplication 的接口 → 系统打开本应用的设置页面。UIApplication 只提供请求入口，打开设置页面由系统完成。

## 为什么一个应用可以有多个 UIWindow

UIWindow 是应用界面中的一个视图对象，负责承载视图层级，与进程无关，也不同于桌面系统中带边框、可拖动的窗口。同一个应用进程可以创建多个 UIWindow。

多数应用只需要一个主 UIWindow。额外的窗口可以显示额外的内容，叠加在主窗口上面。所以用户看到的画面可能由多个窗口叠成，看起来只有一个应用界面，内部也可能有多个 UIWindow。

例如，主 UIWindow 显示首页，再用一个背景透明的 UIWindow 显示悬浮的调试按钮，用户看到的就是首页上浮着一个按钮。这只是一种做法，悬浮按钮也可以直接用普通 UIView 实现。

切换首页和详情页通常通过视图控制器或视图切换完成，不需要为每个页面新建 UIWindow。

## 关键窗口与 makeKeyAndVisible()

关键窗口（key window）是 UIKit 指定的、接收键盘输入等没有触摸坐标的事件的窗口。窗口是不是关键窗口，与窗口是否可见是两回事。

| 操作 | 作用 |
|-|-|
| makeKey() | 请求窗口成为关键窗口，不负责显示隐藏的窗口 |
| isHidden = false | 显示窗口，不等同于将窗口设为关键窗口 |
| makeKeyAndVisible() | 显示窗口，并将窗口设为关键窗口 |

makeKeyAndVisible() 还会把窗口放到 windowLevel（窗口级别）相同或更低的窗口前面。是否为关键窗口，与窗口叠放的前后顺序也是两回事。

非关键窗口照样可以显示，触摸落在它上面时也照样能接收。

“关键窗口只有一个”是对一个 Scene 而言的。不使用 Scene 的传统应用只有一个关键窗口；使用多个 Scene 时，每个 Scene 各有自己的关键窗口。

## UIApplication 与 UIWindow 为什么都分发事件

事件分发是把事件交给合适的对象处理。UIApplication 与 UIWindow 负责分发的层级不同，所以两者都参与分发。

以触摸事件为例，路径可以简化为：UIApplication → 对应的 UIWindow → 窗口内的目标 UIView。UIApplication 把事件交给窗口，UIWindow 再把事件交给目标视图。这里省略了手势识别器等其他参与者。

命中测试是根据触摸位置和视图层级，找出触摸落在哪个视图上。响应链是由每个响应对象的 next 属性串起来的一条链，事件沿着这条链传递。命中测试负责找到第一个接收触摸的视图，响应链负责之后把事件继续往后传。

### 触摸回调与响应链转发

子视图在 `touchesBegan(_:with:)` 中处理了触摸后，父视图不会自动收到这个回调。事件是否继续传递，取决于子视图的实现是否调用了父类实现（父类实现会转发），或者自己显式转发给下一个响应对象。`touchesBegan(_:with:)` 等方法没有返回值，UIKit 无法知道事件是否“已处理”。

`UIResponder` 是所有能接收事件的对象的基类，`next` 属性指向响应链中的下一个响应对象。普通视图的 `next` 是父视图；如果视图是某个视图控制器的根视图，`next` 是这个视图控制器。所以响应链不一定都沿父视图向上。

`super.touchesBegan(touches, with: event)` 调用的是当前类的父类实现，不是父视图的方法。`UIResponder` 的默认实现会把事件转给 `next`；但如果父类是某个控件或自定义类，父类实现可能自己处理触摸而不转发，所以调用了 `super` 也不一定能传到父视图。

| 子视图的实现方式 | 对该触摸回调的影响 |
| --- | --- |
| 重写触摸方法，执行自定义逻辑，不调用 `super`，也不显式转发 | 该回调不会由这个实现继续沿响应链传递 |
| 执行自定义逻辑后调用 `super` | 是否继续传递由父类实现决定；执行过自定义逻辑不妨碍之后继续转发 |
| 未重写触摸方法 | 使用继承来的实现；继承的是默认转发行为时，会继续传递 |

触摸序列是一次触摸从开始、移动到结束或取消的完整过程，`touchesBegan`、`touchesMoved`、`touchesEnded`、`touchesCancelled` 分别对应这几个阶段。如果重写 `touchesBegan` 且不调用 `super` 来自己处理触摸，其余三个方法也要重写且不调用 `super`，即使方法体是空的。只在 `touchesBegan` 中不转发，后续阶段仍会通过默认实现转发出去。

### 父子视图的手势识别与同时识别

`UIGestureRecognizer` 是根据触摸序列识别点击、拖动、长按等手势的对象。一个识别器同一时刻只能添加在一个视图上；把它添加到另一个视图，它会从原来的视图上移除。

一个触点经过命中测试后确定一个目标视图。添加在目标视图上的识别器，以及添加在目标视图所有祖先视图（沿 `superview` 向上的各层父视图）上的识别器，都会收到这个触点的触摸。只是在画面上重叠、但不是祖先的兄弟视图，收不到这次触摸。

例如，`parentView` 包含 `childView`，两者各有一个拖动识别器 `parentPan` 和 `childPan`。触摸落在 `childView` 上时，两个识别器都会收到触摸。但收到触摸不等于识别成功；默认情况下，两个识别器中只有一个能识别成功。

同时识别是指允许两个识别器在同一次交互中都识别成功。UIKit 会调用识别器代理的 `gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)`，询问两个识别器能否同时识别。下面的方法写在遵循 `UIGestureRecognizerDelegate` 的对象中，`parentPan` 和 `childPan` 是这个对象保存的识别器属性，两个识别器的 `delegate` 都已设为这个对象。

```swift
func gestureRecognizer(
    _ gestureRecognizer: UIGestureRecognizer,
    shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
) -> Bool {
    (gestureRecognizer === parentPan && otherGestureRecognizer === childPan)
        || (gestureRecognizer === childPan && otherGestureRecognizer === parentPan)
}
```

返回 `true` 只是允许这两个识别器同时识别，并不保证它们都识别成功，每个识别器仍要满足自己的识别条件。如果还设置了“等另一个识别器失败后才能成功”的依赖，也要一起考虑。识别成功后，每个识别器调用自己的动作方法，分别更新父视图和子视图。

祖先视图上的识别器能收到触摸，与子视图是否在 `touchesBegan` 中调用 `super` 无关：UIKit 先把触摸交给相关的手势识别器，再把触摸回调交给视图。所以子视图不转发触摸回调，也挡不住父视图的识别器。识别器如何取消视图收到的触摸，见下文 [同一按钮的点击与长按协调](#同一按钮的点击与长按协调)。

### UIView 的局部坐标与 bounds

UIView 的局部坐标系用于描述视图内部的位置。默认情况下，bounds.origin 为 (0, 0)，可见矩形的左上角对应局部坐标 (0, 0)，x 轴向右，y 轴向下。

bounds 为 CGRect(x: 0, y: 0, width: 50, height: 50) 的按钮，局部坐标中的中心点是 (25, 25)，右下角是 (50, 50)，单位是点（pt）。无论按钮放在父视图的什么位置，这些局部坐标都不变。

bounds 是视图在自身坐标系中的矩形，frame 是视图在父视图坐标系中的矩形。修改 bounds.origin 后，可见区域左上角的局部坐标就不再是 (0, 0)。UIView.center 使用的是父视图坐标系，与局部中心点不同。

### point(inside:with:) 与 hitTest(_:with:)

point(inside:with:) 判断一个局部坐标点是否属于当前视图的命中区域，返回 Bool。hitTest(_:with:) 在当前视图及其子视图中寻找触摸目标，返回 UIView?；找不到目标时返回 nil。

UIView 默认命中测试可以按以下顺序理解：

1. 当前视图隐藏、关闭了用户交互（isUserInteractionEnabled 为 false），或 alpha 小于 0.01 时，返回 nil。
2. 调用当前视图的 point(inside:with:)。结果为 false 时，返回 nil，并跳过当前视图的子树。
3. 按从前向后的顺序检查子视图。先把触摸点转换到对应子视图的坐标系，再调用子视图的 hitTest(_:with:)。
4. 某个子视图找到目标时，返回该目标。所有子视图都没有找到目标时，返回当前视图。

两个方法接收的 point 都是当前视图自己坐标系中的坐标。所以重写命中区域时，应该把 point 与 bounds 比较，而不是与 frame（父视图坐标系）比较。

子视图按 subviews 数组的倒序检查，也就是先检查显示在最上面的子视图。触摸开始时确定了目标视图后，这次触摸从头到尾都交给这个视图；手指移到其他视图上，触摸也不会转交给那个视图。

需要让容器自身接管原本命中子视图的触摸时，可以改变 hitTest 的返回值：

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

super 返回非 nil，说明触摸点在容器范围内，且默认规则找到了目标。interceptsTouches 为 true 时，容器返回 self，这次触摸的目标就变成容器，子控件收不到触摸。先调用 super 是为了保留隐藏、交互关闭、命中区域等默认判断。容器成为目标后，还要通过触摸方法或手势识别器自己执行动作；另外，这样做也挡不住祖先视图上的手势识别器。

### 事件穿透与重叠视图的触摸目标

事件穿透是让覆盖在上面的视图在某些位置不作为触摸目标，于是父视图继续检查下面的兄弟视图。穿透发生在命中测试选择目标的阶段：上面的视图根本没有收到这次触摸，而不是收到后再转给下面的视图。

假设父视图中有下面的按钮 B 和盖在上面的视图 A。UIKit 先检查 A；只要 A 或 A 的子视图被选为目标，B 就不会被选中。只有 A 的 hitTest 返回 nil 时，父视图才会接着检查 B。把 A 的背景色设为透明，A 仍会挡住触摸。

下面的覆盖层让自身空白区域穿透，同时保留子按钮的触摸响应：

```swift
final class PassthroughOverlay: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let target = super.hitTest(point, with: event)
        return target === self ? nil : target
    }
}
```

点击覆盖层里的按钮时，super 返回按钮，覆盖层照样返回按钮。点击覆盖层上没有子视图的位置时，super 返回覆盖层自己，覆盖层改为返回 nil，父视图继续检查下面的视图。注意：覆盖层上不需要交互的装饰性子视图仍会被命中、挡住触摸，需要把它们的 isUserInteractionEnabled 设为 false。

若需要指定一个完全穿透的矩形区域，可以重写 point(inside:with:)：

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

CGRect.zero 是原点 (0, 0)、宽高都为 0 的空矩形，contains 对任何点都返回 false，所以初始状态下没有穿透区域。赋给 passthroughRect 的矩形使用 overlay 自己的坐标系，例如局部点 (60, 120) 就在这个矩形内。

point(inside:with:) 返回 false 时，命中测试会跳过整个覆盖层及其子视图，所以孔洞范围内覆盖层的子按钮也不会响应。如果孔洞要跟随其他视图移动，应在布局变化后重新计算矩形，例如用 overlay.convert(button.bounds, from: button) 得到按钮在 overlay 中的区域。如果孔洞内的子控件需要响应，改用前一种按命中结果判断的方案。

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

扩大的区域只有在父视图范围之内才有效。触摸点在父视图范围之外时，父视图的 point(inside:with:) 返回 false，父视图直接返回 nil，根本不会询问按钮。这时还要扩大父视图的命中区域，或者重写父视图的 hitTest 把触摸转给按钮。

clipsToBounds 只控制是否裁剪超出边界的绘制内容。设为 false 后，超出部分能显示出来，但仍然不能响应触摸。

### 同一按钮的点击与长按协调

按钮的点击用 touchUpInside 事件处理，长按用 UILongPressGestureRecognizer 识别。把长按识别器直接添加到按钮上，只有从按钮范围内开始的按压才会被识别为长按。

以下代码写在 UIViewController 子类中。button 是已经创建好的 UIButton 属性，setupInteractions() 在界面初始化时调用一次。

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

按住不到 0.5 秒就在按钮上松手，立即触发 touchUpInside。按住达到 0.5 秒、且手指移动不超过允许距离时，长按识别成功，进入 began 状态。

cancelsTouchesInView 为 true 时，长按识别成功后，按钮收到的触摸会被取消，所以这次松手不会再触发 touchUpInside。被取消的是按钮的触摸，长按识别器自己的状态不受影响。

如果按钮用的是 touchDown（按下时就触发），那么长按识别成功时，动作已经执行过了，取消触摸也撤销不了。

如果把长按识别器添加到页面的根视图上，从根视图上任何位置开始的按压都可能被识别为长按。只希望按钮响应长按时，就把识别器添加到按钮上；按钮扩大的命中区域也算在按钮范围内。

### 长按状态与取消后的恢复

UILongPressGestureRecognizer 是连续手势识别器：识别成功后，还会持续跟踪触摸的变化，并在结束或被取消时再次调用动作方法。

| 状态 | 含义 | 适合执行的操作 |
| --- | --- | --- |
| .began | 达到长按识别条件 | 保存原状态，开启临时效果 |
| .changed | 长按已经开始，触摸信息发生变化 | 按需要更新连续交互 |
| .ended | 已识别的长按正常结束，例如用户松手 | 恢复原状态，结束临时效果 |
| .cancelled | 已识别的长按被中断 | 同样恢复原状态并清理 |
| .failed | 在识别开始前未满足条件 | 不执行仅属于长按成功路径的操作 |

例如，长按开始后来电界面弹出并接管了交互，UIKit 取消当前触摸，识别器进入 cancelled。来电横幅不一定会取消触摸，只有触摸被系统实际中断时才会进入 cancelled。

长按开始后，手指移出按钮，长按并不会因此取消。如果需要“移出按钮就结束”，要在 changed 状态中自己检查手指位置。

临时效果要在 began 时保存原状态，在 ended 和 cancelled 时都恢复。只在 ended 中恢复的话，长按被取消时临时效果会残留。

## 屏幕键盘：触摸位置与文字接收对象

第一响应者是当前优先接收键盘输入等事件的响应对象。在文本输入中，成为第一响应者的输入框就是文字的输入目标；但触摸事件仍按触摸位置分发，不会都交给这个输入框。

假设页面同时显示用户名框和密码框，使用手机自带的屏幕键盘输入：

1. 用户点击密码框。UIKit 根据触摸位置找到密码框，密码框成为第一响应者，系统显示屏幕键盘。
2. 用户点击屏幕键盘上的 A 键。这次触摸有坐标，键盘利用该坐标判断用户点击了哪个键。
3. 系统把字母 A 交给当前的第一响应者，也就是密码框。A 键的位置只用来判断按的是哪个键，与文字写进哪个输入框无关。
4. 用户点击用户名框后，再按同一个 A 键，文字就输入到用户名框。键盘位置没变，变的是第一响应者。

所以，点击屏幕键盘时有触摸坐标，但这个坐标只用来确定按键；文字写到哪里，由第一响应者决定。

关键窗口决定事件交给哪个窗口，第一响应者决定交给窗口中的哪个对象。标准的文本输入控件只有在所在窗口是关键窗口、且自己成为第一响应者时，才能正常弹出屏幕键盘。

外接键盘的按键事件完全没有触摸坐标，文字同样交给第一响应者。
