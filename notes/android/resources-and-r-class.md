# Android 资源与 R 类

[返回 Android 分类目录](README.md)

资源是与代码分开存放的布局、图片、文字、颜色和尺寸等内容，通常位于模块的 `res/` 目录。R 类是构建工具根据资源生成的类，用整数字段保存资源 ID。资源 ID 是用来标识资源的整数，不是资源内容本身。

## R 类解决什么问题

Kotlin 代码通过 R 类引用资源。例如，`setContentView(R.layout.activity_main)` 把布局的资源 ID 传给系统。系统再根据资源 ID 加载布局。

代码引用不存在的 `R.layout.activity_mian` 时，编译器会报错。开发者因此可以在运行 App 之前发现拼写错误。

## R 类的结构

下面用 Java 写法示意 R 类的结构。构建工具不一定生成可见的 `R.java` 源文件，示例中的数值也只用于说明。

```java
public final class R {
    public static final class layout {
        public static int activity_main = 0x7f0c001c;
    }
    public static final class id {
        public static int title = 0x7f0900a1;
    }
    public static final class string {
        public static int app_name = 0x7f0f001b;
        public static int refresh = 0x7f0f003e;
    }
}
```

- `layout`、`id`、`string` 是 R 类按资源类型划分的内部类。
- `static` 表示代码不创建对象也能访问字段，例如 `R.string.refresh`。
- 构建工具分配整数值。资源变化后，整数值可能改变。开发者不应手动修改 R 类，也不应持久保存资源 ID。
- 资源字段不一定是编译期常量。编译期常量是编译时就已确定、可以直接写入调用处的值。库模块的资源字段不是编译期常量。Android Gradle Plugin 是 Gradle 用来构建 Android 工程的插件。Android Gradle Plugin 8.0 起，应用模块的资源字段默认也不带 `final`，因此不能用作 Java `switch` 的 `case` 值。

## 资源与字段的对应规则

资源名称来自文件名、XML 中的 `name` 属性或 id 声明。

| 定义方式 | 示例 | 代码中的引用 |
| --- | --- | --- |
| 布局文件 | `res/layout/activity_main.xml` | `R.layout.activity_main` |
| 图片文件 | `res/drawable/ic_note.png` | `R.drawable.ic_note` |
| 文字条目 | `<string name="refresh">` | `R.string.refresh` |
| 颜色条目 | `<color name="brand">` | `R.color.brand` |
| 尺寸条目 | `<dimen name="padding_large">` | `R.dimen.padding_large` |
| 控件 id | `android:id="@+id/title"` | `R.id.title` |

文件型资源用不带扩展名的文件名作为资源名。`values/` 中的条目用 `name` 属性作为资源名，`strings.xml` 这个文件名不参与命名。

`@+id/title` 中的加号表示：名为 title 的 id 尚不存在时，构建工具创建 id。`@id/title` 表示引用已有 id。

## 资源名称与资源内容

资源名称供代码和 XML 查找资源，资源内容供 App 使用。

```xml
<resources>
    <string name="app_name">我的笔记</string>
    <string name="refresh">刷新</string>
</resources>
```

`app_name` 是资源名称，“我的笔记”是资源内容。`getString(R.string.app_name)` 返回“我的笔记”。

`app_name` 只是常用名称，本身没有特殊作用。清单文件通过 `android:label="@string/app_name"` 引用该条目后，“我的笔记”才会成为应用的显示名称。

## 资源命名限制

文件型资源和 XML 条目的命名限制不同。

| 名称来源 | 能否使用大写 | 示例 |
| --- | --- | --- |
| `layout/`、`drawable/` 等目录中的资源文件名 | 不能 | `ic_note.png` |
| `values/` 中的 `name` 属性 | 可以 | `appName` 对应 `R.string.appName` |
| 控件 id | 可以 | `titleView` 对应 `R.id.titleView` |

文件型资源名只能包含小写字母、数字和下划线，并且需要能作为合法的字段名，例如不能以数字开头。

- `ic-note.png` 不合法。连字符在 Kotlin 和 Java 中是减号，不能直接作为字段名的一部分。
- `IcNote.png` 不合法。Kotlin 和 Java 都允许标识符包含大写字母，所以大写限制来自 Android 构建工具。一个常见推断是：统一小写可以避免不同文件系统处理大小写的差异。此推断不是官方给出的原因。

## 从资源 ID 获取内容

代码需要把资源 ID 交给相应的 API，才能得到资源内容或对象。

```kotlin
setContentView(R.layout.activity_main)
val text = getString(R.string.refresh)
val titleView = findViewById<TextView>(R.id.title)
val icon = ContextCompat.getDrawable(this, R.drawable.ic_note)
```

以 `getString(R.string.refresh)` 为例：

1. 代码把资源 ID 传给 `getString()`。
2. 系统在资源表中查找资源 ID。资源表记录资源 ID、设备配置与资源值或文件路径的对应关系，在 APK 安装包中保存为 `resources.arsc`。
3. 系统按当前设备配置选择匹配的资源。
4. `getString()` 返回选中的文字。

`R.id.title` 只是整数。Activity 调用 `findViewById(R.id.title)` 时，系统在 Activity 当前的 View 树中查找控件。不同布局可以声明同名 id，并共用同一个整数。同一棵 View 树中有多个控件使用相同 id 时，`findViewById()` 只返回第一个匹配的控件。

## 资源限定符与多语言

资源限定符是资源目录名中的配置后缀。系统根据语言、深色模式和屏幕密度等配置选择资源。

| 目录 | 适用配置 |
| --- | --- |
| `values/` | 默认配置 |
| `values-en/` | 英文 |
| `values-night/` | 深色模式 |
| `drawable-xxhdpi/` | xxhdpi 屏幕密度 |

例如，`values/strings.xml` 把 `refresh` 定义为“刷新”，`values-en/strings.xml` 把 `refresh` 定义为“Refresh”。两个条目共用 `R.string.refresh`。系统语言是英文时，`getString(R.string.refresh)` 返回“Refresh”。系统语言没有匹配的目录时，`getString(R.string.refresh)` 返回默认目录中的“刷新”。

## XML 引用与代码引用

XML 和代码使用不同写法引用同一个资源。

| 使用位置 | 写法 |
| --- | --- |
| XML | `android:text="@string/refresh"` |
| Kotlin | `getString(R.string.refresh)` |

构建工具编译 XML 时，会把 `@string/refresh` 解析为对资源 ID 的引用。

## 多模块中的 R 类

模块是可以单独构建的一组代码和资源。每个 Android 模块按构建配置中的 `namespace` 生成 R 类。`namespace` 是模块生成类所用的包名。例如，`namespace` 为 `com.example.notes` 时，R 类的完整名称是 `com.example.notes.R`。

非传递 R 类是只包含本模块资源的 R 类。启用非传递 R 类时，模块 A 引用模块 B 的资源，需要使用模块 B 的 R 类。Android Gradle Plugin 8.0 起默认启用非传递 R 类。

[相关笔记：Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)

[相关笔记：Activity、Fragment 与页面组织](activity-fragment-and-navigation.md)
