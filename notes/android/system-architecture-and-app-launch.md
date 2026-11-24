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

ART（Android Runtime）是 Android 上执行 App 代码、管理 App 对象内存的虚拟机。虚拟机是用软件实现的程序执行环境，负责加载代码、执行代码并管理内存。每个 App 进程中都运行着一份 ART。

### 字节码与机器码

机器码是某种 CPU 能直接执行的二进制指令。每种 CPU 架构有自己的指令集，指令集是这种 CPU 能识别的全部指令及其编码格式。ARM64 的机器码只能在 ARM64 CPU 上执行。

字节码是格式由虚拟机规定、不针对某种 CPU 的指令。CPU 不能直接执行字节码，要由虚拟机把字节码转换成本机 CPU 的机器码，或者由虚拟机逐条解释执行。DEX 是 ART 能执行的字节码文件格式。App 代码的编译链路是：

```text
Kotlin／Java 源码 → .class 字节码 → .dex 文件 → 打包进 APK
```

以一个 Kotlin 函数为例：

```kotlin
fun add(a: Int, b: Int): Int = a + b
```

编译成 DEX 字节码后，用文字表示类似下面的指令（编译器实际生成的寄存器编号可能不同）：

```text
add-int v0, p0, p1    // 把参数寄存器 p0 和 p1 相加，结果放进 v0
return v0             // 返回 v0
```

ART 在 ARM64 设备上把这段字节码编译成机器码，用汇编表示是：

```text
add w0, w0, w1        // 把 CPU 寄存器 w0 和 w1 相加，结果存回 w0
ret                   // 返回
```

汇编是机器码的文字写法。`add w0, w0, w1` 的机器码是 4 字节的 `0x0B010000`，CPU 读取这 4 个字节就执行加法。

两段代码中的寄存器含义不同。DEX 中的 `v0`、`p0`、`p1` 是 DEX 格式规定的虚拟寄存器，只是编号；ARM64 机器码中的 `w0`、`w1` 是 CPU 中真实存在的硬件存储单元。

Android 使用字节码，原因是 Android 设备的 CPU 架构不止一种，例如 ARM64、32 位 ARM，模拟器还使用 x86-64。App 只提供一份 DEX，每台设备上的 ART 把 DEX 转换成本机 CPU 的机器码。iOS 设备都使用 ARM64 架构，Xcode 直接把 Swift 编译成 ARM64 机器码，iOS 上的 App 运行时没有字节码虚拟机这一层。

Android App 中也有直接编译成机器码的代码。NDK 是 Android 提供的 C/C++ 开发工具集，C/C++ 代码经 NDK 编译成 `.so` 动态库。`.so` 中保存的是机器码，APK 的 `lib/` 目录按 CPU 架构分别存放各架构的 `.so`，见[安装包：APK、AAB 与 R8](project-structure-and-build.md#安装包apkaab-与-r8)。

### AOT 与 JIT

ART 把 DEX 编译成机器码的时机有两种：

- AOT（提前编译）：在安装时或设备空闲时，把字节码提前编译成机器码。
- JIT（即时编译）：在 App 运行时，把执行频繁的代码编译成机器码。

ART 结合使用这两种方式。

### ART 管理对象内存

App 代码创建对象时，ART 在堆上为对象分配内存。堆是进程中存放对象的内存区域；变量中保存的是指向堆上对象的引用。

ART 中的垃圾回收器（GC）找出不再可达的对象，回收这些对象占用的内存，App 代码不需要编写释放对象的代码。GC 判断对象是否可达的方式，以及 GC 与 Swift ARC 的区别，见[ARC 与 Android GC](../language/swift-memory-and-references.md#arc-与-android-gc引用计数可达性与引用环)。

GC 主要在 App 分配新对象、堆的用量达到 ART 设定的阈值时触发；App 退到后台等时机，ART 也会安排一次回收。GC 的大部分工作在 ART 的后台线程上执行。

### ART、Dalvik 与 JVM

JVM（Java 虚拟机）是在电脑和服务器上运行 Java 程序的虚拟机，执行 `.class` 字节码。Android 使用 ART 执行 DEX 字节码；Android 5.0 之前，Android 使用的虚拟机是 Dalvik。

JVM 与 ART 都从 GC Roots 出发判断对象是否可达，所以 Java 资料中关于 GC Roots 和可达性的知识同样适用于 Android。

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
