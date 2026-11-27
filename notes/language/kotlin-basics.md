# Kotlin 基础语法与 Swift 对照

[返回笔记目录](../README.md)

Kotlin 是 Android 开发的主要语言，运行在 Java 虚拟机和 Android 的 ART 虚拟机上，可以直接调用 Java 代码。本文按 Swift 开发者的视角说明 Kotlin 的变量与常量、空安全、数字字面量与除法、数组与集合、类与继承、Java 属性访问、特殊的类声明、对象表达式、函数与 lambda、`when` 和作用域函数。协程见[Kotlin 协程与 suspend](kotlin-coroutines-and-suspend.md)。

## 变量与空安全

| Kotlin | Swift | 含义 |
| --- | --- | --- |
| `val name = "笔记"` | `let name = "笔记"` | 只能赋值一次的变量 |
| `var count = 0` | `var count = 0` | 可以重新赋值的变量 |
| `val title: String?` | `let title: String?` | 可空类型，值可以是 `null` |
| `title?.length` | `title?.count` | 安全调用：`title` 为 `null` 时整个表达式得到 `null` |
| `title ?: "无标题"` | `title ?? "无标题"` | 左侧为 `null` 时使用右侧的值，Kotlin 中叫 Elvis 运算符 |
| `title!!` | `title!` | 断言非空；值为 `null` 时抛出 `NullPointerException` |

### val、const val 与 Swift 的 let

Kotlin 文档把 `val` 声明的变量叫只读变量，Swift 文档把 `let` 声明的值叫常量。两者的规则相同：值在初始化时确定，之后不能再赋值；初始化的值可以在运行时才算出来，例如 `val now = System.currentTimeMillis()` 每次运行得到不同的值。

编译期常量用 `const val` 声明：

```kotlin
const val MAX_COUNT = 100
```

编译时，编译器把用到 `MAX_COUNT` 的地方直接替换成 `100`。`const val` 只能写在文件顶层、`object` 或 `companion object` 中，值只能是 `Int`、`Boolean` 等基本类型或 `String`。

`val` 只限制变量不能指向另一个对象，对象内部仍可以修改：

```kotlin
val notes = mutableListOf("第一条")
notes.add("第二条")        // 可以：notes 仍指向同一个列表对象，只是列表内容变了
notes = mutableListOf()   // 编译报错：val 不能指向另一个列表
```

Swift 的 `let` 修饰值类型（`struct`、`Array`、`Dictionary`）时，整个值的内容都不能修改，所以 `let list = [1, 2]` 之后不能调用 `append`。Swift 的 `let` 修饰 class 实例时，只限制不能换成另一个实例，实例的属性仍可以修改，这种情况与 Kotlin 的 `val` 效果相同：

```swift
class User { var name = "Tom" }

let user = User()
user.name = "Jerry"   // 可以：user 指向的实例没变，只修改了实例的属性
user = User()         // 编译报错：let 不能重新赋值
```

### 数字字面量与除法

数字字面量的后缀决定类型：`100` 是 `Int`，`100L` 是 `Long`，`100f` 或 `100F` 是 `Float`，`100.0` 是 `Double`。Kotlin 的整数字面量不会根据上下文变成浮点数，`val x: Float = 100` 会编译报错。Swift 的 `let x: Float = 100` 可以编译，因为 Swift 的字面量类型由上下文推断。

除法结果的类型由两边的类型决定：

| 表达式 | 结果类型 | 结果 |
| --- | --- | --- |
| `100 / 440` | `Int` | `0`，整数除法舍去小数部分 |
| `100f / 440` | `Float` | 约 `0.227` |

所以需要小数结果时，至少一边要写成浮点数。

## 数组与集合：Array、List 与 MutableList

Kotlin 的数组和列表是不同的类型：

| 类型 | 长度能否改变 | 元素能否修改 | 创建方式 |
| --- | --- | --- | --- |
| `Array<T>`、`IntArray` 等 | 不能，长度在创建时确定 | 能 | `arrayOf(1, 2)`、`IntArray(3)` |
| `List<T>` | 不能 | 不能（只读） | `listOf(1, 2)` |
| `MutableList<T>` | 能 | 能 | `mutableListOf(1, 2)` |

- `Array<T>` 是数组，即一块长度固定的连续内存。在 Android 上，`Array<T>` 编译后就是 Java 数组。
- `List<T>` 是接口。接口只规定一个类型必须提供哪些方法，本身不存储数据，作用类似 Swift 的 protocol。`List<T>` 只规定 `get`、`size`、`contains` 等读取方法，没有 `add`、`remove` 等修改方法，所以 `listOf(1, 2)` 返回的列表调用 `add(3)` 会编译报错。
- `MutableList<T>` 继承 `List<T>`，在读取方法之外增加了 `add`、`remove`、`set` 等修改方法。

存放元素的是实现 `List` 接口的类。在 Android 上，`mutableListOf()` 返回 `ArrayList` 对象。`ArrayList` 内部用数组存放元素；数组放满时，`ArrayList` 新建一个更大的数组，再把旧元素复制过去。所以与 Swift `Array` 用法最接近的是 `MutableList`：长度可变，可以增删元素。

Kotlin 的列表是引用类型，Swift 的 `Array` 是值类型。把列表赋给另一个变量后，两门语言的结果不同：

```kotlin
val a = mutableListOf(1, 2)
val b = a          // b 和 a 指向同一个列表对象
b.add(3)
println(a)         // 输出 [1, 2, 3]
```

```swift
var a = [1, 2]
var b = a          // b 得到 a 的一份拷贝
b.append(3)
print(a)           // 输出 [1, 2]
```

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
- 调用 Java 类时，Java 的 `getXxx()` 方法可以写成读取属性 `xxx`，`setXxx(value)` 可以写成给属性 `xxx` 赋值。例如 `recyclerView.context` 调用的是 `recyclerView.getContext()`，`titleView.text = "标题"` 调用的是 `titleView.setText("标题")`。

## data class、object、companion object 与 sealed class

| Kotlin | Swift 中相近的写法 | 说明 |
| --- | --- | --- |
| `data class Note(val id: Long, val title: String)` | `struct Note: Equatable, Hashable` | 编译器自动生成 `equals()`、`hashCode()`、`toString()` 和 `copy()` |
| `object ImageCache { ... }` | `static let shared` 单例 | `object` 直接声明一个单例；第一次访问时创建，之后一直是同一个对象 |
| `companion object { const val TAG = "Note" }` | `static let tag = "Note"` | 类内部的伴生对象，成员通过类名访问，例如 `NoteActivity.TAG`；Kotlin 没有 `static` 关键字 |
| `sealed class LoadState` | 带关联值的 `enum` | 所有子类在编译时已知，配合 `when` 可以做穷尽判断 |

data class 是引用类型，Swift 的 `struct` 是值类型：把一个 `Note` 对象赋给另一个变量，两个变量指向同一个对象。需要修改部分字段时，用 `note.copy(title = "新标题")` 创建新对象。

### 对象表达式：创建匿名子类的对象

`object` 也可以写在表达式中，称为对象表达式。对象表达式当场定义一个没有名字的子类，并立即创建这个子类的一个对象：

```kotlin
val listener = object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        println(newState)
    }
}
```

- `object` 表示在这里创建一个对象。
- `: RecyclerView.OnScrollListener()` 表示这个无名子类继承 `OnScrollListener`，括号表示调用父类的构造函数，有参数时写在括号中。
- 花括号中是子类的内容，例如重写的方法。

上面的代码等价于先声明有名字的子类，再创建对象：

```kotlin
class PrintStateListener : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        println(newState)
    }
}

val listener = PrintStateListener()
```

子类只在一处使用时，对象表达式省去了起名字。每次执行对象表达式都会创建一个新对象，这一点与 `object ImageCache { ... }` 这种单例声明不同。Swift 没有对应写法，需要先声明有名字的子类。

## 函数、扩展函数与 lambda

```kotlin
fun add(a: Int, b: Int): Int {                 // fun 声明函数，返回类型写在参数列表之后
    return a + b
}

fun String.isEmail(): Boolean = contains("@")  // 扩展函数：给 String 增加 isEmail() 方法

val double = { x: Int -> x * 2 }               // lambda，相当于 Swift 的 { x in x * 2 }

button.setOnClickListener { titleView.text = "已点击" }
```

- 函数体只有一个返回值表达式时，可以用 `=` 代替 `{ return ... }`，称为单表达式函数。`fun String.isEmail(): Boolean = contains("@")` 与 `fun String.isEmail(): Boolean { return contains("@") }` 等价。
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
