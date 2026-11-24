# Android View、ViewGroup 与 XML 布局

[返回 Android 分类目录](README.md)

Android 的 View 体系是以 `View` 和 `ViewGroup` 对象组成界面的 UI 体系。本文说明两类对象的职责、父子关系、XML 和代码创建视图的关系，以及布局使用的尺寸单位。View 的测量、布局和绘制流程见[Android View 绘制流程与屏幕刷新](view-drawing-and-frame-refresh.md)。

## View、ViewGroup 与 UIKit 的对应关系

`View` 是 Android 视图的基类，提供尺寸、绘制和输入事件等基础能力。`TextView`、`ImageView`、`Button` 都属于 View 类型。`ViewGroup` 是继承自 View 的抽象容器类，负责管理子视图；`LinearLayout` 等具体容器各自实现测量和布局规则。

ViewGroup 是 View 的子类。只有 ViewGroup 及其子类能添加和管理子视图，`TextView` 这类普通 View 不能。`LinearLayout` 和 `FrameLayout` 都是 ViewGroup 的具体子类。

| Android 概念 | UIKit 中相近的概念 | 类比边界 |
| --- | --- | --- |
| View | UIView | 都属于界面中的视图对象；UIView 自身已经提供子视图管理能力。 |
| TextView／ImageView／Button | UILabel／UIImageView／UIButton | 只比较常见用途，不表示继承结构和 API 完全相同。 |
| ViewGroup | 用作容器的 UIView | Android 用 ViewGroup 扩展子视图管理；UIKit 无须另外继承一个容器基类。 |
| addView(child) | addSubview(child) | 都可以建立视图父子关系；转移已有父视图的子视图时，两端行为不能直接等同。 |

ViewGroup 对应的是用作容器的 UIView，而非 UIViewController。UIViewController 管理页面的根视图和生命周期，ViewGroup 本身是视图。UIKit 的父子关系见[UIKit 视图层级与父子关系](../ios/uikit-layout-and-mvvm.md#视图层级与父子关系)。

## ViewGroup 可以嵌套 ViewGroup

ViewGroup 可以把另一个 ViewGroup 加为子视图，因为 ViewGroup 也是 View。父容器负责安排子容器的尺寸和位置，子容器再按自己的规则布局内部子视图。

下面的 Kotlin 片段建立三层父子关系，省略了尺寸配置。`context` 是调用处已有的 Android Context，提供创建视图所需的环境和资源访问能力。

```kotlin
val outer = FrameLayout(context)
val inner = LinearLayout(context)
val label = TextView(context)
label.text = "示例评论"

inner.addView(label)
outer.addView(inner)
```

执行结果是 `outer → inner → label`。以下 Swift 片段建立相同的 UIKit 父子关系；区别是 UIView 不会像 LinearLayout 那样自动线性排列子视图。

```swift
let outer = UIView()
let inner = UIView()
let label = UILabel()
label.text = "示例评论"

inner.addSubview(label)
outer.addSubview(inner)
```

同一个 Android View 只能有一个父容器。View 已有父容器时，要先从原容器移除，再加入新容器；直接调用 addView 会抛出异常。UIKit 的 addSubview 则会自动把子视图从原父视图移除。

## 常见布局容器及与 UIKit 的对照

在传统 Android View 体系中，布局容器按自身规则确定子视图的尺寸和位置。LinearLayout、FrameLayout、ConstraintLayout 和 RelativeLayout 都是 ViewGroup 的具体子类。

| 容器 | 排列规则 | 典型示例 |
| --- | --- | --- |
| LinearLayout（线性布局） | 沿水平或垂直方向依次排列子视图。 | 横向按钮组、纵向表单。 |
| FrameLayout（帧布局） | 允许子视图叠放，通过对齐方式和边距调整位置。 | 图片上覆盖播放按钮。 |
| ConstraintLayout（约束布局） | 通过子视图与父容器或同级视图之间的约束确定尺寸和位置。 | 头像靠左，用户名位于头像右侧，描述位于用户名下方。 |
| RelativeLayout（相对布局） | 通过相对位置规则排列子视图。 | 控件位于另一控件下方，或与父容器底部对齐。 |

LinearLayout 的排列思路类似 UIStackView，但两者的尺寸分配规则和 API 不同。FrameLayout 类似用于叠放子视图的 UIView；区别是 FrameLayout 会按对齐方式和边距测量、摆放子视图，普通 UIView 没有这套规则。

ConstraintLayout 的约束思路类似 UIKit Auto Layout。ConstraintLayout 是一个具体的 ViewGroup 容器；Auto Layout 是 UIKit 的约束布局机制，可用于任何 UIView。界面位置关系复杂时，ConstraintLayout 能在一个容器内表达多种关系，省去只为排列控件而嵌套的容器。嵌套减少后性能是否提升，要看具体场景。

XML 是描述布局的方式。以上容器既可以在 XML 中声明，也可以用 Kotlin 或 Java 代码创建。以上内容针对传统 View 体系。Jetpack Compose 是 Android 的声明式 UI 框架，用 Kotlin 函数描述界面应该显示什么，作用接近 SwiftUI；Compose 有自己的布局组件。

## XML 和代码都可以创建 Android 视图

XML 是描述布局的标记语言；与 UIKit 同一层次的是 Android View 体系。Android XML 标签描述视图类型、属性和嵌套结构。App 加载布局资源时，按标签创建 View／ViewGroup 对象并建立父子关系。

Android 也可以通过 Kotlin 或 Java 直接创建 View／ViewGroup，再调用 addView 建立层级。XML 与代码可以混合使用，例如用 XML 定义主体布局，再在运行时更新文字或添加视图。

## 尺寸单位：px、dp 与 sp

Android 设备的屏幕密度差别很大。屏幕密度是每英寸的物理像素数，单位是 dpi。布局尺寸如果直接用物理像素表示，同一个按钮在高密度屏幕上会显得更小。所以 Android 布局使用与密度无关的单位：

| 单位 | 含义 | 用途 |
| --- | --- | --- |
| px | 屏幕上的物理像素 | 很少在布局中直接使用 |
| dp | 与屏幕密度无关的长度单位，`px = dp × (dpi ÷ 160)` | 布局尺寸、边距；作用接近 iOS 的 pt |
| sp | 字体专用单位，在 dp 的基础上还会随用户在系统设置中调整的字号缩放 | 文字大小 |

例如 `16dp` 的边距，在 160 dpi 的屏幕上是 16 px，在 480 dpi 的屏幕上是 48 px，两块屏幕上看起来的物理长度相同。用户在系统设置中把字号调大到 1.3 倍时，`18sp` 的文字在 480 dpi 屏幕上显示为 18 × 3 × 1.3 ≈ 70 px。

## Android 布局编辑器的示例数据来自哪里

Android Studio 的 Layout Editor 根据 XML 布局、资源和设计时属性显示视图。查看常规 XML 布局时，开发者通常无需运行 Activity 的业务流程。预览自定义视图时，Layout Editor 会执行自定义视图的构造和绘制代码。

示例数据有两种来源。一是 Android Studio 内置的示例文字、图片和列表数据，开发者在布局编辑器中选择。二是开发者自己配置的预览文字、图片资源和列表项布局，用来检查具体业务内容的效果。编辑器拿不到后台的真实帖子和评论。

例如，TextView 的 tools:text="这是一条示例评论" 只在编辑器预览中显示；App 运行时的文字由 android:text 或业务代码设置。使用 tools 属性前，要在 XML 中声明 tools 命名空间。

```xml
<TextView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    tools:text="这是一条示例评论" />
```

列表预览还可以指定列表项布局和显示数量。观察一般间距时用内置示例即可；检查长评论换行、空内容和不同图片比例时，需要选择或准备对应的数据。
