# Android View 绘制流程与屏幕刷新

[返回 Android 分类目录](README.md)

本文说明系统如何通过测量、布局、绘制三步确定 View 的尺寸、位置和内容，View 如何请求重新布局和重绘，以及屏幕刷新信号如何驱动每一帧的工作。View 与 ViewGroup 的父子关系见[Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)。

## 测量、布局与绘制

系统从根视图开始，沿 View 树依次执行三步：

| 步骤 | 回调方法 | 做什么 | UIKit 中相近的方法 |
| --- | --- | --- | --- |
| 测量（measure） | `onMeasure()` | 父 View 把尺寸约束传给子 View，子 View 算出自己想要的宽高 | `sizeThatFits(_:)`、`intrinsicContentSize` |
| 布局（layout） | `onLayout()` | 父 View 根据测量结果确定每个子 View 的位置和最终尺寸 | `layoutSubviews()` 中设置子视图的 `frame` |
| 绘制（draw） | `onDraw()` | View 用 `Canvas` 画出自己的内容 | `draw(_:)` |

`Canvas` 是 View 绘制时使用的画布对象，提供画线、画文字、画图片等方法。

自定义 ViewGroup 时通常重写 `onMeasure()` 和 `onLayout()`，决定子 View 的尺寸和位置；自定义普通 View 时通常重写 `onMeasure()` 和 `onDraw()`，决定自身尺寸和显示内容。

## MeasureSpec：父 View 传给子 View 的尺寸约束

MeasureSpec 是测量阶段父 View 传给子 View 的尺寸约束，由“模式 + 大小”两部分组成。

| 模式 | 含义 | 常见来源 |
| --- | --- | --- |
| `EXACTLY` | 子 View 的尺寸必须等于给定大小 | XML 中写了具体尺寸（如 `100dp`），或写了 `match_parent` 且父 View 尺寸已确定 |
| `AT_MOST` | 子 View 的尺寸最大不超过给定大小 | XML 中写了 `wrap_content` |
| `UNSPECIFIED` | 不限制子 View 的尺寸 | ScrollView 测量内容高度时，对子 View 的高度不设上限 |

例如，父 View 宽 1080 px，子 View 的 `layout_width` 是 `wrap_content`。父 View 传给子 View 的宽度约束是 `AT_MOST 1080`。子 View 的文字只需要 300 px 宽，子 View 就把自己的测量宽度定为 300 px。

## requestLayout 与 invalidate

| 方法 | 作用 | UIKit 中相近的方法 |
| --- | --- | --- |
| `requestLayout()` | 请求在下一帧重新测量和布局这个 View 及其父 View 链；布局结果变化时随后重绘 | `setNeedsLayout()` |
| `invalidate()` | 请求在下一帧只重新执行绘制 | `setNeedsDisplay()` |

尺寸可能变化的修改（例如 TextView 的文字变长）需要重新测量，所以要请求布局；只改变外观的修改（例如改变颜色）只需要重绘。系统控件在属性变化时会自己调用这两个方法；自定义 View 修改影响尺寸或外观的属性后，要自己调用对应的方法。

两个方法都只做标记，不会立即执行测量或绘制。同一帧内多次调用会合并为一次工作。

## VSYNC 与 Choreographer

VSYNC 是屏幕每次准备刷新画面时发出的同步信号。60 Hz 的屏幕每秒刷新 60 次，每隔约 16.6 ms 发出一次 VSYNC 信号。

Choreographer 是 Android 中接收 VSYNC 信号、协调每一帧工作的系统对象。Choreographer 收到 VSYNC 信号后，在主线程按顺序执行这一帧的工作：

```text
VSYNC 信号到达
  → 处理输入事件
  → 执行动画计算（例如 ValueAnimator 计算本帧的动画值）
  → 遍历 View 树：测量 → 布局 → 绘制
  → 把绘制结果交给系统合成并显示
```

ValueAnimator 每帧的执行时机见[ValueAnimator、插值器与滚动动画](valueanimator-and-interpolation.md)。iOS 中相近的机制是 `CADisplayLink` 与 Core Animation 的提交时机，见[Core Animation 渲染与性能分析工具](../ios/rendering-and-performance.md)。

一帧的工作超过 16.6 ms 时，这一帧错过本次刷新，屏幕继续显示上一帧的画面，用户看到的就是掉帧。所以主线程上的耗时工作会直接造成卡顿；View 树层级越深、需要测量的 View 越多，测量和布局阶段耗时越长。
