# Kotlin 协程与 suspend

[返回笔记目录](../README.md#语言与运行时)

本文说明协程、挂起函数与线程的关系，并用 Swift 并发中的相近概念辅助理解。

## 协程与挂起函数的关系

协程是一段可以暂停、之后再继续执行的计算过程。线程是操作系统调度执行代码的资源。协程执行代码时要占用线程；协程挂起等待时，线程可以去执行其他工作。多个协程可以轮流使用同一个线程，所以协程数量与线程数量没有对应关系。

挂起是指协程保存继续执行所需的状态，然后暂停。等待的条件满足后，协程库再安排协程继续执行。挂起期间线程不必停在原处等待，协程恢复时也不一定回到原来的线程。

`suspend` 是 Kotlin 声明挂起函数的修饰符。挂起函数中可以调用其他挂起函数，但调用不一定真的挂起。调用挂起函数不会创建新协程，`suspend` 本身也不会创建线程或切换线程。

协程是执行过程，`suspend fun` 是在这个过程中被调用的函数。一个协程可以依次调用多个挂起函数；同一个挂起函数也可以被不同协程调用。

## 启动、调用与执行顺序

`CoroutineScope` 是协程作用域，为其中启动的协程提供协程上下文，并统一管理这些协程的生命周期，例如作用域取消时取消所有协程。协程上下文保存协程的 Job（用于取消和等待）、调度器等信息。调度器决定协程在哪些线程上执行。`launch` 是 `kotlinx.coroutines` 提供的协程构建函数，通常用于启动不直接返回计算结果的协程。

挂起函数只能在另一个挂起函数或协程代码块（例如 `launch` 的代码块）中调用，普通函数不能直接调用挂起函数。下面的例子依赖 `kotlinx.coroutines`，作用域由调用方传入。

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

`loadAndPrint` 调用 `launch` 启动协程，然后返回，不等待协程完成。协程开始执行后调用 `loadNumber()`。`delay(1000)` 使协程挂起等待约一秒，等待期间不阻塞执行该协程的线程。约一秒后，协程被安排恢复执行，`loadNumber()` 返回 `42`，再执行 `println(number)`。

同一个协程中的代码仍按顺序执行：调用 `loadNumber()` → 等待调用返回 → 把结果赋给 `number` → 打印结果。挂起期间，协程不会跳过尚未返回的调用继续往下执行。如果作用域在等待期间被取消，协程会提前结束，不再打印。

## 挂起与阻塞的区别

阻塞是线程停在某个操作上等待，在此期间这个线程不能做其他工作。挂起暂停的是协程，等待期间线程可以执行其他工作。

在 JVM 平台上，`Thread.sleep(1000)` 会阻塞当前线程；把这行代码写进 `suspend fun`，线程照样被阻塞。`suspend` 不会把阻塞调用变成非阻塞调用，也不会把耗时计算移到后台线程。

## 与 Swift 并发的对应关系

Swift 的任务是由 Swift 并发运行时管理的一段异步工作。Swift 的异步函数使用 `async` 声明；调用方通常使用 `await` 标记可能挂起的调用位置。

| Kotlin 概念或写法 | Swift 中相近的概念或写法 | 对应边界 |
| --- | --- | --- |
| 协程 | Swift 任务 | 都保存异步工作的执行状态；两者的组织和调度规则不完全相同 |
| `suspend fun load(): Int` | `func load() async -> Int` | 都声明函数中可以挂起；声明本身不创建任务 |
| `scope.launch { ... }` | `Task { ... }` | 都启动一段异步工作；生命周期、上下文继承和父子关系的规则不同 |
| 在协程中调用 `load()` | 在异步上下文中调用 `await load()` | Kotlin 调用挂起函数时不写任何标记；Swift 必须用 `await` 标出可能挂起的位置 |

Kotlin 的 `suspend` 修饰函数声明，对应 Swift 的 `async`，而不是调用处的 `await`。两种语言中，“可以挂起”都不代表“一定挂起”，也都不会因此切换到主线程或后台线程。Swift 任务的具体挂起过程见 [任务挂起与线程释放](swift-concurrency.md#任务挂起与线程释放)。

Android 页面中的协程作用域、线程切换和状态收集见[ViewModel 状态暴露与页面收集：StateFlow 与 repeatOnLifecycle](../android/stateflow-and-lifecycle-collection.md)。
