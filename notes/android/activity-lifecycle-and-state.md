# Android 页面生命周期与状态保存

[返回 Android 分类目录](README.md)

Android 与 iOS 都需要组织用户输入、数据加载、分页、错误状态和界面刷新。两端都可以把界面展示与业务逻辑分开。可迁移的是职责划分和数据流设计；状态保存方式需要结合平台的页面生命周期。

## Activity 的基本生命周期

生命周期是组件从创建、进入不同运行状态到销毁的过程。生命周期回调是框架在这些状态变化时调用的方法。Activity 的生命周期由 Android 框架驱动。

| 回调 | 含义与常见用途 |
| --- | --- |
| onCreate() | 创建新 Activity 实例时初始化页面和必要状态。 |
| onStart() | Activity 进入对用户可见的阶段。 |
| onResume() | Activity 进入前台交互阶段；不应把该回调简单等同于获得唯一输入焦点。 |
| onPause() | Activity 离开 Resumed 状态，但界面可能仍然可见。 |
| onStop() | Activity 已不再对用户可见；Activity 实例不一定立即销毁。 |
| onDestroy() | 框架正常销毁 Activity 实例时调用；进程直接被系统结束时，不保证调用。 |

一个新 Activity 正常进入前台的常见回调顺序是 onCreate() → onStart() → onResume()。已停止但未销毁的 Activity 重新进入前台时，常见顺序是 onRestart() → onStart() → onResume()。这些顺序不表示任意启动请求都会完整执行到可交互状态。

## Fragment 与其视图的生命周期

Fragment 实例和 Fragment 创建的 View 层级具有不同的生命周期。对于创建视图的 Fragment，视图可以先销毁，而 Fragment 实例仍然保留。

在新 Fragment 创建视图的常见流程中，框架先调用 onCreate()，再调用 onCreateView() 创建根视图；根视图创建成功后，框架调用 onViewCreated()，应用可以在其中绑定控件和配置界面。

onDestroyView() 表示该次视图生命周期结束。onDestroy() 表示 Fragment 实例进入销毁阶段。应用应在 onDestroyView() 中清除已经失效的视图引用；依赖视图的观察应绑定到 viewLifecycleOwner，即该次视图生命周期的持有者。

例如，使用替换操作切走某个 Fragment，并将该操作加入 FragmentManager 返回栈时，旧 Fragment 的视图可以被销毁，而 Fragment 实例继续保留。返回时，框架可以重新调用视图创建回调。因此，不能假设同一个 Fragment 实例只创建一次视图，也不能只在 onDestroy() 中清理视图引用。

Fragment 的生命周期受宿主和 FragmentManager 管理；页面是否可见、视图是否存在以及 Fragment 实例是否存在，是需要分别判断的状态。

[相关笔记：Activity、Fragment 与页面组织](activity-fragment-and-navigation.md)

## Activity 重建与 UIKit 旋转的区别

页面生命周期是页面承载组件从创建、显示到销毁的过程。Activity 是 Android 中承载界面和用户交互的应用组件；一个 Activity 也可以承载多个页面。本文讨论以 Activity 为作用域的页面状态。

配置变化是屏幕方向等运行配置发生改变。在允许旋转、方向变化触发重建且应用未自行处理该配置变化时，Android 默认会销毁旧 Activity，再创建新的 Activity。UIKit 在通常的旋转处理中保留原 UIViewController，并调整视图尺寸和布局。两者不能按相同的对象存活方式设计。

新 Activity 不会自动继承旧 Activity 普通成员属性的值。假设搜索词为“猫”，页面已有 20 条搜索结果；如果这些数据只保存在旧 Activity 的普通属性中，且没有其他持有者或保存恢复机制，新 Activity 就不能仅靠重新创建界面得到原来的数据。

部分 View 可以通过框架的视图状态保存机制恢复自身状态，但这不等于 Activity 的任意业务属性都会自动恢复。需要保留的搜索词、加载结果和加载状态，应明确由谁保存、在何时恢复。

## ViewModel 如何跨配置变化保留状态

Jetpack ViewModel 是保存页面状态并协调相关逻辑的框架组件。通过框架获取并关联到 Activity 作用域的 ViewModel，可以在配置变化引起的 Activity 重建期间继续存活。

`旋转前：旧 Activity → SearchViewModel → 搜索词、搜索结果`

`旋转后：新 Activity → 同一个 SearchViewModel → 原来的数据`

新 Activity 按相同作用域和 key 获取 ViewModel，再根据 ViewModel 中的状态重新创建和填充界面。因此，已有搜索结果无须仅因旋转而重复请求。把普通类命名为 ViewModel，或者每次直接创建一个新实例，都不会自动获得这种保留能力。

用 Swift 类比：一个存活时间更长的对象持有 SearchViewModel。旧 UIViewController 释放后，新 UIViewController 继续使用同一个 SearchViewModel。这个类比解释的是持有关系；UIKit 不会因为类名包含 ViewModel 就自动提供 Jetpack 的作用域管理。

这里的保留只适用于对应作用域仍然有效的情况。Activity 因 finish() 等操作永久结束时，关联的 ViewModel 会被清理。ViewModel 不是永久缓存，也不会保留旧 Activity 本身。

## 进程结束后如何恢复页面

进程是应用执行代码并持有运行时内存的环境。进程被系统结束时，内存中的 Activity 和 ViewModel 实例都会消失；再次打开页面时，应用不能取回原来的 ViewModel 对象。

SavedStateHandle 是与 ViewModel 配合使用的键值状态容器，能够通过系统的状态保存机制保留少量恢复信息。应用需要显式写入搜索词、条目 ID 等数据。在状态已保存、任务记录仍保留且系统按该记录恢复页面时，新 ViewModel 可以读取这些数据，再从数据源重新加载结果。

`保存搜索词 → 系统结束进程 → 系统恢复页面 → 新 ViewModel 读取搜索词 → 重新加载结果`

| 需求 | 适合保存的位置 | 边界 |
| --- | --- | --- |
| 配置变化期间保留已经加载的页面状态 | Jetpack ViewModel | 仅保留在进程内；作用域永久结束后会清理。 |
| 系统结束进程后恢复搜索词等少量信息 | SavedStateHandle 等保存状态机制 | 依赖系统保存和恢复记录；不保证用户主动结束页面或移除任务后仍然存在。 |
| 长期保留草稿、业务记录或其他持久数据 | 数据库或文件 | 持久化是将数据写入可跨进程存续的存储；需要由应用按业务规则读写。 |

SavedStateHandle 应保存重建界面所需的最小信息，不应装入大量图片或完整的大型结果集。需要长期保留的数据不能只放在 ViewModel 或保存状态中。

## 异步结果与旧页面的关系

异步请求可能在 Activity 重建后才返回。如果回调直接持有旧 Activity 或旧 View，回调可能更新已经失效的界面，并延长旧对象的存活时间。

需要跨配置变化继续执行的加载工作可以由 ViewModel 协调，并将结果写入页面状态。新 Activity 按自身生命周期观察状态并刷新界面；旧 Activity 的观察应随生命周期停止或解除。ViewModel 应避免持有旧 Activity 或旧 View。

两端共同的数据流可以表达为：用户输入 → 业务或数据层执行加载 → 状态持有者更新状态 → 当前有效界面刷新。具体任务是否取消、结果是否保留，应由任务所属作用域和业务需要决定。

相关笔记：[UIKit 布局、列表与 MVVM](../ios/uikit-layout-and-mvvm.md)；[Swift 基础语义与内存管理](../language/swift-basics.md#基础语义与内存管理)。
