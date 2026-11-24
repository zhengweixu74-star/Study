# iOS 后台执行机制

[返回笔记目录](../README.md)

## 本文目录

- [挂起与后台执行的前提](#挂起与后台执行的前提)
- [短时后台任务：beginBackgroundTask](#短时后台任务beginbackgroundtask)
- [后台模式：持续执行某一类工作](#后台模式持续执行某一类工作)
- [BGTaskScheduler：由系统安排执行时间](#bgtaskscheduler由系统安排执行时间)
- [后台 URLSession：由系统进程完成传输](#后台-urlsession由系统进程完成传输)
- [静默推送：由服务端唤醒应用](#静默推送由服务端唤醒应用)
- [与 Android 后台执行方式的对照](#与-android-后台执行方式的对照)

## 挂起与后台执行的前提

挂起是指应用进程仍在内存中，但系统不再给应用的线程分配 CPU 时间，应用的代码完全不执行。iOS 应用进入后台后，系统默认很快把应用挂起。内存不足时，系统直接终止挂起的应用，并且不通知应用。运行状态的完整说明见[iOS 应用生命周期、启动与窗口](app-lifecycle-and-events.md#五种运行状态)。

iOS 没有一个让应用在后台通用运行的组件。应用要在后台执行代码，必须按用途使用系统提供的专门机制。每种机制允许的工作类型、执行时机和可用时长都由系统决定。

## 短时后台任务：beginBackgroundTask

应用刚进入后台时，可以调用 `beginBackgroundTask(withName:expirationHandler:)` 向系统申请一段额外的执行时间，用来完成手头的工作，例如把编辑到一半的草稿写入文件，或者发完一个正在进行的小请求。

```swift
let taskID = UIApplication.shared.beginBackgroundTask(withName: "SaveDraft") {
    // 系统给的时间即将用完时调用这个闭包，应用必须在这里停止工作
}
saveDraft()
UIApplication.shared.endBackgroundTask(taskID)   // 工作完成，通知系统可以挂起应用
```

系统给的时间不固定，通常只有几十秒。每次调用 `beginBackgroundTask` 都必须对应一次 `endBackgroundTask`。

## 后台模式：持续执行某一类工作

后台模式是应用向系统声明的后台用途。开发者在 Xcode target 的 Signing & Capabilities 中添加 Background Modes 并勾选用途，Xcode 把勾选结果写入 `Info.plist` 的 `UIBackgroundModes`。应用正在执行声明的工作时，系统让应用在后台持续运行：

| 模式 | 允许持续运行的条件 |
| --- | --- |
| `audio` | 正在播放音频 |
| `location` | 正在持续获取位置，例如导航 |
| `voip` | 正在进行网络通话 |
| `bluetooth-central` | 正在作为中心设备与蓝牙外设通信 |

以音乐播放为例：应用声明 `audio` 模式，并把 `AVAudioSession` 的类别设为 `.playback`。音频播放期间，应用在后台持续运行；音乐停止后，系统很快挂起应用。

每种后台模式只允许对应用途。应用声明了某种模式却不执行对应工作，提交 App Store 审核时会被拒绝。

## BGTaskScheduler：由系统安排执行时间

`BGTaskScheduler` 从 iOS 13 开始提供。应用提交后台任务请求，系统挑选合适的时间唤醒应用执行任务。任务分两种：

| 任务类型 | 用途 | 执行条件 |
| --- | --- | --- |
| `BGAppRefreshTask` | 短时刷新，例如提前拉取最新内容 | 运行时间通常是几十秒 |
| `BGProcessingTask` | 耗时较长的处理，例如整理数据库 | 系统通常在设备充电且无人使用时执行 |

系统根据用户使用应用的频率、电量和网络状况决定执行时间。用户很少打开的应用，提交的任务可能很久都不执行。

## 后台 URLSession：由系统进程完成传输

应用用 `URLSessionConfiguration.background(withIdentifier:)` 创建 `URLSession`，在这个 session 上发起下载或上传。传输由系统的独立进程执行，所以应用被挂起或被系统终止后，传输仍会继续。

传输完成后，系统在后台唤醒应用，调用 `application(_:handleEventsForBackgroundURLSession:completionHandler:)`。应用在这个方法中处理下载好的文件，处理完调用传入的 `completionHandler`。

## 静默推送：由服务端唤醒应用

静默推送是不向用户显示内容、只用于唤醒应用的远程推送。应用需要声明 `remote-notification` 后台模式；服务端在推送的 `aps` 中设置 `"content-available": 1`。

系统收到静默推送后，在后台唤醒应用，调用 `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)`，应用有大约 30 秒拉取数据。系统会限制静默推送的唤醒频率，所以静默推送不能保证每条都唤醒应用。

## 与 Android 后台执行方式的对照

Android App 进入后台后，系统默认允许 App 继续运行，只在内存不足或后台限制生效时停止或结束 App；App 需要长时间运行时，启动前台 Service 并显示通知，见[Android 应用组件、Context 与权限](../android/app-components-and-context.md#app-进入后台后的进程优先级)。iOS 应用进入后台后默认被挂起，只有在系统允许的用途中才能执行代码。

| 需求 | Android | iOS |
| --- | --- | --- |
| 进入后台后完成手头的工作 | 线程或协程继续执行；进程可能被结束时交给 WorkManager | `beginBackgroundTask` |
| 后台播放音乐 | 前台 Service + 通知 | `audio` 后台模式 |
| 导航时持续定位 | 前台 Service（`location` 类型）+ 通知 | `location` 后台模式 |
| 定期同步数据 | WorkManager | `BGTaskScheduler` |
| 下载大文件 | `DownloadManager` 或 WorkManager | 后台 `URLSession` |
| 服务端通知应用拉取新数据 | 推送消息 + WorkManager | 静默推送 |
