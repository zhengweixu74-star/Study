# Android View 绘制流程与屏幕刷新

[返回 Android 分类目录](README.md)

本文说明系统如何通过测量、布局、绘制三步确定 View 的尺寸、位置和内容，View 如何请求重新布局和重绘，硬件加速下 RenderNode 如何缓存绘制结果，滚动偏移如何移动内容，以及屏幕刷新信号如何驱动每一帧的工作。View 与 ViewGroup 的父子关系见[Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)。

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

## 硬件加速与 RenderNode

硬件加速指由 GPU 执行 View 的绘制工作。Android 4.0 起，应用默认开启硬件加速。开启硬件加速后，每个 View 都有一个 RenderNode。

RenderNode 是 View 在硬件加速下的绘制记录，包含两部分：

- 绘制指令列表：`onDraw()` 中调用 `Canvas` 的每一步，例如“在 (0, 0) 画一个 200×100 的红色矩形”“在 (20, 40) 画文字‘你好’”。开启硬件加速后，`Canvas` 的方法只把指令记进列表，不直接画出像素。
- 显示参数：`translationX/Y`、`scaleX/Y`、`rotation`、`alpha`，以及 View 在父 View 中的位置。

每一帧，主线程把有变化的 RenderNode 交给 RenderThread。RenderThread 是负责与 GPU 通信的系统线程。GPU 按每个 RenderNode 的显示参数，把绘制指令列表重新执行一遍，画到屏幕上。

绘制指令列表缓存了 `onDraw()` 的结果：View 内容没变时，主线程不再执行 `onDraw()`，GPU 直接复用旧列表。所以修改 View 属性的开销，取决于这次修改要重做哪一步：

| 修改内容 | 例子 | 主线程要做的工作 | 开销 |
| --- | --- | --- | --- |
| 显示参数 | 改 `translationY`、`scaleX`、`alpha` | 只更新 RenderNode 的显示参数，复用绘制指令列表 | 小 |
| 绘制内容 | 改背景色、改文字颜色 | `invalidate()` → 重新执行 `onDraw()`，重新记录绘制指令列表 | 中 |
| 尺寸或位置规则 | 改宽高、改 margin | `requestLayout()` → 重新测量、布局，结果变化时再重绘 | 大 |

修改背景色必须调用 `invalidate()`：绘制指令列表里记录的仍是旧颜色，只修改颜色字段时，GPU 仍按旧指令画旧颜色。`invalidate()` 把 View 的绘制指令列表标记为过期，下一帧主线程重新执行 `onDraw()`，用新颜色重新记录指令。

iOS 中相近的对应关系：修改 `transform` 对应修改 RenderNode 显示参数，`setNeedsDisplay()` 对应 `invalidate()`。两者缓存的内容不同：CALayer 缓存绘制好的位图，RenderNode 缓存绘制指令。View 设置 `LAYER_TYPE_HARDWARE` 后，Android 也把 View 的绘制结果缓存为一张纹理。

## 滚动偏移：scrollTo 与 scrollBy

滚动偏移是容器内容相对容器自身移动的距离，保存在 View 的 `scrollX`、`scrollY` 中。`scrollTo(x, y)` 把滚动偏移设为指定值，`scrollBy(dx, dy)` 在当前偏移上再加 `dx`、`dy`。iOS 中对应的是 `UIScrollView` 的 `contentOffset`，即 `bounds.origin`。

滚动偏移不属于 RenderNode 的显示参数。容器绘制时，系统在容器自己的绘制指令列表开头记录一条“画布平移 (−scrollX, −scrollY)”指令，再记录子 View 的绘制。所以 `scrollTo()` 修改偏移后调用 `invalidate()`，下一帧只重新记录容器自己的这份短列表。子 View 的绘制指令列表原样复用，也不重新测量和布局。

例如 ScrollView 内有一个高 3000 px 的 LinearLayout，执行 `scrollView.scrollTo(0, 500)`：

1. ScrollView 的 `scrollY` 变为 500，ScrollView 调用 `invalidate()`。
2. 下一帧，ScrollView 重新记录自己的绘制指令：先“画布平移 (0, −500)”，再“绘制 LinearLayout 的 RenderNode”。
3. LinearLayout 及其子 View 的绘制指令列表没有变化，GPU 直接复用，内容整体上移 500 px。

`translationY = -500` 移动的是 ScrollView 自身：ScrollView 的背景和裁剪区域会一起移走，内容不会在 ScrollView 的范围内滚动。所以滚动内容时修改滚动偏移。

RecyclerView 的 `scrollBy()` 采用另一种方式，直接移动每一行的位置，见[RecyclerView 的滚动过程](listview-recyclerview-and-viewholder.md#recyclerview-的滚动过程回收填充与平移)。

### 屏幕坐标与目标滚动位置

`view.getLocationOnScreen(location)` 读取 View 左上角在整个屏幕上的坐标。调用方传入长度为 2 的 `IntArray`，方法把 x 写入 `location[0]`，把 y 写入 `location[1]`。屏幕左上角是 (0, 0)，往右 x 增大，往下 y 增大。

屏幕坐标已经包含了容器当前的滚动偏移。例如 ScrollView 的 `scrollY` 为 500，ScrollView 内某个 View 距离内容顶部 3860 px。这个 View 的屏幕 y 坐标是 3860 − 500 = 3360。

要让这个 View 滚到屏幕上 y = 960 的位置，有两个不同的量：

| 量 | 含义 | 计算 | 例子 |
| --- | --- | --- | --- |
| 滚动距离 | 从现在起还要再滚多少 | 当前屏幕坐标 − 目标屏幕坐标 | 3360 − 960 = 2400 |
| 目标滚动位置 | 滚完后 `scrollY` 应等于多少 | 当前 `scrollY` + 滚动距离 | 500 + 2400 = 2900 |

`scrollTo()` 需要目标滚动位置，`scrollBy()` 需要滚动距离。动画每帧调用 `scrollTo()` 时，传入的值从 500 逐步变到 2900。判断目标是否超出某个滚动范围，或把结果限制为不小于 0 时，也要用目标滚动位置比较，因为 `scrollY` 的取值范围是按位置定义的。

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
