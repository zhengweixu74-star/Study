# Android 系统分层与应用启动

[返回 Android 分类目录](README.md)

本文说明 Android 系统由哪几层组成、App 代码如何被执行、App 进程之间如何隔离和通信，以及用户点击图标后 App 如何启动到首帧。

## 系统分层

Android 系统从下到上分为四层。上层调用下层提供的能力。

```text
App（开发者编写的应用）
  ↓ 调用
Framework（系统用 Java／Kotlin 提供的 API，例如 Activity、View）
  ↓ 依赖
ART 虚拟机 + 原生 C/C++ 库（图形、媒体、SQLite 等）
  ↓ 运行在
Linux 内核（进程、内存、驱动、Binder）
```

Framework 是 Android 提供给 App 调用的系统 API 集合。App 使用的 Activity、View、Intent 都属于 Framework。

## ART 与 DEX：App 代码如何被执行

ART（Android Runtime）是 Android 上执行 App 代码的虚拟机。虚拟机是用软件实现的程序执行环境，负责加载代码、执行代码并管理内存。

DEX 是 ART 能执行的字节码文件格式。字节码是介于源码和机器码之间的中间格式，需要虚拟机翻译后才能在 CPU 上运行。App 代码的编译链路是：

```text
Kotlin／Java 源码 → .class 字节码 → .dex 文件 → 打包进 APK
```

ART 把 DEX 编译成机器码的时机有两种：

- AOT（提前编译）：在安装时或设备空闲时，把字节码提前编译成机器码。
- JIT（即时编译）：在 App 运行时，把执行频繁的代码编译成机器码。

ART 结合使用这两种方式。iOS 的做法是 Xcode 直接把 Swift 编译成机器码，iOS 上的 App 运行时没有这一层字节码虚拟机。

## 进程、沙盒与 Binder

### 每个 App 一个进程和一个 UID

系统为每个 App 创建独立的 Linux 进程。进程是操作系统分配内存和 CPU 时间的基本单位，不同进程的内存互相隔离。

系统还为每个 App 分配一个独立的 Linux 用户 ID（UID）。Linux 按 UID 判断文件的访问权限，所以一个 App 只能读写自己 UID 拥有的文件。进程隔离和 UID 隔离合起来，构成 Android 的应用沙盒。

### Zygote 与 system_server

Zygote 是系统开机时创建的进程，Zygote 进程中已经预加载了 Framework 的类和资源。系统创建新 App 进程时，从 Zygote 复制（fork）出一个子进程。子进程直接继承已加载的类和资源，省去重复加载，所以 App 进程创建得更快。

system_server 是运行系统服务的进程。系统服务是管理全局资源、供所有 App 调用的服务对象：

| 系统服务 | 职责 |
| --- | --- |
| AMS（ActivityManagerService）／ATMS（ActivityTaskManagerService） | 管理进程、Activity 和任务栈 |
| WMS（WindowManagerService） | 管理窗口的显示和层级 |
| PMS（PackageManagerService） | 管理已安装的应用，解析清单文件 |

### Binder 跨进程通信

跨进程通信（IPC）是一个进程调用另一个进程里的功能，并传递数据。Binder 是 Android 的跨进程通信机制。App 调用 `startActivity()` 时，Framework 通过 Binder 把启动请求发给 system_server 里的 ATMS。

Binder 有两个特点：

- 一次调用只拷贝一次数据。内核把接收方进程的一块内存同时映射到内核空间，发送方的数据从用户空间拷贝到这块内存一次，接收方就能直接读到。
- 内核在每次调用中附上调用方的 UID 和 PID（进程号）。接收方根据内核提供的身份做权限校验，调用方无法伪造身份。

iOS 上相近的机制是 XPC 和 Mach 消息，见[iOS 进程间通信与应用扩展](../ios/interprocess-communication-and-extensions.md)。

## 冷启动：从点击图标到首帧

冷启动是 App 进程不存在时的启动。用户点击桌面图标后，执行顺序如下：

1. 桌面 App（Launcher）通过 Binder 请求 ATMS 启动目标 Activity。目标 Activity 是清单中声明了 `MAIN` 动作和 `LAUNCHER` 类别的 Activity。
2. ATMS 发现目标进程不存在，请求 Zygote 复制出新进程。
3. 新进程执行 `ActivityThread.main()`。`ActivityThread.main()` 为主线程创建 Looper，再通过 Binder 向 AMS 登记新进程。Looper 的作用见[Android 主线程、Handler 与 ANR](main-thread-and-handler.md)。
4. AMS 通知 App 进程初始化应用。App 进程创建 `Application` 对象，创建清单中声明的 ContentProvider，然后调用 `Application.onCreate()`。
5. AMS 通知 App 进程启动 Activity。App 进程创建 Activity 对象，依次调用 `onCreate()`、`onStart()`、`onResume()`。
6. Activity 的 View 树完成第一次测量、布局和绘制，屏幕显示首帧。

`Application` 是每个进程只有一个的对象，在进程中最先被创建，适合放全局初始化。`Application.onCreate()` 的位置接近 iOS 的 `application(_:didFinishLaunchingWithOptions:)`。`Application` 和 ContentProvider 的职责见[Android 应用组件、Context 与权限](app-components-and-context.md)。

### 系统负责创建 Activity

Android App 中没有开发者编写的 `main()` 入口。系统读取清单找到入口 Activity，然后由系统创建 Activity 对象并调用生命周期方法。开发者编写 Activity 类，但不在代码中调用 Activity 的构造函数。

系统还可能销毁 Activity 后重新创建，例如屏幕旋转或进程被回收后用户返回页面。所以 Activity 的启动参数放在 Intent 中，由系统在重建时重新交给新的 Activity，见[Activity、Fragment 与页面组织](activity-fragment-and-navigation.md)；页面状态的保存与恢复见[Android 页面生命周期与状态保存](activity-lifecycle-and-state.md)。

启动阶段的耗时集中在第 4 步和第 5 步。`Application.onCreate()` 和首个 Activity 的 `onCreate()` 中的同步工作越多，首帧出现得越晚。

[相关笔记：iOS 应用生命周期、启动与窗口](../ios/app-lifecycle-and-events.md)
