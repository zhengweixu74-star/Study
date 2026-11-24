# Android 内存泄漏与 OOM

[返回 Android 分类目录](README.md)

本文说明 Android 内存泄漏的常见引用链、修正方式与检测工具，以及 OOM 和 Bitmap 内存占用的计算方法。

## 泄漏来自仍然可达的引用链

Android 上的 Kotlin／Java 对象由垃圾回收器（GC）管理。GC 从 GC Roots 出发，沿强引用找出所有可达的对象，回收不可达的对象。GC Roots 是 GC 追踪对象时使用的一组起点，例如静态变量、各线程正在执行的方法中的局部变量。

GC 能回收互相引用、但从 GC Roots 走不到的对象，所以在 Android 上，大多数闭包捕获页面对象时不需要像 Swift 那样写 `[weak self]`。GC 与 ARC 的完整对比见[ARC 与 Android GC：引用计数、可达性与引用环](../language/swift-memory-and-references.md#arc-与-android-gc引用计数可达性与引用环)。

内存泄漏是对象已经不再需要、却仍然无法回收的情况。Android 的内存泄漏通常是这种结构：一个存活时间长的对象，仍然引用着一个本该销毁的对象，最常见的被引用对象是 Activity。Activity 持有整棵 View 树，一个 Activity 泄漏会让页面的所有 View 和 View 引用的图片都无法回收。

## 常见泄漏场景

| 场景 | 引用链 | 修正方式 |
| --- | --- | --- |
| 单例或静态变量保存 Activity | GC Root（静态变量）→ 单例 → Activity | 改存 `applicationContext`，或在页面销毁时清空引用 |
| Handler 发送延迟消息 | 主线程 `MessageQueue` → `Message` → Handler → Activity | 页面销毁时调用 `handler.removeCallbacksAndMessages(null)` 移除消息 |
| 注册监听后没有注销 | 系统服务或全局对象 → 监听器 → Activity | 在对称的生命周期方法中注销，例如 `onStart()` 注册、`onStop()` 注销 |
| 后台线程或长时间运行的协程引用页面 | 正在运行的线程 → 代码块 → Activity | 使用随页面取消的作用域，例如 `lifecycleScope`、`viewModelScope` |

以单例为例：

```kotlin
object ImageCache {              // object 声明单例，单例与进程存活时间相同
    var context: Context? = null
}

// 在 DetailActivity 中
ImageCache.context = this        // 单例引用了 DetailActivity
```

用户关闭详情页后，DetailActivity 本应被回收。但 `ImageCache` 是静态可达的单例，`ImageCache.context` 仍引用 DetailActivity，GC 从 GC Roots 能走到 DetailActivity，所以 DetailActivity 和整棵 View 树都无法回收。

修正方式是 `ImageCache.context = applicationContext`。`applicationContext` 本来就与进程存活时间相同，被单例引用不会延长任何页面的存活时间。Context 的种类见[Context 与 Application](app-components-and-context.md#context-与-application)。

Handler 场景中，Kotlin 的匿名对象或 Java 的非静态内部类会隐式引用外部的 Activity。延迟消息在 `MessageQueue` 中等待期间，`Message` 通过 `target` 字段引用 Handler，Handler 又引用 Activity。消息执行或被移除之前，Activity 无法回收。Handler 与消息队列见[Android 主线程、Handler 与 ANR](main-thread-and-handler.md)。

## 用 LeakCanary 检测泄漏

LeakCanary 是检测 Android 内存泄漏的开源库。debug 包接入 LeakCanary 后，LeakCanary 按以下顺序工作：

1. Activity、Fragment 等对象销毁时，LeakCanary 用弱引用记录这些对象。
2. 等待几秒并主动触发 GC 后，对象仍未被回收，LeakCanary 判定对象可能泄漏。
3. LeakCanary 导出堆内存快照，分析出从 GC Roots 到泄漏对象的引用链。
4. LeakCanary 用通知展示引用链，开发者根据引用链找到应该断开的引用。

iOS 上排查无用对象的方法见[iOS 内存泄漏与无用对象排查](../ios/memory-leak-diagnosis.md)。

## OOM 与 Bitmap 内存

OOM（`OutOfMemoryError`）是 App 申请内存时，Java 堆的已用内存加上本次申请超过系统给进程设定的 Java 堆上限，ART 抛出的错误。Java 堆是 ART 存放 Kotlin／Java 对象的内存区域。内存泄漏让无用对象持续占用 Java 堆，是 OOM 的常见原因之一。

Bitmap 是 Android 中保存解码后图片像素的对象。Bitmap 占用的内存只由像素尺寸和像素格式决定，与图片文件的大小无关：

```text
Bitmap 内存 = 宽 × 高 × 每像素字节数
```

例如，一张 1080 × 1920 的图片以 `ARGB_8888` 格式解码，每像素 4 字节，Bitmap 占用 1080 × 1920 × 4 = 8,294,400 字节，约 7.9 MB。即使这张图片的 JPEG 文件只有 300 KB，解码后也占用约 7.9 MB。

Android 3.0 至 7.1，Bitmap 的像素数据存放在 Java 堆中，解码几张大图就可能触发 OOM。Android 8.0（API level 26）起，Bitmap 的像素数据存放在原生内存中，不计入 Java 堆上限，但仍计入进程占用的总内存。进程占用内存过多时，系统内存不足会结束进程。所以展示大图时，要按显示尺寸缩小后再解码。压缩数据、像素内存与降采样的原理见[iOS 图片加载、解码与内存](../ios/image-loading-decoding-and-memory.md)，两端的计算方式相同。
