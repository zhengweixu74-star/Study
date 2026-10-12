# Activity、Fragment 与页面组织

[返回 Android 分类目录](README.md)

Activity 是 Android 系统可以启动和管理的应用组件。对于提供界面的常规 Activity，Activity 提供窗口并承载用户交互。Fragment 是依附于宿主、管理一部分界面及相关逻辑的组件。本文讨论 AndroidX Fragment 和传统 View 体系。

## Activity、Fragment 与 View 的职责

View 是实际参与显示、布局和输入事件的视图对象。Activity 和 Fragment 都不是 View；Activity 和 Fragment 可以创建、持有并管理由多个 View 组成的视图层级。

| 对象 | 主要职责 | 与其他对象的关系 |
| --- | --- | --- |
| Activity | 作为系统可启动的界面入口，提供窗口并接收系统生命周期回调。 | 可以直接管理 View，也可以承载一个或多个 Fragment。 |
| Fragment | 管理一部分界面的视图、交互逻辑和自身生命周期。 | 由 Activity 或父 Fragment 承载，最终依附于 Activity。 |
| View / ViewGroup | 显示内容、参与布局并处理输入；ViewGroup 还管理子视图。 | 组成 Activity 或 Fragment 所管理的实际界面。 |

宿主是承载 Fragment 并为 Fragment 提供运行环境的 Activity 或父 Fragment。Fragment 不能像 Activity 一样由系统独立启动。一个 Fragment 可以表示整页内容，也可以表示页面中的一个区域。

## 界面、页面和窗口的范围

界面是用户可见、可交互内容的统称。页面通常指产品中一个相对完整的操作单元。这两个词不限定对应一个 Activity、一个 Fragment，还是一组 View；描述组件关系时，应直接写明具体对象。

窗口是系统管理界面显示区域及其输入交互的单位。窗口不等于设备的物理屏幕。常规 Activity 拥有自己的 Window；应用可以包含多个 Activity，也不要求所有 Activity 同时可见。

“Activity 与系统对接”表示系统以 Activity 为单位管理启动、显示状态等行为，不表示整个应用只能有一个 Activity。应用采用多少个 Activity，由页面组织和系统交互需求决定。

## 页面如何组合与切换

Activity 与 Fragment 的职责划分，使系统管理窗口和应用组织页面可以分别进行。Activity 提供宿主环境，Fragment 划分页面逻辑；多个 Fragment 可以切换显示，也可以同时组成一个界面。

| 组织方式 | 最小示例 | 说明 |
| --- | --- | --- |
| Activity 直接管理 View | LoginActivity → 输入框和按钮 | 简单界面可以不使用 Fragment。 |
| 一个 Activity 承载多个 Fragment | MainActivity → HomeFragment / LoginFragment | 页面切换可以发生在同一个 Activity 内。 |
| 多个 Activity 分别承载界面 | MainActivity、LoginActivity | 每个 Activity 可以直接管理 View，也可以承载 Fragment。 |

应用已经有 MainActivity 时，可以在 MainActivity 中显示 LoginFragment，不需要为登录页再创建 LoginActivity。只有 Fragment 而没有 Activity 宿主，则不能独立显示这类 Fragment 界面。

FragmentManager 是管理 Fragment 添加、移除、替换、状态恢复和返回栈的对象。返回栈记录需要在返回操作中撤销的页面变更。Fragment 切换不必启动新的 Activity；嵌套的 Fragment 由父 Fragment 的子 FragmentManager 管理。

例如，邮件列表和邮件详情可以分别由两个 Fragment 管理。手机可以在两个 Fragment 之间切换；平板可以在同一个 Activity 中同时展示两个 Fragment。共享的是页面职责和代码，实际布局仍需按可用空间配置。

## AndroidManifest.xml 为什么要声明 Activity

AndroidManifest.xml 是应用向系统声明组件及相关配置的清单文件。应用中使用的 Activity 必须在清单中声明；Fragment 不需要单独在清单中声明。

系统根据清单确定目标 Activity 是否存在、如何启动，以及调用方是否有权启动。声明 Activity 为系统提供可定位的组件入口；仅编写一个 Activity 类，并不会自动完成这项注册。

声明 Activity 不等于允许所有其他应用启动该 Activity。对外访问还受到 android:exported、权限等配置限制。声明 Activity 也不等于创建桌面图标；常规启动入口还需要相应的 MAIN 和 LAUNCHER 配置。

## 启动 Activity 的作用与流程

对于提供界面的常规 Activity，启动 Activity 的作用是请求系统进入该 Activity 所代表的交互界面。启动请求可以来自应用内部，也可以来自桌面入口、通知点击等系统交互。

Intent 是描述目标组件或希望执行动作、并可携带数据的消息对象。应用可以向 startActivity() 传入 Intent，请求系统启动目标 Activity。

1. 调用方发出启动请求。系统根据 Intent 查找目标 Activity，并检查相关声明、权限和启动条件。
2. 系统根据启动配置及当前状态创建 Activity 实例，或者复用已有实例。目标进程尚未运行时，系统需要先创建进程。
3. 在创建新实例的常见流程中，系统调用 onCreate()。应用在回调中初始化页面，并设置 View 内容或配置 Fragment 宿主。
4. 系统继续推进 Activity 的生命周期并安排窗口显示。Activity 进入可交互状态后，用户可以操作界面。

启动 Activity 不等于每次都启动新的应用进程，也不等于每次都创建新的 Activity 实例。已有 Activity 内的 Fragment 切换通常只是该 Activity 内部的页面变更。

## 与 UIViewController 的类比边界

UIViewController 是 UIKit 中管理视图层级及相关交互逻辑的控制器。就管理整页或局部界面、组合子页面这些职责而言，Fragment 与 UIViewController 更接近，尤其是在一个 Activity 承载多个 Fragment 的结构中。

Activity 也能直接管理整页界面，但 Activity 还承担 Android 系统组件入口和窗口相关职责。因此，Activity、Fragment 与 UIViewController 之间不存在完整的一一对应关系。不能直接套用两端的生命周期或对象存活规则。

[相关笔记：Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)

[相关笔记：Android 页面生命周期与状态保存](activity-lifecycle-and-state.md)
