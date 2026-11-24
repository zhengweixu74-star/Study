# Kotlin 基础语法与 Swift 对照

[返回笔记目录](../README.md)

Kotlin 是 Android 开发的主要语言，运行在 Java 虚拟机和 Android 的 ART 虚拟机上，可以直接调用 Java 代码。本文按 Swift 开发者的视角说明 Kotlin 的变量、空安全、类与继承、特殊的类声明、函数与 lambda、`when` 和作用域函数。协程见[Kotlin 协程与 suspend](kotlin-coroutines-and-suspend.md)。

## 变量与空安全

| Kotlin | Swift | 含义 |
| --- | --- | --- |
| `val name = "笔记"` | `let name = "笔记"` | 声明后不能重新赋值的变量 |
| `var count = 0` | `var count = 0` | 可以重新赋值的变量 |
| `val title: String?` | `let title: String?` | 可空类型，值可以是 `null` |
| `title?.length` | `title?.count` | 安全调用：`title` 为 `null` 时整个表达式得到 `null` |
| `title ?: "无标题"` | `title ?? "无标题"` | 左侧为 `null` 时使用右侧的值，Kotlin 中叫 Elvis 运算符 |
| `title!!` | `title!` | 断言非空；值为 `null` 时抛出 `NullPointerException` |

Kotlin 的 `val` 只限制变量不能指向另一个对象，对象内部仍可以修改。例如 `val notes = mutableListOf<String>()` 之后，`notes.add("新笔记")` 可以执行，`notes = mutableListOf()` 不能执行。Swift 中 `let` 声明的数组是值类型，`let` 数组整体不能修改。

## 类、继承与构造

```kotlin
open class BaseActivity : AppCompatActivity()   // ":" 表示继承；open 允许其他类继承 BaseActivity

class NoteViewModel(private val api: NotesApi) : ViewModel() {
    // 括号中是主构造函数的参数；private val 同时声明了私有属性 api
}
```

- `:` 后面写父类，父类名后的 `()` 表示调用父类的构造函数。
- Kotlin 的类默认是 `final`，不能被继承；加 `open` 后才能被继承。Swift 的类默认可以被继承，加 `final` 才禁止继承。
- 重写父类方法时必须写 `override`，与 Swift 相同。
- 创建对象时直接写 `NoteViewModel(api)`，不写 `new`。

## data class、object、companion object 与 sealed class

| Kotlin | Swift 中相近的写法 | 说明 |
| --- | --- | --- |
| `data class Note(val id: Long, val title: String)` | `struct Note: Equatable, Hashable` | 编译器自动生成 `equals()`、`hashCode()`、`toString()` 和 `copy()` |
| `object ImageCache { ... }` | `static let shared` 单例 | `object` 直接声明一个单例；第一次访问时创建，之后一直是同一个对象 |
| `companion object { const val TAG = "Note" }` | `static let tag = "Note"` | 类内部的伴生对象，成员通过类名访问，例如 `NoteActivity.TAG`；Kotlin 没有 `static` 关键字 |
| `sealed class LoadState` | 带关联值的 `enum` | 所有子类在编译时已知，配合 `when` 可以做穷尽判断 |

data class 是引用类型，Swift 的 `struct` 是值类型：把一个 `Note` 对象赋给另一个变量，两个变量指向同一个对象。需要修改部分字段时，用 `note.copy(title = "新标题")` 创建新对象。

## 函数、扩展函数与 lambda

```kotlin
fun add(a: Int, b: Int): Int {                 // fun 声明函数，返回类型写在参数列表之后
    return a + b
}

fun String.isEmail(): Boolean = contains("@")  // 扩展函数：给 String 增加 isEmail() 方法

val double = { x: Int -> x * 2 }               // lambda，相当于 Swift 的 { x in x * 2 }

button.setOnClickListener { titleView.text = "已点击" }
```

- 扩展函数的作用与 Swift 的 `extension` 相同。扩展函数内部的 `this` 指被扩展的对象，上例中 `contains("@")` 就是 `this.contains("@")`。
- 函数的最后一个参数是 lambda 时，lambda 可以写在括号外；括号中没有其他参数时，括号可以省略。`setOnClickListener { ... }` 就是这种写法，与 Swift 的尾随闭包相同。
- lambda 只有一个参数时，可以不声明参数名，用 `it` 指代这个参数，例如 `notes.filter { it.title.isNotEmpty() }`。

## when 表达式

`when` 的作用接近 Swift 的 `switch`，并且可以作为表达式返回值：

```kotlin
sealed class LoadState {
    object Loading : LoadState()
    data class Success(val notes: List<Note>) : LoadState()
    data class Error(val message: String) : LoadState()
}

val text = when (state) {
    is LoadState.Loading -> "加载中"
    is LoadState.Success -> "共 ${state.notes.size} 条"   // is 判断类型后，state 自动转换为 Success
    is LoadState.Error -> state.message
}
```

`state` 是 `sealed class` 类型时，编译器知道所有子类。`when` 作为表达式使用时，缺少任何一个子类的分支都会编译报错。

## 作用域函数：let、apply、also、run、with

作用域函数是在一个对象上执行代码块的标准库函数。五个函数的区别只在两点：代码块中用 `it` 还是 `this` 指代对象，以及函数返回对象本身还是代码块最后一行的结果。

| 函数 | 代码块中的对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `let` | `it` | 代码块结果 | 对可空对象执行代码：`title?.let { show(it) }` |
| `apply` | `this` | 对象本身 | 创建对象后设置属性 |
| `also` | `it` | 对象本身 | 在链式调用中插入附加操作，例如打印日志 |
| `run` | `this` | 代码块结果 | 在对象上执行计算并返回结果 |
| `with(对象)` | `this` | 代码块结果 | 对同一个对象连续调用多个方法；`with` 是普通函数，对象作为参数传入 |

```kotlin
val titleView = TextView(context).apply {
    text = "我的笔记"      // 等价于 this.text = "我的笔记"
    textSize = 18f
}
```

`apply` 的代码块中，`this` 指新建的 TextView，所以可以直接写属性名。`apply` 返回 TextView 本身，`titleView` 得到的是设置好属性的 TextView。
