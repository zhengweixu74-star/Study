# ValueAnimator、插值器与滚动动画

[返回笔记目录](../README.md)

ValueAnimator 按动画时间计算数值，并通过更新回调把当前结果提供给业务代码。ValueAnimator 本身不会自动修改 View 的位置或滚动距离；业务代码需要在回调中应用结果。

## 每一帧的执行顺序

以下示例限定为正向播放、无重复、无暂停、未手动调整进度的两端点浮点动画。时间进度是当前播放时间占本轮动画时长的比例。动画进度是插值器根据时间进度计算出的比例。动画值是根据动画进度和起终点计算出的数值。

```text
ValueAnimator 计算时间进度
→ 调用插值器的 getInterpolation(timeFraction)
→ 保存计算结果，供 animatedFraction 读取
→ 根据起终点计算 animatedValue
→ 调用 addUpdateListener 注册的更新回调
→ 业务代码读取结果并更新视图
```

对默认浮点评估方式和两个端点，可写成：动画值 = 起点 + (终点 − 起点) × 动画进度。回调次数和间隔由实际调度决定，不能假设每次回调都固定增加同一段距离。

## 插值器对象与 animatedFraction 的区别

interpolator 属性保存实现 TimeInterpolator 接口的插值器对象。getInterpolation(input) 是插值器的方法：方法接收时间进度，返回动画进度。animatedFraction 属性返回 ValueAnimator 最近一帧使用的动画进度，类型是 Float。

```kotlin
valueAnimator.interpolator
// 获取插值器对象，不会返回当前进度数值

valueAnimator.interpolator.getInterpolation(timeFraction)
// 显式传入时间进度，计算动画进度

valueAnimator.animatedFraction
// 读取 ValueAnimator 已经计算好的本帧动画进度
```

插值器对象本身不知道 ValueAnimator 已经播放了多久。直接读取 interpolator，不能得到当前帧的进度。手动调用 getInterpolation() 时，调用方必须提供正确的时间进度。ValueAnimator 已经完成这项计算，因此更新回调通常直接读取 animatedFraction。

不要把 animatedFraction 再传给同一个插值器来“取得当前进度”。animatedFraction 已经经过插值器计算；再次调用会重复应用时间曲线。

Swift 类比：interpolator 类似保存计算规则的对象，getInterpolation(_:) 类似接收 Float 并返回 Float 的方法，animatedFraction 类似保存本次计算结果的 Float 属性。类比只说明规则和结果的区别，不表示两端动画 API 一一对应。

## LinearInterpolator 与 animatedValue

LinearInterpolator 使用线性规则：输入 0.25，输出也是 0.25。动画进度随时间进度等比例变化。对于两端点浮点动画，动画值因而按时间匀速变化。未显式设置插值器时，ValueAnimator 默认使用 AccelerateDecelerateInterpolator，动画通常先加速、再减速。

animatedValue 返回当前计算出的动画值。对于 ValueAnimator.ofFloat(0F, 1F) 的默认浮点评估方式，animatedValue 与 animatedFraction 的数值相同。对于 ValueAnimator.ofFloat(100F, 300F)，当 animatedFraction 为 0.5 时，animatedValue 为 200F。两项属性的含义不同。

一般插值器的输出不一定限制在 0 到 1。带有超出目标效果的插值器可能输出大于 1 或小于 0 的数值。本笔记的滚动示例明确使用 LinearInterpolator，因此不能直接把示例中的范围结论推广到所有插值器。

## 把动画进度转换成滚动距离

下面是从评论定位场景提炼出的简化示例。scrollView 表示要滚动的视图；totalDistance 表示计划移动的距离。示例在主线程配置并启动动画。

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

当 animatedFraction 为 0.3 时，travelledDistance 为 300 像素。业务代码把滚动位置设为起始位置加 300。这个乘法把比例转换成距离，不会再次执行插值。LinearInterpolator 只决定进度变化规则，不知道哪个视图应该移动多少。

多层滚动容器也可以共享一个 ValueAnimator：业务代码先计算累计移动距离，再按预定规则分配给各容器。例如外层需要移动 600 像素、内层需要移动 400 像素时，累计距离不超过 600 的部分分给外层，剩余部分分给内层。该规则是先外层、后内层，不表示两个容器全程同时滚动。

累计距离和单帧增量必须区分。scrollTo() 接收目标滚动坐标；scrollBy() 接收本次增加的滚动距离。如果使用 scrollBy()，应先计算当前累计距离减去上次累计距离，不能每一帧重复传入累计距离。

## 启动、取消与结束回调

ofFloat() 创建并配置数值范围，start() 启动动画。更新 UI 的动画应在主线程运行。用户触摸需要打断自动滚动时，业务代码可以在触摸按下事件中调用正在运行的 animator.cancel()；ValueAnimator 不会自动监听用户触摸。

cancel() 停止动画，不会强制把动画值设为终点。对于已启动的动画，取消会先触发 onAnimationCancel()，随后触发 onAnimationEnd()。因此，onAnimationEnd() 不能单独作为“正常到达终点”的证据。end() 则会把动画值推进到终点并结束动画。
