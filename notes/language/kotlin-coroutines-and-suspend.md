# Kotlin 协程与 suspend

[返回笔记目录](../README.md#语言与运行时)

分类：语言与运行时。本文说明协程、挂起函数与线程的关系，并用 Swift 并发中的相近概念辅助理解。

## 协程与挂起函数的关系

协程是一段可以暂停并在之后继续执行的计算过程。线程是操作系统调度代码执行的资源。协程执行代码时需要线程；协程因等待而挂起时，可以让出线程的执行机会。多个协程可以共用线程，协程数量不等于线程数量。

挂起是保存后续执行所需的状态，并暂停当前协程的执行。等待条件满足后，运行时安排协程继续执行。挂起不要求当前线程停在原处等待，也不保证恢复后仍使用原来的线程。

`suspend` 是 Kotlin 声明挂起函数的修饰符。挂起函数允许执行可能挂起的调用，但不保证每次调用都会实际挂起。普通调用挂起函数不会自动创建新协程，也不会仅因 `suspend` 而新建线程或切换线程。

协程是执行过程，`suspend fun` 声明的是可在该过程中调用的函数。一个协程可以依次调用多个挂起函数；同一个挂起函数也可以被不同协程调用。

## 启动、调用与执行顺序

`CoroutineScope` 是协程作用域，用于提供协程上下文，并组织协程的生命周期。协程上下文保存任务管理和执行调度等信息。调度器是上下文中的组件，用于安排协程在哪些线程上执行。`launch` 是 `kotlinx.coroutines` 提供的协程构建函数，通常用于启动不直接返回计算结果的协程。

挂起函数通常从另一个挂起函数或允许挂起的代码块中调用，例如 `launch` 的代码块。给函数添加 `suspend`，不会让普通同步函数自动具备调用该函数的能力。下面的例子依赖 `kotlinx.coroutines`，作用域由调用方传入。

```kotlin
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun loadNumber(): Int {
    delay(1000)
    return 42
}

fun loadAndPrint(scope: CoroutineScope) {
    scope.launch {
        val number = loadNumber()
        println(number)
    }
}
```

`loadAndPrint` 调用 `launch` 启动协程，然后返回，不等待协程完成。协程开始执行后调用 `loadNumber()`。`delay(1000)` 使协程挂起等待约一秒，等待期间不阻塞执行该协程的线程。协程恢复后返回 `42`，再执行 `println(number)`；恢复时机还受调度影响。

同一协程中的普通调用仍保持顺序：调用 `loadNumber()` → 等待调用完成 → 将结果赋给 `number` → 打印结果。挂起不会让当前协程越过尚未完成的调用。示例按正常完成路径说明；作用域取消等情况可能使协程提前结束。

## 挂起与阻塞的区别

阻塞是线程停在某个操作中等待，暂时不能继续执行分配给该线程的其他工作。挂起暂停的是协程的执行过程，等待期间可以让线程执行其他工作。

在 JVM 平台，`Thread.sleep(1000)` 会阻塞当前线程。即使把该调用写进 `suspend fun`，线程仍会阻塞。`suspend` 不会自动把阻塞接口转换为非阻塞接口，也不会把耗时计算自动移到后台线程。

## 与 Swift 并发的对应关系

Swift 的任务是由 Swift 并发运行时管理的一段异步工作。Swift 的异步函数使用 `async` 声明；调用方通常使用 `await` 标记可能挂起的调用位置。

| Kotlin 概念或写法 | Swift 中相近的概念或写法 | 对应边界 |
| --- | --- | --- |
| 协程 | Swift 任务 | 都保存异步工作的执行状态；任务组织和调度规则并不完全相同 |
| `suspend fun load(): Int` | `func load() async -> Int` | 都允许函数包含可能挂起的调用；声明本身不创建新的执行任务 |
| `scope.launch { ... }` | `Task { ... }` | 都能启动异步工作；生命周期、上下文继承和父子任务关系需要分别判断 |
| 在挂起上下文中调用 `load()` | 在异步上下文中调用 `await load()` | Kotlin 不要求在每次挂起函数调用前写 `await`；Swift 用 `await` 显式标记可能挂起的位置 |

不能把 Kotlin 的 `suspend` 直接等同于 Swift 调用处的 `await`：`suspend` 修饰函数声明，更接近 Swift 的 `async`。两种语言中的“允许挂起”都不等于“一定挂起”，也不单独保证切换到主线程或后台线程。Swift 任务的具体挂起过程见 [任务挂起与线程释放](swift-basics.md#任务挂起与线程释放)。
