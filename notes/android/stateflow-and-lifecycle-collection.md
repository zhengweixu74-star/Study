# ViewModel 状态暴露与页面收集：StateFlow 与 repeatOnLifecycle

[返回 Android 分类目录](README.md)

本文用一个笔记列表示例说明：ViewModel 如何在后台线程加载数据并把结果保存为可观察状态，页面如何只在可见期间接收状态并刷新界面，以及 LiveData 与 StateFlow 的区别。协程、挂起和 `launch` 的基础概念见[Kotlin 协程与 suspend](../language/kotlin-coroutines-and-suspend.md)，ViewModel 跨配置变化存活的规则见[ViewModel 如何跨配置变化保留状态](activity-lifecycle-and-state.md#viewmodel-如何跨配置变化保留状态)。

## StateFlow 与 MutableStateFlow

`StateFlow` 是 Kotlin 协程库提供的可观察状态容器：`StateFlow` 始终保存一个当前值，值变化时通知所有正在收集的观察者。`MutableStateFlow` 是可以修改当前值的 `StateFlow`，通过给 `value` 赋值更新状态。

收集是观察者接收 Flow 发出值的过程，通常调用挂起函数 `collect`。收集 `StateFlow` 时：

- `collect` 开始后立即收到当前值，之后每次值变化再收到新值。
- `collect` 不会自行结束，会一直挂起等待下一个值，直到所在协程被取消。
- 新值与当前值相等时，`StateFlow` 不通知收集者。

`StateFlow` 与 Combine 的 `CurrentValueSubject` 相近：两者都保存当前值，新的观察者开始观察时都立即收到当前值。区别是，`CurrentValueSubject` 发送与当前值相等的值时仍会通知订阅者。`CurrentValueSubject` 的行为见[CurrentValueSubject：可主动发送的当前状态](../ios/reactive-programming.md#currentvaluesubject可主动发送的当前状态)。

| Kotlin | Combine | 说明 |
| --- | --- | --- |
| `MutableStateFlow(初始值)` | `CurrentValueSubject(初始值)` | 创建时都必须提供初始值 |
| `_notes.value = result` | `notes.send(result)` | 更新当前值并通知观察者 |
| `notes.collect { ... }` | `notes.sink { ... }` | 开始观察，先收到当前值 |

## 私有可变状态与公开只读状态

```kotlin
class NotesViewModel(private val api: NotesApi) : ViewModel() {
    private val _notes = MutableStateFlow<List<Note>>(emptyList())
    val notes: StateFlow<List<Note>> = _notes

    fun load() {
        viewModelScope.launch {
            val result = withContext(Dispatchers.IO) { api.fetchNotes() }
            _notes.value = result
        }
    }
}
```

`NotesApi` 是示例中假设的网络接口，`fetchNotes()` 返回笔记列表。

- `_notes` 是私有的 `MutableStateFlow`，保存当前笔记列表，初始值是空列表。只有 `NotesViewModel` 能修改 `_notes`。
- `notes` 是公开的 `StateFlow`，与 `_notes` 指向同一个对象。`StateFlow` 类型只提供读取和收集，不提供赋值。页面通过 `notes` 读取和观察列表。

这种写法让 `NotesViewModel` 成为笔记列表唯一的修改者。页面想刷新数据只能调用 `load()`，无法直接给列表赋值。列表状态出错时，只需检查 `NotesViewModel` 中修改 `_notes` 的位置。

Swift 中的对应写法是：ViewModel 保存私有的 `CurrentValueSubject`，对外只暴露 `AnyPublisher`；或者声明 `@Published private(set) var notes`，外部只能读取和订阅，不能赋值。

## ViewModel 中加载数据的执行顺序

- `viewModelScope` 是 ViewModel 自带的协程作用域。在 `viewModelScope` 中启动的协程默认在主线程执行。ViewModel 被清理时，`viewModelScope` 取消其中所有尚未结束的协程。
- `Dispatchers.IO` 是协程库提供的调度器，把代码安排到一组后台线程上执行，适合网络请求、文件读写等需要等待的工作。
- `withContext(Dispatchers.IO) { ... }` 把代码块切换到 `Dispatchers.IO` 执行。代码块执行完后，`withContext` 返回代码块的结果，协程回到调用前的调度器继续执行。

假设服务器返回 3 条笔记，页面调用 `load()` 后的执行顺序是：

1. 页面调用 `viewModel.load()`。
2. `viewModelScope.launch` 在主线程启动一个协程。
3. 协程执行到 `withContext(Dispatchers.IO)`，把 `api.fetchNotes()` 交给后台线程执行。
4. 协程挂起，等待网络结果。挂起期间主线程不被占用，可以继续处理滑动、点击和界面绘制。
5. 后台线程拿到 3 条笔记，`withContext` 结束。协程回到主线程，`result` 是这 3 条笔记。
6. 协程在主线程执行 `_notes.value = result`，`_notes` 的当前值从空列表变为 3 条笔记。
7. 正在收集 `notes` 的页面收到新列表，并刷新 RecyclerView。

`页面调用 load() → 主线程启动协程 → 后台线程请求网络 → 协程回到主线程 → _notes 更新 → 收集 notes 的页面刷新界面`

第 6 步在主线程执行，原因是收集 `notes` 的页面收到新值后会立即修改 View，而 Android 只允许主线程修改 View。`withContext` 结束时协程已回到主线程，所以第 6 步不需要再次切换线程。

| 情况 | 结果 |
| --- | --- |
| 请求进行中，用户旋转屏幕 | 旧 Activity 被销毁，新 Activity 被创建，`NotesViewModel` 继续存活。协程继续执行，请求完成后更新 `_notes`。新 Activity 收集 `notes` 时立即得到当前列表。 |
| 请求进行中，用户退出页面 | `NotesViewModel` 被清理，`viewModelScope` 取消协程，第 6 步不会执行。 |
| 页面在请求完成前开始收集 `notes` | 页面先收到初始的空列表，请求完成后再收到 3 条笔记。 |

## 页面按生命周期收集状态

```kotlin
// 在 MainActivity 的 onCreate() 中
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.notes.collect { notes -> adapter.submitList(notes) }
    }
}
```

`adapter` 是示例中的 `ListAdapter`，`submitList()` 把新列表交给 Adapter 刷新列表，见[局部更新与性能边界](listview-recyclerview-and-viewholder.md#局部更新与性能边界)。

- `lifecycleScope` 是 Activity 和 Fragment 自带的协程作用域。在 `lifecycleScope` 中启动的协程默认使用 `Dispatchers.Main.immediate`，在主线程执行，见[协程回到主线程](main-thread-and-handler.md#协程回到主线程dispatchersmain-与-dispatchersmainimmediate)。生命周期进入销毁状态时，`lifecycleScope` 取消其中的协程。Fragment 中操作视图的协程使用 `viewLifecycleOwner.lifecycleScope`，见[Fragment 中操作视图的协程](activity-lifecycle-and-state.md#fragment-中操作视图的协程使用-viewlifecycleownerlifecyclescope)。`repeatOnLifecycle` 和 `collect` 都是挂起函数，需要在协程中调用，因此先用 `lifecycleScope.launch` 启动协程。
- `Lifecycle.State.STARTED` 表示 Activity 的 `onStart()` 已调用、`onStop()` 尚未调用的状态。处于该状态时，页面至少部分可见。
- `repeatOnLifecycle(state) { ... }` 在生命周期达到 `state` 时启动新协程执行代码块，在生命周期低于 `state` 时取消该协程，再次达到 `state` 时重新启动新协程，从头执行代码块。

## 外层协程与内层协程

示例中有两层协程。`lifecycleScope.launch` 创建外层协程，外层协程创建后立即开始执行。`repeatOnLifecycle` 在外层协程中另外创建内层协程。

| 协程 | 创建并启动者 | 存活时间 | 执行内容 |
| --- | --- | --- | --- |
| 外层协程 | `lifecycleScope.launch` | `onCreate()` 到 `onDestroy()` | 调用 `repeatOnLifecycle`，然后在 `repeatOnLifecycle` 中挂起 |
| 内层协程 | `repeatOnLifecycle` | 每次 `onStart()` 到 `onStop()` | 执行 `collect`，接收列表并刷新界面 |

按时间顺序：

1. `onCreate()`：`launch` 创建外层协程，外层协程立即开始执行，调用 `repeatOnLifecycle` 后挂起。外层协程一直挂起到页面销毁。
2. 第一次 `onStart()`：`repeatOnLifecycle` 创建内层协程 1，内层协程 1 开始 `collect`，立即收到当前值。
3. `onStop()`：`repeatOnLifecycle` 取消内层协程 1，`collect` 停止。
4. 再次 `onStart()`：`repeatOnLifecycle` 创建内层协程 2，内层协程 2 重新开始 `collect`，立即收到最新的当前值。
5. `onDestroy()`：`lifecycleScope` 取消外层协程，`repeatOnLifecycle` 结束，不再创建内层协程。

## 页面不可见时停止收集

页面在后台时，用户看不到界面，继续刷新列表只会白白消耗 CPU 和电量。`repeatOnLifecycle(Lifecycle.State.STARTED)` 在 `onStop()` 时取消收集，省掉这部分开销。

停止收集后，页面回来时仍能拿到最新状态。`StateFlow` 始终保存当前值，页面重新进入 `STARTED` 状态时，新的 `collect` 立即收到当前值。例如页面在后台期间，`_notes` 从 3 条更新为 4 条；页面重新可见后，`collect` 立即收到 4 条笔记。页面只错过后台期间出现过的中间值。

旋转屏幕时，旧 Activity 被销毁，旧 Activity 的外层协程和内层协程都被取消。新 Activity 再次执行收集代码，进入 `STARTED` 状态后立即从同一个 `NotesViewModel` 得到当前列表，所以旋转不会引起重复的网络请求。

UIKit 中的近似做法是：在 `viewWillAppear` 中用 `sink` 订阅发布者，在 `viewDidDisappear` 中取消订阅。`repeatOnLifecycle` 根据生命周期自动开始和取消收集，开发者只需在一处编写收集代码。

## LiveData 与 StateFlow

LiveData 是 Jetpack 提供的可观察数据容器。页面调用 `liveData.observe(lifecycleOwner) { ... }` 注册观察者，传入的 `lifecycleOwner` 通常是 Activity 或 Fragment 的视图生命周期。LiveData 根据 `lifecycleOwner` 的状态通知观察者：

- 页面处于 `STARTED` 或 `RESUMED` 状态时，值变化后通知观察者。
- 页面处于后台时不通知；页面重新可见后，观察者收到最新值。
- 页面销毁时，LiveData 自动移除观察者。

| 对比项 | LiveData | StateFlow |
| --- | --- | --- |
| 来源 | Jetpack 的 Android 库 | Kotlin 协程库，不依赖 Android |
| 感知生命周期 | `observe()` 自带 | 自身不感知，由 `repeatOnLifecycle` 控制收集时机 |
| 初始值 | 可以没有初始值 | 必须有初始值 |
| 数据变换 | 提供 `map`、`switchMap` 等少量变换 | 可以使用 Flow 的全部操作符，例如 `debounce`、`combine` |

两者都能把 ViewModel 中的状态交给页面。LiveData 用 `observe()` 一步完成生命周期绑定；StateFlow 配合协程和 Flow 操作符，可以在 ViewModel 中组合更复杂的数据流，所以新的 Kotlin 代码多用 StateFlow。

[相关笔记：Android 页面生命周期与状态保存](activity-lifecycle-and-state.md)

[相关笔记：响应式编程与事件通知](../ios/reactive-programming.md)
