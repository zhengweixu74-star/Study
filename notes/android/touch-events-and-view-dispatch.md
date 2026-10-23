# Android 触摸事件与 View 事件处理

[返回 Android 分类目录](README.md)

本文说明传统 Android View 体系中的触摸事件、ACTION_DOWN 的用途，以及 OnTouchListener 与 onTouchEvent() 的调用关系。View 与 ViewGroup 的基础职责见 [Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)。

## MotionEvent 与触摸序列

`MotionEvent` 是 Android 传递触摸信息的对象，包含事件类型、触摸坐标和时间等信息。`MotionEvent.ACTION_DOWN` 是事件类型常量，表示一次触摸序列开始。`event.actionMasked` 返回事件类型，不包含多指触摸中的触点索引信息。

| 事件 | 含义 |
| --- | --- |
| ACTION_DOWN | 第一个触点按下，触摸序列开始。 |
| ACTION_MOVE | 触点移动，可以发生多次，也可以没有。 |
| ACTION_UP | 最后一个触点抬起，触摸序列正常结束。 |
| ACTION_CANCEL | 当前接收对象的触摸处理被取消，应清理状态，不应当作正常点击完成。 |

一次单指触摸通常按 ACTION_DOWN → 零次或多次 ACTION_MOVE → ACTION_UP 的顺序发生。如果父容器中途接管触摸等情况使当前 View 失去这次触摸，当前 View 会收到 ACTION_CANCEL。多指触摸中，后续手指按下通常对应 ACTION_POINTER_DOWN，而不是再次发送 ACTION_DOWN。

ACTION_DOWN 只表示按下，不代表点击成功。用户后续可能拖动，也可能取消。普通点击优先使用 setOnClickListener()；需要在手指刚按下时执行动作，才直接判断 ACTION_DOWN。

## ACTION_DOWN 的常见用途

| 用途 | 按下时执行的动作 |
| --- | --- |
| 记录触摸起点 | 保存 event.x 和 event.y，供后续计算位移；这两个值使用接收事件的 View 的局部坐标，单位为像素。 |
| 初始化交互状态 | 记录开始时间，初始化本次拖动所需的状态。 |
| 停止已有动画 | 立即取消正在执行的自动动画，让后续交互响应用户操作。 |

按下时执行的动作应与需求对应。如果只需停止动画，就不必同时实现拖动或点击识别。动画的取消与结束回调规则见 [ValueAnimator、插值器与滚动动画](valueanimator-and-interpolation.md#启动取消与结束回调)。

## OnTouchListener 与 onTouchEvent 的调用关系

`OnTouchListener` 是通过 setOnTouchListener() 给 View 注册的外部回调，监听器的 onTouch(view, event) 接收视图和事件。`onTouchEvent(event)` 是 View 自身处理触摸事件的方法，系统控件有默认实现，自定义 View 可以重写该方法。

`dispatchTouchEvent(event)` 是 View 分发触摸事件的方法，负责决定调用哪个处理入口。对于已经收到事件、处于启用状态并沿用默认分发逻辑的 View，下列流程省略与本主题无关的过滤步骤：

```text
View.dispatchTouchEvent(event)
    ↓
是否注册了 OnTouchListener？
    ├─ 没有 → 调用 View.onTouchEvent(event)
    └─ 有 → 调用监听器的 onTouch(view, event)
                ├─ 返回 true → 该事件由监听器消费，不再调用 onTouchEvent
                └─ 返回 false → 继续调用 View.onTouchEvent(event)
```

“消费事件”是声明当前事件已经由这个处理者处理。监听器和 onTouchEvent() 是不同的处理入口，但调用关系并不独立：监听器的返回值会影响同一个 View 的 onTouchEvent() 是否处理当前事件。注册监听器不会替换或重写 onTouchEvent()，监听器只是获得优先处理事件的机会。

## 监听按下事件的最小示例

以下函数为一个已有 Button 注册触摸监听器。animator 是调用方传入的已有 ValueAnimator。用户按下按钮时，监听器取消动画，然后返回 false，让按钮继续处理默认触摸行为。

```kotlin
import android.animation.ValueAnimator
import android.view.MotionEvent
import android.widget.Button

fun cancelAnimationOnPress(button: Button, animator: ValueAnimator) {
    button.setOnTouchListener { _, event ->
        if (event.actionMasked == MotionEvent.ACTION_DOWN) {
            animator.cancel()
        }
        false
    }
}
```

Android 把触摸事件传入监听器。监听器读取 event.actionMasked，遇到 ACTION_DOWN 时取消动画。由于监听器返回 false，Button.onTouchEvent() 仍可以处理按下状态，并在满足点击条件时触发点击行为。

## 返回值、后续事件与取消处理

OnTouchListener 返回 false，表示监听器没有消费当前事件，Android 继续调用这个 View 的 onTouchEvent()；这不等于直接把事件交给父容器。监听器返回 true，表示当前事件已由监听器消费，该 View 的默认 onTouchEvent() 不再处理当前事件。

如果监听器和 View 自身都不处理 ACTION_DOWN，这个 View 通常不会继续收到同一触摸序列的后续事件。如果自行实现拖动，通常需要在 ACTION_DOWN 时接收触摸，记录起点，再处理 ACTION_MOVE 的位移，并在 ACTION_UP 时结束拖动。

接收 ACTION_DOWN 不保证后续触摸永远归当前 View 处理。父容器仍可能中途接管触摸，因此还需要在 ACTION_CANCEL 时清理拖动状态、取消尚未完成的操作，且不执行依赖正常抬起的点击动作。

## 与 UIKit 触摸回调的对照

按单指触摸的阶段类比，ACTION_DOWN 接近 UIKit 的 touchesBegan，ACTION_MOVE 接近 touchesMoved，ACTION_UP 接近 touchesEnded，ACTION_CANCEL 接近 touchesCancelled。类比只说明开始、移动、结束和取消四个阶段，不表示两端的事件分发和多指处理规则完全相同。
