# ValueAnimator、插值器与滚动动画

[返回笔记目录](../README.md)

ValueAnimator 按动画时间计算数值，并在每一帧调用更新回调，把当前结果交给业务代码。ValueAnimator 只计算数值；业务代码在更新回调中读取结果，自己修改 View 的位置或滚动距离。

## 每一帧的执行顺序

以下流程适用于只有起点和终点两个值的浮点动画，并且动画正向播放、不重复、不暂停、不手动调整进度。时间进度是当前播放时间占本轮动画时长的比例。动画进度是插值器根据时间进度计算出的比例。动画值是根据动画进度和起终点计算出的数值。

```text
ValueAnimator 计算时间进度
→ 调用插值器的 getInterpolation(timeFraction)
→ 保存计算结果，供 animatedFraction 读取
→ TypeEvaluator 根据动画进度和起终点计算 animatedValue
→ 调用 addUpdateListener 注册的更新回调
→ 业务代码读取结果并更新视图
```

`ofFloat()` 创建的动画默认使用 FloatEvaluator：动画值 = 起点 + (终点 − 起点) × 动画进度。TypeEvaluator 的职责见下文[TypeEvaluator 与颜色渐变](#typeevaluator-与颜色渐变)。

回调次数和间隔由帧调度决定，相邻两次回调之间动画值的增量通常不相等。

## 插值器对象与 animatedFraction 的区别

interpolator 属性保存插值器对象，插值器实现 TimeInterpolator 接口。插值器的 getInterpolation(input) 方法接收时间进度，返回动画进度。animatedFraction 属性返回 ValueAnimator 最近一帧使用的动画进度，类型是 Float。

```kotlin
valueAnimator.interpolator
// 获取插值器对象，不会返回当前进度数值

valueAnimator.interpolator.getInterpolation(timeFraction)
// 显式传入时间进度，计算动画进度

valueAnimator.animatedFraction
// 读取 ValueAnimator 已经计算好的本帧动画进度
```

插值器对象不记录 ValueAnimator 播放了多久，所以读取 interpolator 只能拿到插值器对象，拿不到当前帧的进度。手动调用 getInterpolation() 时，调用方必须自己传入正确的时间进度。ValueAnimator 每帧已经算好动画进度，所以更新回调通常直接读取 animatedFraction。

animatedFraction 已经经过插值器计算。把 animatedFraction 再传给同一个插值器，会把时间曲线应用两次，得到错误的进度。

## LinearInterpolator 与 animatedValue

LinearInterpolator 使用线性规则：输入 0.25，输出也是 0.25，动画进度与时间进度相等。所以两端点浮点动画的动画值随时间匀速变化。未显式设置插值器时，ValueAnimator 默认使用 AccelerateDecelerateInterpolator，动画通常先加速、再减速。

animatedValue 返回当前计算出的动画值。对于 ValueAnimator.ofFloat(0F, 1F) 的默认浮点评估方式，animatedValue 与 animatedFraction 的数值相同。对于 ValueAnimator.ofFloat(100F, 300F)，animatedFraction 为 0.5 时，animatedValue 为 200F。animatedFraction 是比例，animatedValue 是按起终点换算后的数值。

插值器的输出可以超出 0 到 1。带回弹或越过目标效果的插值器，可能输出大于 1 或小于 0 的数值。下面的滚动示例使用 LinearInterpolator，动画进度始终在 0 到 1 之间；换用其他插值器时，要重新确认输出范围。

## TypeEvaluator 与颜色渐变

TypeEvaluator 是把动画进度换算成动画值的接口，只有一个方法：

```kotlin
fun evaluate(fraction: Float, startValue: T, endValue: T): T
```

ValueAnimator 每帧调用 `evaluate()`，传入本帧的动画进度、起点和终点，`evaluate()` 返回本帧的动画值。

插值器和 TypeEvaluator 负责每帧计算中前后相邻的两步：

| 对象 | 输入 | 输出 | 决定什么 |
| --- | --- | --- | --- |
| 插值器 | 时间进度 | 动画进度 | 动画的快慢节奏；不知道起点和终点 |
| TypeEvaluator | 动画进度、起点、终点 | 动画值 | 进度如何换算成具体值 |

`ofFloat()` 默认使用 FloatEvaluator，`ofInt()` 默认使用 IntEvaluator。两者都按“起点 + (终点 − 起点) × 动画进度”计算，IntEvaluator 再把结果取整。

例如 `ValueAnimator.ofFloat(100F, 300F)` 的时长为 1000 ms，播放到第 250 ms：

1. 时间进度 = 250 ÷ 1000 = 0.25。
2. 插值器计算动画进度。使用 LinearInterpolator 时，动画进度为 0.25；使用默认的 AccelerateDecelerateInterpolator 时，动画进度约为 0.146。
3. FloatEvaluator 计算动画值。动画进度为 0.25 时，动画值 = 100 + 200 × 0.25 = 150；动画进度为 0.146 时，动画值约为 129。

### 颜色为什么需要 ArgbEvaluator

Android 用一个 Int 保存颜色。颜色由透明度 A、红 R、绿 G、蓝 B 四个通道组成，每个通道取值 0 到 255，四个通道按 A、R、G、B 的顺序拼成一个 Int。例如红色是 `0xFFFF0000`（A=255、R=255、G=0、B=0），蓝色是 `0xFF0000FF`（A=255、R=0、G=0、B=255）。

用 IntEvaluator 让颜色从红色变到蓝色时，IntEvaluator 把两个颜色当作两个普通整数插值。动画进度为 0.5 时，结果拆回四个通道是 A=255、R=127、G=128、B=128，显示为灰色。原因是整数插值把四个通道当作一个整体计算：红通道差值 255 的一半是 127.5，小数部分 0.5 按位权折算进绿通道，绿通道从 0 变成了 128。

ArgbEvaluator 是颜色专用的 TypeEvaluator，每帧按三步计算：

1. 把起点颜色和终点颜色各拆成 A、R、G、B 四个通道。
2. 每个通道分别按“起点 + (终点 − 起点) × 动画进度”插值。进度为 0.5 时，R 从 255 到 0 约得 128，G 保持 0，B 从 0 到 255 约得 128。
3. 把四个通道拼回一个 Int，得到 R≈128、G=0、B≈128 的紫色。

Android 8.0 及以上的 ArgbEvaluator 先把通道值转换到线性亮度空间再插值，中间色的数值偏亮，色相仍是紫色。

```kotlin
val animator = ValueAnimator.ofObject(ArgbEvaluator(), Color.RED, Color.BLUE).apply {
    duration = 1000L
    addUpdateListener { view.setBackgroundColor(it.animatedValue as Int) }
}
animator.start()
```

`ofObject()` 的第一个参数指定 TypeEvaluator，ValueAnimator 每帧用这个 TypeEvaluator 计算动画值。`ValueAnimator.ofArgb(Color.RED, Color.BLUE)` 是等效的简写，内部使用 ArgbEvaluator。

## 把动画进度转换成滚动距离

下面的示例用动画把滚动视图平滑移动一段距离。scrollView 是要滚动的视图，totalDistance 是计划移动的距离。代码在主线程配置并启动动画。

```kotlin
val startScrollY = scrollView.scrollY
val totalDistance = 1000

val animator = ValueAnimator.ofFloat(0F, 1F).apply {
    duration = 300L
    interpolator = LinearInterpolator()
    addUpdateListener { valueAnimator ->
        val travelledDistance =
            (totalDistance * valueAnimator.animatedFraction).roundToInt()
        scrollView.scrollTo(
            scrollView.scrollX,
            startScrollY + travelledDistance
        )
    }
}
animator.start()
```

animatedFraction 为 0.3 时，travelledDistance 为 300 像素，业务代码把滚动位置设为起始位置加 300。`totalDistance * animatedFraction` 只把比例换算成距离，不再插值。LinearInterpolator 只决定进度如何随时间变化；移动哪个视图、移动多少，由业务代码决定。

多层滚动容器可以共用一个 ValueAnimator。业务代码先计算累计移动距离，再按预定规则分给各容器。例如外层需要移动 600 像素、内层需要移动 400 像素：累计距离在 600 以内的部分分给外层，超出 600 的部分分给内层。按这个规则，外层先滚完，内层再开始滚动。

scrollTo() 接收目标滚动坐标，即累计距离；scrollBy() 接收本次增加的滚动距离，即单帧增量。使用 scrollBy() 时，每帧传入“当前累计距离 − 上一帧累计距离”。每帧都传累计距离会让视图越滚越远。

## 启动、取消与结束回调

ofFloat() 创建动画并设定数值范围，start() 启动动画。更新 UI 的动画在主线程运行。ValueAnimator 不监听用户触摸；用户触摸需要打断自动滚动时，业务代码在触摸按下事件中调用 animator.cancel()。

cancel() 让动画停在当前值。已启动的动画被取消时，先触发 onAnimationCancel()，再触发 onAnimationEnd()。所以动画被取消和正常播完都会触发 onAnimationEnd()；要判断是否正常到达终点，需要结合 onAnimationCancel() 是否被调用。end() 把动画值直接推进到终点，再结束动画。

## AnimationHandler：每帧调用动画的对象

AnimationHandler 是 Android 动画框架中负责在每一帧调用正在运行的动画的类。每个线程最多有一个 AnimationHandler 实例，同一线程上的所有 ValueAnimator 共用这个实例。AnimationHandler 内部保存一个列表，记录当前正在运行的 ValueAnimator。

一个动画从启动到结束，AnimationHandler 的工作顺序如下：

```text
调用 animator.start()
→ ValueAnimator 把自己加入当前线程 AnimationHandler 的列表
→ 列表原本为空时，AnimationHandler 向 Choreographer 登记下一帧回调

每次 VSYNC 到达
→ Choreographer 调用 AnimationHandler 登记的回调
→ AnimationHandler 遍历列表，每个 ValueAnimator 计算本帧的值并调用更新回调
→ 列表不为空时，AnimationHandler 再向 Choreographer 登记下一帧

动画播完，或调用 animator.cancel()
→ ValueAnimator 把自己从列表中移除
→ 列表为空时，AnimationHandler 不再登记下一帧
```

Choreographer 每帧只调用 AnimationHandler 一次，再由 AnimationHandler 分发给所有正在运行的动画。屏幕上同时有 5 个动画时，5 个 ValueAnimator 不需要各自向 Choreographer 登记。Choreographer 见[VSYNC 与 Choreographer](view-drawing-and-frame-refresh.md#vsync-与-choreographer)。

### 运行中的动画与页面回收

动画运行期间存在一条引用链：

```text
主线程 → AnimationHandler → 列表 → ValueAnimator → 更新回调 → 回调中使用的页面对象
```

运行中的线程属于 GC Roots，所以这条链上的页面对象在动画结束前无法被 GC 回收。GC Roots 与内存泄漏的判断见[Android 内存泄漏与 OOM](memory-leaks-and-oom.md#泄漏来自仍然可达的引用链)。

页面关闭时没有调用 `cancel()`，结果取决于动画会不会自己结束：

| 动画 | 页面关闭后发生什么 | 结果 |
| --- | --- | --- |
| 有固定时长。例如时长 700 ms，在第 200 ms 时关闭页面 | 再过 500 ms 动画播完，ValueAnimator 离开 AnimationHandler 的列表，引用链断开 | 页面晚 500 ms 才能回收，不算泄漏；这 500 ms 里更新回调仍在修改已离开屏幕的 View |
| 无限重复。例如 `repeatCount = ValueAnimator.INFINITE` | 动画永远不会结束，ValueAnimator 一直留在列表中 | 页面和整棵 View 树一直无法回收，形成内存泄漏 |

所以页面销毁时要调用 `cancel()`。Fragment 中修改视图的动画，在 `onDestroyView()` 中取消。

### cancel() 与 CADisplayLink 的 invalidate()

`ValueAnimator.cancel()` 与 iOS 中 `CADisplayLink.invalidate()` 的作用相同：停止每帧回调，并从“每帧负责调用自己的对象”中移除自己。`invalidate()` 的规则见[RunLoop 的持有与 invalidate](../ios/runloop-timer-and-displaylink.md#runloop-的持有与-invalidate)。

| 职责 | iOS | Android |
| --- | --- | --- |
| 提供每帧时机 | 屏幕刷新驱动 RunLoop 回调 | VSYNC 驱动 Choreographer |
| 持有每帧要调用的对象 | RunLoop | AnimationHandler |
| 每帧被调用的对象 | CADisplayLink | ValueAnimator |
| 引用页面的位置 | `target`（例如控制器） | 更新回调 |
| 停止 | `displayLink.invalidate()` | `animator.cancel()` |

两者有三处区别：

- `invalidate()` 释放 CADisplayLink 对 `target` 的强引用；`cancel()` 只把 ValueAnimator 移出 AnimationHandler 的列表，ValueAnimator 仍保存更新回调。原因是两端回收内存的规则不同。ARC 按引用计数回收，“控制器 → CADisplayLink → 控制器”这样的引用环要主动断开；GC 按可达性回收，“页面 → ValueAnimator → 更新回调 → 页面”这样的环从 GC Roots 走不到时，整个环一起回收。两种规则见[ARC 与 Android GC](../language/swift-memory-and-references.md#arc-与-android-gc引用计数可达性与引用环)。
- 失效的 CADisplayLink 不能重新启动，需要时重新创建；被取消的 ValueAnimator 可以再次调用 `start()`。
- `cancel()` 会触发 `onAnimationCancel()` 和 `onAnimationEnd()`；`invalidate()` 不调用任何结束回调。

## ValueAnimator、ObjectAnimator 与 ViewPropertyAnimator

三种动画方式都由 Choreographer 每帧驱动，区别在于动画值如何写到 View 上：

| 方式 | 写法 | 如何修改 View |
| --- | --- | --- |
| ValueAnimator | `ValueAnimator.ofFloat(0F, 1F)` 加更新回调 | 只计算数值，业务代码在回调中自己修改任意属性 |
| ObjectAnimator | `ObjectAnimator.ofFloat(view, "translationY", 0F, 300F)` | ValueAnimator 的子类，按属性名通过反射找到 `setTranslationY()`，每帧调用 |
| ViewPropertyAnimator | `view.animate().translationY(300F)` | 直接修改 View 的显示参数，不经过反射；同时修改多个属性时共用一个动画 |

性能主要取决于动画修改的是哪种属性，与选用哪个动画类关系不大。修改 `translationY`、`scaleX`、`alpha` 等显示参数时，每帧只更新 RenderNode；修改背景色时每帧都要 `invalidate()` 并重新执行 `onDraw()`；修改宽高时每帧都要重新测量和布局。三类修改的开销见[硬件加速与 RenderNode](view-drawing-and-frame-refresh.md#硬件加速与-rendernode)。

例如用 ValueAnimator 在回调中设置 `view.translationY = value`，与用 ViewPropertyAnimator 执行 `view.animate().translationY(300F)`，每帧做的工作都是更新 RenderNode 的显示参数，开销同级。ViewPropertyAnimator 省去了反射和回调的少量开销，适合只改 View 显示参数的动画；滚动距离、绘制颜色这类不属于显示参数的值，需要用 ValueAnimator 在回调中自己写入。
