# Android View、ViewGroup 与 XML 布局

[返回 Android 分类目录](README.md)

Android 的 View 体系是以 `View` 和 `ViewGroup` 对象组成界面的 UI 体系。本文说明两类对象的职责、父子关系，以及 XML 和代码创建视图的关系。

## View、ViewGroup 与 UIKit 的对应关系

`View` 是 Android 视图的基类，提供尺寸、绘制和输入事件等基础能力。`TextView`、`ImageView`、`Button` 都属于 View 类型。`ViewGroup` 是继承自 View 的抽象容器类，负责管理子视图；具体容器实现相应的测量和布局规则。

ViewGroup 是一种 View，不是每个 View 都持有的“管理子视图的属性”。普通 View 没有 ViewGroup 提供的子视图管理能力。`LinearLayout` 和 `FrameLayout` 都是 ViewGroup 的具体子类。

| Android 概念 | UIKit 中相近的概念 | 类比边界 |
| --- | --- | --- |
| View | UIView | 都属于界面中的视图对象；UIView 自身已经提供子视图管理能力。 |
| TextView／ImageView／Button | UILabel／UIImageView／UIButton | 只比较常见用途，不表示继承结构和 API 完全相同。 |
| ViewGroup | 用作容器的 UIView | Android 用 ViewGroup 扩展子视图管理；UIKit 无须另外继承一个容器基类。 |
| addView(child) | addSubview(child) | 都可以建立视图父子关系；转移已有父视图的子视图时，两端行为不能直接等同。 |

ViewGroup 不是 UIViewController 的对应物。UIViewController 管理页面的根视图和生命周期；ViewGroup 本身仍是视图。UIKit 的父子关系见[UIKit 视图层级与父子关系](../ios/uikit-layout-and-mvvm.md#视图层级与父子关系)。

## ViewGroup 可以嵌套 ViewGroup

ViewGroup 可以把另一个 ViewGroup 加为子视图，因为子容器也属于 View。父容器负责安排子容器的尺寸和位置，子容器再按自己的规则布局内部子视图。

下面的 Kotlin 片段只建立父子关系。`context` 是调用处已有的 Android Context，提供创建视图所需的环境和资源访问能力。片段没有完整配置各层尺寸。

```kotlin
val outer = FrameLayout(context)
val inner = LinearLayout(context)
val label = TextView(context)
label.text = "示例评论"

inner.addView(label)
outer.addView(inner)
```

执行结果是 `outer → inner → label`。对应的 UIKit 父子关系可以用以下 Swift 片段表达；UIView 本身不具有 LinearLayout 的自动线性排列规则。

```swift
let outer = UIView()
let inner = UIView()
let label = UILabel()
label.text = "示例评论"

inner.addSubview(label)
outer.addSubview(inner)
```

同一个 Android View 不能同时挂在两个父容器下。把已有父容器的 View 加入新容器前，应先从原容器移除。UIKit 的 addSubview 在转移已有子视图时会自动解除原父视图关系，因此不能把 UIKit 的这项行为直接套到 Android。

## XML 和代码都可以创建 Android 视图

XML 是描述布局的标记语言，不是与 UIKit 同一层次的 UI 框架。Android XML 标签描述视图类型、属性和嵌套结构。App 加载布局资源时，会创建相应的 View／ViewGroup 对象并建立父子关系。

Android 也可以通过 Kotlin 或 Java 直接创建 View／ViewGroup，再调用 addView 建立层级。XML 与代码可以混合使用，例如用 XML 定义主体布局，再在运行时更新文字或添加视图。

## Android 布局编辑器的示例数据来自哪里

Android Studio 的 Layout Editor 根据 XML 布局、资源和设计时属性显示视图。对于常规 XML 布局，开发者通常不需要先执行完整 Activity 业务流程就能查看布局。自定义视图仍可能执行自身的构造和绘制代码，因此不能理解为预览完全不执行代码。

示例数据有两种来源。Android Studio 提供内置的示例文字、图片和列表数据，开发者可以在布局编辑器中选择。开发者也可以配置自己的预览文字、图片资源和列表项布局，以检查具体业务内容的效果。编辑器不会自动知道后台的真实帖子和评论。

例如，TextView 的 tools:text="这是一条示例评论" 只指定编辑器预览中的文字，不设置 App 运行时的文字。使用 tools 属性时，需要在 XML 中声明 tools 命名空间；运行时文字由 android:text 或业务代码设置。

```xml
<TextView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    tools:text="这是一条示例评论" />
```

列表预览还可以指定列表项布局和显示数量。观察一般间距可以使用内置示例；检查长评论换行、空内容和不同图片比例等情况，需要选择或准备对应数据。
