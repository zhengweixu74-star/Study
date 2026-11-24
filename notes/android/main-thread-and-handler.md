# Android 主线程、Handler 与 ANR

[返回 Android 分类目录](README.md)

本文说明 Android 主线程的规则、主线程如何靠消息循环运行、Handler 如何把代码投递到指定线程，以及主线程被阻塞时系统如何判定 ANR。

## 主线程规则

主线程是 App 进程启动时创建的第一个线程，也叫 UI 线程。Android 只允许主线程修改 View，这条规则和 iOS 只允许主线程修改 UIView 相同。

主线程同时负责处理触摸、绘制界面和执行生命周期回调。主线程执行耗时工作时，这些工作都要排队等待，界面就会卡住。所以网络请求、读写大文件、解码大图要放到子线程执行，执行完再回到主线程更新 View。

Android 在主线程发起网络请求时直接抛出 `NetworkOnMainThreadException`，App 崩溃。iOS 不会因为在主线程发网络请求而崩溃。

## Looper、MessageQueue 与 Handler

Android 主线程靠消息循环运行：主线程不断从消息队列取出一条消息，执行，再取下一条。用户点击、界面绘制、生命周期回调，都是队列中的一条条消息。消息循环的思路与 iOS 的 RunLoop 相同，RunLoop 见[RunLoop、Timer 与 CADisplayLink](../ios/runloop-timer-and-displaylink.md)。

| 对象 | 职责 |
| --- | --- |
| `Message` | 一条待执行的消息，记录执行时间、要执行的代码或数据，以及处理这条消息的 Handler |
| `MessageQueue` | 按执行时间排序的消息队列 |
| `Looper` | 绑定在一个线程上，在线程中不断从 `MessageQueue` 取出到期的消息，交给消息记录的 Handler 处理 |
| `Handler` | 向某个 Looper 的 `MessageQueue` 投递消息；Looper 取出消息后，在 Looper 所在线程上调用 Handler 处理消息 |

App 进程启动时，`ActivityThread.main()` 为主线程创建 Looper，并调用 `Looper.loop()` 进入消息循环，见[Android 系统分层与应用启动](system-architecture-and-app-launch.md)。子线程默认没有 Looper，需要时调用 `Looper.prepare()` 创建。

## 用 Handler 回到主线程

```kotlin
val mainHandler = Handler(Looper.getMainLooper())   // 向主线程消息队列投递消息的 Handler

thread {                                            // 创建并启动一个子线程
    val text = loadFromDisk()                       // 子线程读磁盘，主线程继续处理其他消息
    mainHandler.post { titleView.text = text }      // 把更新界面的代码投递到主线程
}
```

执行顺序：

1. 子线程调用 `mainHandler.post {}`。
2. `mainHandler` 把代码块包装成一个 `Message`，放进主线程的 `MessageQueue`。
3. 主线程的 Looper 取出这个 `Message`。
4. 主线程执行代码块，修改 `titleView`。

`mainHandler.post {}` 的作用相当于 iOS 的 `DispatchQueue.main.async {}`。`Handler.postDelayed(代码块, 1000)` 把消息的执行时间设为 1 秒后，Looper 到时间才取出这条消息。

Kotlin 协程在 Android 上的 `Dispatchers.Main` 也通过主线程的 Handler 投递任务。协程切换到 `Dispatchers.Main` 时，最终同样是往主线程的 `MessageQueue` 放入一条消息。协程的基本概念见[Kotlin 协程与 suspend](../language/kotlin-coroutines-and-suspend.md)，ViewModel 中用协程加载数据的写法见[ViewModel 状态暴露与页面收集](stateflow-and-lifecycle-collection.md)。

## 消息队列为空时线程休眠

`Looper.loop()` 是一个不会主动退出的循环。队列中没有到期的消息时，`MessageQueue.next()` 通过 Linux 的 `epoll` 机制让线程休眠；新消息进入队列或延迟消息到期时，线程被唤醒。`epoll` 是 Linux 提供的等待事件的系统调用，线程在等待期间不占用 CPU。所以主线程一直处在循环中，却不会空转耗电。

## ANR：应用无响应

ANR（Application Not Responding，应用无响应）是系统检测到 App 长时间没有完成某项必须及时完成的工作时，弹出的“应用无响应”对话框。用户可以选择等待或关闭 App。iOS 中相近的机制是 watchdog 结束长时间卡死的 App。

AOSP（Android 开源项目）中常见的默认阈值：

| 场景 | 阈值 |
| --- | --- |
| 输入事件（触摸、按键）没有处理完 | 5 秒 |
| 前台广播的 `onReceive()` 没有执行完 | 10 秒 |
| 前台 Service 的生命周期方法没有执行完 | 20 秒 |

这些工作都在主线程执行，所以 ANR 的直接原因通常是主线程被阻塞：主线程在做耗时计算或磁盘读写，或者主线程在等待锁、等待其他线程的结果。

Handler 持有页面导致的内存泄漏见[Android 内存泄漏与 OOM](memory-leaks-and-oom.md)。
