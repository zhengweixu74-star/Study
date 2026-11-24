# Android 应用组件、Context 与权限

[返回 Android 分类目录](README.md)

本文说明 Android 的四种应用组件、Service 与 WorkManager 的后台任务、Context 与 Application，以及清单权限和运行时权限。Activity 的职责和启动流程见[Activity、Fragment 与页面组织](activity-fragment-and-navigation.md)。

## 四种应用组件

应用组件是系统可以直接创建、启动和销毁的入口类。每个组件都要在清单文件 `AndroidManifest.xml` 中声明，系统读取声明后才能找到并启动组件。

一个 Android App 可以有多个入口：桌面图标启动一个 Activity，系统广播拉起一个 BroadcastReceiver，其他 App 查询一个 ContentProvider。iOS App 的入口集中在 AppDelegate／SceneDelegate 和各个 App Extension 上，没有与 Android 组件一一对应的概念。

| 组件 | 作用 | 例子 |
| --- | --- | --- |
| Activity | 提供一个带界面的页面 | 列表页、详情页 |
| Service | 在没有界面的情况下持续执行任务 | 播放音乐、上传大文件 |
| BroadcastReceiver | 接收系统或其他 App 发出的广播消息 | 接收网络状态变化、开机完成的广播 |
| ContentProvider | 通过统一的 URI 和增删改查接口，向其他 App 共享数据 | 系统的通讯录、媒体库 |

## Service

Service 是没有界面、在后台执行任务的组件。

### Service 的代码运行在主线程

系统在主线程上调用 Service 的 `onCreate()`、`onStartCommand()` 等生命周期方法，Service 不会自动创建新线程。在 `onStartCommand()` 中直接下载大文件，主线程同样会被阻塞，界面会卡住。耗时工作要放到子线程或协程中执行。主线程的规则见[Android 主线程、Handler 与 ANR](main-thread-and-handler.md)。

### 启动方式：startService 与 bindService

| 方式 | 行为 | 结束时机 |
| --- | --- | --- |
| `startService()` | 启动后 Service 独立运行，调用方不持有 Service | 调用 `stopService()`，或 Service 调用 `stopSelf()` |
| `bindService()` | 调用方与 Service 建立连接，系统在 `ServiceConnection.onServiceConnected()` 中把 Service 的 `onBind()` 返回的接口对象交给调用方，调用方通过这个接口直接调用 Service 的方法 | 所有调用方都调用 `unbindService()` 断开连接后，系统销毁 Service |

一个 Service 可以同时被启动和被绑定。这种情况下，Service 要等两种结束条件都满足后才被销毁。

### 前台 Service、后台限制与 WorkManager

App 退到后台后，进程仍可以继续运行，但内存不足时系统会优先结束后台进程。iOS App 退到后台后通常很快被挂起，这一点与 Android 不同。

前台 Service 是告诉系统“用户正在关心这项任务”的 Service。前台 Service 必须显示一条通知，系统因此不会轻易结束前台 Service 所在的进程。音乐播放、导航这类用户能感知的长任务使用前台 Service。

Android 8.0（API level 26）起，系统限制后台 App 启动普通 Service。可以延后执行、但必须完成的任务，交给 Jetpack 的 WorkManager。WorkManager 把任务持久化保存，App 被结束或设备重启后，WorkManager 仍会在条件满足时执行任务，例如在连接 Wi-Fi 时同步数据。WorkManager 的用途接近 iOS 的 `BGTaskScheduler`。

## BroadcastReceiver 与 ContentProvider

### BroadcastReceiver

广播是系统或 App 发出、可以被多个接收方收到的消息。BroadcastReceiver 是接收广播并在 `onReceive()` 中执行代码的组件。广播的作用接近 iOS 的 `NotificationCenter`，区别是广播可以跨 App 发送，系统事件也以广播的形式通知 App。

| 注册方式 | 写法 | 特点 |
| --- | --- | --- |
| 静态注册 | 在清单中声明 `<receiver>` | App 没有运行时，系统也能拉起 App 接收广播。Android 8.0 起，大部分隐式广播已不能通过静态注册接收。 |
| 动态注册 | 在代码中调用 `registerReceiver()` | 只在注册期间接收广播。页面销毁时要调用 `unregisterReceiver()` 注销，否则系统一直持有 BroadcastReceiver，造成内存泄漏。 |

隐式广播是没有指定接收方 App 的广播，例如“网络状态变化”。

### ContentProvider

ContentProvider 是向其他 App 共享结构化数据的组件。其他 App 通过 `content://` 开头的 URI 定位数据，通过 `ContentResolver` 调用 `query()`、`insert()`、`update()`、`delete()` 读写数据。读取系统通讯录、媒体库时，App 访问的就是系统提供的 ContentProvider。

App 进程启动时，系统会在调用 `Application.onCreate()` 之前创建本 App 在清单中声明的 ContentProvider，见[冷启动：从点击图标到首帧](system-architecture-and-app-launch.md#冷启动从点击图标到首帧)。

## Context 与 Application

### Context

`Context` 是 App 访问资源、获取系统服务和启动组件的入口对象：

| 用途 | 示例 |
| --- | --- |
| 读取资源 | `context.getString(R.string.app_name)` |
| 启动组件 | `context.startActivity(intent)` |
| 获取系统服务 | `context.getSystemService(LayoutInflater::class.java)` |
| 访问私有目录 | `context.filesDir`，见[Android 数据存储](../network-and-storage/android-data-storage.md) |

`Activity`、`Service` 和 `Application` 都是 `Context` 的子类。不同 Context 对象的存活时间不同：

| Context 对象 | 存活时间 |
| --- | --- |
| Activity | 与页面相同。页面销毁后，Activity 应该被回收。 |
| Application（`applicationContext`） | 与进程相同 |

活得比页面久的对象需要保存 Context 时，保存 `applicationContext`。单例保存 Activity 会让已关闭的页面无法回收，见[Android 内存泄漏与 OOM](memory-leaks-and-oom.md)。

### Application

`Application` 是每个 App 进程只有一个的对象，在进程启动后最先创建。App 在清单的 `<application android:name=".NotesApp">` 中指定自定义的 `Application` 子类，并在子类的 `onCreate()` 中做全局初始化，例如初始化日志库和网络库。`Application.onCreate()` 的位置接近 iOS 的 `application(_:didFinishLaunchingWithOptions:)`。

## 清单权限与运行时权限

权限是 App 使用受保护的系统能力前必须获得的授权。Android 按风险把权限分为两类：

| 类型 | 例子 | 获得方式 |
| --- | --- | --- |
| 普通权限 | 访问网络 `INTERNET` | 在清单中声明 `<uses-permission>`，安装时自动获得 |
| 危险权限 | 相机、定位、通讯录 | 在清单中声明，并且从 Android 6.0（API level 23）起，还要在运行时调用系统 API 弹窗，由用户选择是否允许 |

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
```

iOS 的相近流程是：先在 `Info.plist` 中写明权限的使用说明，再调用授权 API 让系统弹窗。
