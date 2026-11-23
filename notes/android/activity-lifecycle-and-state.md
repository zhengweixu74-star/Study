# Android 页面生命周期与状态保存

[返回 Android 分类目录](README.md)

Android 与 iOS 页面都要处理用户输入、数据加载、分页、错误状态和界面刷新，两端都可以把界面展示与业务逻辑分开。职责划分和数据流设计可以在两端通用；状态保存方式要按各平台的页面生命周期设计。

## Activity 的基本生命周期

生命周期是组件从创建、经历不同运行状态到销毁的过程。生命周期回调是框架在状态变化时调用的方法。Activity 的生命周期由 Android 框架驱动。

| 回调 | 含义与常见用途 |
| --- | --- |
| onCreate() | 创建新 Activity 实例时初始化页面和必要状态。 |
| onStart() | Activity 进入对用户可见的阶段。 |
| onResume() | Activity 进入前台交互阶段；处于该阶段的 Activity 不一定独占输入焦点。 |
| onPause() | Activity 离开 Resumed 状态，但界面可能仍然可见。 |
| onStop() | Activity 已不再对用户可见；Activity 实例不一定立即销毁。 |
| onDestroy() | 框架正常销毁 Activity 实例时调用；进程直接被系统结束时，不保证调用。 |

新 Activity 正常进入前台时，回调顺序通常是 `onCreate()` → `onStart()` → `onResume()`。已停止但未销毁的 Activity 回到前台时，回调顺序通常是 `onRestart()` → `onStart()` → `onResume()`。有些启动过程会在到达可交互状态之前中止，后续回调不再执行。

## Fragment 与其视图的生命周期

Fragment 实例和 Fragment 创建的 View 层级各有一套生命周期。View 层级可以先被销毁，Fragment 实例仍然保留。

新 Fragment 创建视图时，框架先调用 `onCreate()`，再调用 `onCreateView()` 创建根视图。根视图创建成功后，框架调用 `onViewCreated()`，应用在其中绑定控件、配置界面。

`onDestroyView()` 表示这一次视图生命周期结束；`onDestroy()` 表示 Fragment 实例进入销毁阶段。应用在 `onDestroyView()` 中清除指向旧视图的引用。依赖视图的观察要绑定到 `viewLifecycleOwner`，`viewLifecycleOwner` 代表当前这一次视图生命周期。

例如，用替换操作切走某个 Fragment，并把替换操作加入 FragmentManager 返回栈。这时旧 Fragment 的视图被销毁，Fragment 实例继续保留。用户返回时，框架再次调用视图创建回调。所以同一个 Fragment 实例可以多次创建视图，视图引用要在 `onDestroyView()` 中清理，只在 `onDestroy()` 中清理就太晚了。

Fragment 的生命周期由宿主和 FragmentManager 管理。页面是否可见、视图是否存在、Fragment 实例是否存在，是三个独立的状态。

[相关笔记：Activity、Fragment 与页面组织](activity-fragment-and-navigation.md)

## Activity 重建与 UIKit 旋转的区别

页面生命周期是承载页面的组件从创建、显示到销毁的过程。Activity 是 Android 中承载界面和用户交互的应用组件，一个 Activity 可以承载多个页面。本文讨论以 Activity 为作用域的页面状态。

配置变化指屏幕方向等运行配置发生改变。以下条件同时满足时，Android 默认销毁旧 Activity，再创建新的 Activity：页面允许旋转，方向变化会触发重建，应用没有自行处理这项配置变化。UIKit 旋转时通常保留原 UIViewController，只调整视图尺寸和布局。所以两端的对象存活方式要分别设计。

新 Activity 拿不到旧 Activity 普通成员属性的值。假设搜索词为“猫”，页面已有 20 条搜索结果。如果这些数据只保存在旧 Activity 的普通属性中，又没有其他对象持有或保存恢复机制，新 Activity 重建界面后，这 20 条结果就丢失了。

框架的视图状态保存机制可以恢复部分 View 自身的状态；Activity 的业务属性要由应用自己保存。搜索词、加载结果和加载状态需要保留时，要明确由哪个对象保存、在什么时机恢复。

## ViewModel 如何跨配置变化保留状态

Jetpack ViewModel 是保存页面状态、协调页面逻辑的框架组件。应用通过框架获取 ViewModel，并把 ViewModel 关联到 Activity 作用域。配置变化导致 Activity 重建时，这个 ViewModel 继续存活。

```text
旋转前：旧 Activity → SearchViewModel → 搜索词、搜索结果
旋转后：新 Activity → 同一个 SearchViewModel → 原来的数据
```

新 Activity 用相同的作用域和 key 获取 ViewModel，拿到的是旧 Activity 用过的同一个实例。新 Activity 再根据 ViewModel 中的状态创建和填充界面，所以已有搜索结果不必因为旋转而重新请求。这种保留能力来自框架的获取方式：普通类即使命名为 ViewModel，或者每次直接 new 一个新实例，都得不到同一个实例。

ViewModel 只在作用域有效期间保留。Activity 因 `finish()` 等操作永久结束时，框架清理关联的 ViewModel。ViewModel 保留的是页面状态，旧 Activity 本身仍会被销毁；ViewModel 也不是永久缓存。

## 进程结束后如何恢复页面

进程是应用执行代码、持有运行时内存的环境。系统结束进程时，内存中的 Activity 和 ViewModel 实例都会消失，再次打开页面时只能创建新的 ViewModel。

SavedStateHandle 是配合 ViewModel 使用的键值状态容器，借助系统的状态保存机制保留少量恢复信息。应用需要显式写入搜索词、条目 ID 等数据。以下条件同时满足时，新 ViewModel 可以读取这些数据，再从数据源重新加载结果：状态已经保存，任务记录仍然保留，系统按该记录恢复页面。

`保存搜索词 → 系统结束进程 → 系统恢复页面 → 新 ViewModel 读取搜索词 → 重新加载结果`

| 需求 | 适合保存的位置 | 边界 |
| --- | --- | --- |
| 配置变化期间保留已经加载的页面状态 | Jetpack ViewModel | 仅保留在进程内；作用域永久结束后会清理。 |
| 系统结束进程后恢复搜索词等少量信息 | SavedStateHandle 等保存状态机制 | 依赖系统保存和恢复记录；不保证用户主动结束页面或移除任务后仍然存在。 |
| 长期保留草稿、业务记录或其他持久数据 | 数据库或文件 | 持久化是将数据写入可跨进程存续的存储；需要由应用按业务规则读写。 |

SavedStateHandle 只保存重建界面所需的最小信息，例如搜索词和条目 ID；大量图片和完整的大型结果集不要放进去。需要长期保留的数据写入数据库或文件。

## 异步结果与旧页面的关系

异步请求可能在 Activity 重建之后才返回。如果回调直接持有旧 Activity 或旧 View，回调会更新一个已经失效的界面，还会让旧对象迟迟无法释放。

需要跨配置变化继续执行的加载任务交给 ViewModel 协调。ViewModel 把结果写入页面状态，新 Activity 按自身生命周期观察这个状态并刷新界面，旧 Activity 的观察随其生命周期结束而停止。ViewModel 不持有旧 Activity 或旧 View。

两端通用的数据流是：用户输入 → 业务层或数据层执行加载 → 状态持有者更新状态 → 当前有效的界面刷新。任务是否取消、结果是否保留，由任务所属的作用域和业务需要决定。

用 StateFlow 暴露状态、用 repeatOnLifecycle 按生命周期收集状态的完整示例见[ViewModel 状态暴露与页面收集：StateFlow 与 repeatOnLifecycle](stateflow-and-lifecycle-collection.md)。

相关笔记：[UIKit 布局、列表与 MVVM](../ios/uikit-layout-and-mvvm.md)；[Swift 基础语义与内存管理](../language/swift-basics.md#基础语义与内存管理)。
