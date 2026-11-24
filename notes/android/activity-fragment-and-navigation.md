# Activity、Fragment 与页面组织

[返回 Android 分类目录](README.md)

Activity 是 Android 系统可以启动和管理的应用组件。提供界面的常规 Activity 持有一个窗口，用户在窗口中与界面交互。Fragment 依附于宿主，管理界面中的一部分及其交互逻辑。本文讨论 AndroidX Fragment 和传统 View 体系。

## Activity、Fragment 与 View 的职责

View 是实际参与显示、布局和输入事件的视图对象。Activity 和 Fragment 都不是 View。Activity 和 Fragment 创建、持有并管理由多个 View 组成的视图层级。

| 对象 | 主要职责 | 与其他对象的关系 |
| --- | --- | --- |
| Activity | 作为系统可启动的界面入口，提供窗口并接收系统生命周期回调。 | 可以直接管理 View，也可以承载一个或多个 Fragment。 |
| Fragment | 管理一部分界面的视图、交互逻辑和自身生命周期。 | 由 Activity 或父 Fragment 承载，最终依附于 Activity。 |
| View / ViewGroup | 显示内容、参与布局并处理输入；ViewGroup 还管理子视图。 | 组成 Activity 或 Fragment 所管理的实际界面。 |

宿主是承载 Fragment 的 Activity 或父 Fragment，为 Fragment 提供运行环境。系统只能启动 Activity，Fragment 必须放在宿主中显示。一个 Fragment 可以表示整页内容，也可以表示页面中的一个区域。

## 界面、页面和窗口的范围

界面泛指用户可见、可交互的内容。页面通常指产品中一个相对完整的操作单元。一个界面或页面可以由一个 Activity、一个 Fragment 或一组 View 实现，因此描述组件关系时直接写出具体对象。

窗口是系统管理显示区域和输入交互的单位，与设备的物理屏幕是两个概念。每个常规 Activity 拥有自己的 Window。一个应用可以包含多个 Activity，这些 Activity 不必同时可见。

系统以 Activity 为单位管理启动和显示状态。一个应用可以有多个 Activity，具体数量由页面组织方式和系统交互需求决定。

## Activity、Window 与 DecorView

Activity 是 `Context` 的间接子类，不继承自 View。`Context` 是访问资源、获取系统服务和启动组件的入口对象，见[Context 与 Application](app-components-and-context.md#context-与-application)。常规 Activity 通过 Window 显示界面，View 树挂在 Window 上：

```text
MainActivity（Activity，不是 View）
└── Window（Activity 持有的窗口对象，不是 View）
    └── DecorView（整棵 View 树的根视图，是 ViewGroup）
        └── 内容区域（id 为 android.R.id.content 的 ViewGroup）
            └── LinearLayout（activity_main.xml 的根视图）
                ├── TextView（id 为 title）
                └── Button（id 为 refresh）
```

- Window 承载 Activity 的整棵 View 树。
- DecorView 是 Window 中最外层的 View，其他 View 都位于 DecorView 之下。
- `setContentView(R.layout.activity_main)` 加载布局并创建 View 对象，再把布局的根视图放入内容区域。

UIViewController 也不是 UIView，UIViewController 通过 `view` 属性持有根视图。Activity 与根视图之间多了一层 Window，所以 `Activity.findViewById()` 通过 Window 在整棵 View 树中查找，见[从资源 ID 获取内容](resources-and-r-class.md#从资源-id-获取内容)。

## 页面如何组合与切换

系统负责管理 Activity 的窗口，应用负责用 Fragment 组织页面，两件事互相独立。Activity 提供宿主环境，Fragment 划分页面逻辑。多个 Fragment 可以轮流显示，也可以同时组成一个界面。

| 组织方式 | 最小示例 | 说明 |
| --- | --- | --- |
| Activity 直接管理 View | LoginActivity → 输入框和按钮 | 简单界面可以不使用 Fragment。 |
| 一个 Activity 承载多个 Fragment | MainActivity → HomeFragment / LoginFragment | 页面切换可以发生在同一个 Activity 内。 |
| 多个 Activity 分别承载界面 | MainActivity、LoginActivity | 每个 Activity 可以直接管理 View，也可以承载 Fragment。 |

应用已有 MainActivity 时，可以直接在 MainActivity 中显示 LoginFragment，无需为登录页再创建 LoginActivity。LoginFragment 必须放进 MainActivity 这样的 Activity 宿主才能显示。

FragmentManager 负责添加、移除和替换 Fragment，并负责 Fragment 的状态恢复和返回栈。返回栈记录用户按返回键时需要撤销的页面变更。切换 Fragment 时无需启动新的 Activity。嵌套的 Fragment 由父 Fragment 的子 FragmentManager 管理。

例如，邮件列表和邮件详情分别由两个 Fragment 管理。手机上，MainActivity 在两个 Fragment 之间切换；平板上，同一个 Activity 同时展示两个 Fragment。两种设备复用的是 Fragment 的页面职责和代码，具体布局仍要按屏幕可用空间分别配置。

## AndroidManifest.xml 为什么要声明 Activity

AndroidManifest.xml 是应用向系统声明组件和配置的清单文件，内容包括应用名称和图标、组件、启动入口和所需权限。系统安装应用时解析清单，之后启动组件时使用解析结果。应用用到的每个 Activity 都必须在清单中声明；Fragment 不需要在清单中声明。

系统根据清单判断三件事：目标 Activity 是否存在、如何启动、调用方是否有权启动。只编写 Activity 类，系统找不到这个组件；在清单中声明后，系统才能定位并启动这个 Activity。

其他应用能否启动该 Activity，由 `android:exported` 和权限配置决定。Activity 要在桌面上显示图标，还需要配置 MAIN 动作和 LAUNCHER 类别。

## 启动 Activity 的作用与流程

启动常规 Activity，就是请求系统显示该 Activity 的交互界面。启动请求可以来自应用内部，也可以来自桌面图标、通知点击等系统交互。

Intent 是一种消息对象，描述目标组件或希望执行的动作，并可以携带数据。应用把 Intent 传给 `startActivity()`，请求系统启动目标 Activity。

1. 调用方发出启动请求。系统根据 Intent 查找目标 Activity，并检查清单声明、权限和启动条件。
2. 系统根据启动配置和当前状态，创建新的 Activity 实例或复用已有实例，规则见下文[任务栈与启动模式](#任务栈与启动模式)。目标进程还没有运行时，系统先创建进程，冷启动流程见[Android 系统分层与应用启动](system-architecture-and-app-launch.md)。
3. 创建新实例时，系统调用 `onCreate()`。应用在 `onCreate()` 中初始化页面，设置 View 内容或配置 Fragment 宿主。
4. 系统继续推进 Activity 的生命周期并安排窗口显示。Activity 进入可交互状态后，用户可以操作界面。

目标进程已经运行时，启动 Activity 不会新建进程；启动配置允许复用时，系统也不会新建 Activity 实例。在已有 Activity 内切换 Fragment，只改变该 Activity 内部的页面，不经过上面的启动流程。

## 显式 Intent、隐式 Intent 与页面参数

Intent 按是否指定目标组件分为两种：

| 类型 | 写法 | 系统如何找到目标 |
| --- | --- | --- |
| 显式 Intent | `Intent(this, DetailActivity::class.java)` | Intent 直接指定目标类 |
| 隐式 Intent | `Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))` | Intent 只描述动作和数据。系统在已安装 App 的清单中查找 `<intent-filter>` 声明能处理这个动作的组件；有多个候选时，系统让用户选择 |

`DetailActivity::class.java` 得到 `DetailActivity` 类对应的 Java `Class` 对象，Intent 用 `Class` 对象标识目标 Activity。

启动页面时，参数以键值对的形式放进 Intent，键值对叫 extra：

```kotlin
// 在 MainActivity 中
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("note_id", note.id)
startActivity(intent)
```

```kotlin
// 在 DetailActivity 的 onCreate() 中
val noteId = intent.getLongExtra("note_id", -1L)   // 取启动 DetailActivity 的 Intent 中的参数
```

UIKit 中，调用方可以写 `DetailViewController(noteId: id)`，自己创建页面对象并通过构造函数传参。Android 的 Activity 对象只能由系统创建，所以参数只能通过 Intent 交给系统。系统销毁 Activity 后重新创建时，会把当初的 Intent 再次交给新的 Activity，放在 Intent 中的参数因此不会丢失。

extra 只能存放系统能够序列化保存的类型，例如数字、字符串和实现了 `Parcelable` 接口的对象。`Parcelable` 是 Android 的对象序列化接口，对象按接口规定把字段写入字节数据，系统可以保存和跨进程传递这些字节数据。

## 任务栈与启动模式

任务栈（Task）是用户为完成一件事依次打开的一组 Activity，系统按栈的顺序保存这些 Activity。新启动的 Activity 压入栈顶；用户按返回键时，系统结束栈顶的 Activity，下面的 Activity 重新显示。

Task 的行为类似 `UINavigationController` 的导航栈。区别是 Task 由系统管理，返回操作由系统处理，一个 Task 中还可以包含不同 App 的 Activity。例如笔记 App 调起系统相机拍照时，相机的 Activity 可以压入笔记 App 的 Task。

启动模式（`launchMode`）在清单的 `<activity>` 中声明，决定启动 Activity 时系统新建实例还是复用已有实例：

| 启动模式 | 行为 |
| --- | --- |
| `standard`（默认） | 每次启动都新建实例并压入栈顶。连续启动 A 三次，栈中有 3 个 A。 |
| `singleTop` | 目标 Activity 已在栈顶时不新建实例，系统调用栈顶实例的 `onNewIntent()` 传入新 Intent；不在栈顶时照常新建。 |
| `singleTask` | 栈中已有目标 Activity 的实例时，系统结束这个实例上方的所有 Activity，复用这个实例并调用 `onNewIntent()`。首页常用这种模式。 |
| `singleInstance` | 目标 Activity 独占一个 Task，这个 Task 中不放其他 Activity。 |

例如栈中依次是首页、列表页、详情页，首页的启动模式是 `singleTask`。在详情页启动首页时，系统结束详情页和列表页，栈中只剩首页，首页的 `onNewIntent()` 收到新的 Intent。

## 与 UIViewController 的类比边界

UIViewController 是 UIKit 中管理视图层级和交互逻辑的控制器。在管理整页或局部界面、组合子页面这些职责上，Fragment 更接近 UIViewController，一个 Activity 承载多个 Fragment 的结构尤其如此。

Activity 也能直接管理整页界面，但 Activity 还是 Android 的系统组件入口，并持有窗口。所以 Activity、Fragment 与 UIViewController 之间没有完整的一一对应关系，两端的生命周期和对象存活规则需要分别理解。

[相关笔记：Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)

[相关笔记：Android 页面生命周期与状态保存](activity-lifecycle-and-state.md)
