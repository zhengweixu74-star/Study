# ValueAnimator、插值器与滚动动画

[返回笔记目录](../README.md)

ValueAnimator 按动画时间计算数值，并在每一帧调用更新回调，把当前结果交给业务代码。ValueAnimator 只计算数值；业务代码在更新回调中读取结果，自己修改 View 的位置或滚动距离。

## 每一帧的执行顺序

以下流程适用于只有起点和终点两个值的浮点动画，并且动画正向播放、不重复、不暂停、不手动调整进度。时间进度是当前播放时间占本轮动画时长的比例。动画进度是插值器根据时间进度计算出的比例。动画值是根据动画进度和起终点计算出的数值。

```text
ValueAnimator 计算时间进度
→ 调用插值器的 getInterpolation(timeFraction)
→ 保存计算结果，供 animatedFraction 读取
→ 根据起终点计算 animatedValue
→ 调用 addUpdateListener 注册的更新回调
→ 业务代码读取结果并更新视图
```

使用默认浮点评估方式时：动画值 = 起点 + (终点 − 起点) × 动画进度。回调次数和间隔由帧调度决定，相邻两次回调之间动画值的增量通常不相等。

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

## ValueAnimator、ObjectAnimator 与 ViewPropertyAnimator

三种动画方式都由 Choreographer 每帧驱动，区别在于动画值如何写到 View 上：

| 方式 | 写法 | 如何修改 View |
| --- | --- | --- |
| ValueAnimator | `ValueAnimator.ofFloat(0F, 1F)` 加更新回调 | 只计算数值，业务代码在回调中自己修改任意属性 |
| ObjectAnimator | `ObjectAnimator.ofFloat(view, "translationY", 0F, 300F)` | ValueAnimator 的子类，按属性名通过反射找到 `setTranslationY()`，每帧调用 |
| ViewPropertyAnimator | `view.animate().translationY(300F)` | 直接修改 View 的显示参数，不经过反射；同时修改多个属性时共用一个动画 |

性能主要取决于动画修改的是哪种属性，与选用哪个动画类关系不大。修改 `translationY`、`scaleX`、`alpha` 等显示参数时，每帧只更新 RenderNode；修改背景色时每帧都要 `invalidate()` 并重新执行 `onDraw()`；修改宽高时每帧都要重新测量和布局。三类修改的开销见[硬件加速与 RenderNode](view-drawing-and-frame-refresh.md#硬件加速与-rendernode)。

例如用 ValueAnimator 在回调中设置 `view.translationY = value`，与用 ViewPropertyAnimator 执行 `view.animate().translationY(300F)`，每帧做的工作都是更新 RenderNode 的显示参数，开销同级。ViewPropertyAnimator 省去了反射和回调的少量开销，适合只改 View 显示参数的动画；滚动距离、绘制颜色这类不属于显示参数的值，需要用 ValueAnimator 在回调中自己写入。
