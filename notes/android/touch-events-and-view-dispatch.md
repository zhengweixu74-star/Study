# Android 触摸事件与 View 事件处理

[返回 Android 分类目录](README.md)

本文说明传统 Android View 体系中的触摸事件、ACTION_DOWN 的用途，OnTouchListener 与 onTouchEvent() 的调用关系，ViewGroup 如何拦截子 View 的触摸事件，以及一次触摸从窗口到 View 的完整分发路径。View 与 ViewGroup 的基础职责见 [Android View、ViewGroup 与 XML 布局](view-system-and-layout.md)。

## MotionEvent 与触摸序列

`MotionEvent` 是 Android 传递触摸信息的对象，包含事件类型、触摸坐标和时间等信息。`MotionEvent.ACTION_DOWN` 是事件类型常量，表示一次触摸序列开始。`event.actionMasked` 只返回事件类型，去掉了多指触摸中的触点索引信息。

| 事件 | 含义 |
| --- | --- |
| ACTION_DOWN | 第一个触点按下，触摸序列开始。 |
| ACTION_MOVE | 触点移动，可以发生多次，也可以没有。 |
| ACTION_UP | 最后一个触点抬起，触摸序列正常结束。 |
| ACTION_CANCEL | 当前接收对象的触摸处理被取消；接收对象要清理状态，并按未完成的点击处理。 |

一次单指触摸通常按 ACTION_DOWN → 零次或多次 ACTION_MOVE → ACTION_UP 的顺序发生。当前 View 中途失去这次触摸时（例如父容器接管了触摸），当前 View 收到 ACTION_CANCEL。多指触摸中，第二根及之后的手指按下时，通常发送 ACTION_POINTER_DOWN。

ACTION_DOWN 只表示手指按下。用户随后可能抬起完成点击，也可能拖动或取消。普通点击用 setOnClickListener() 处理；只有需要在手指刚按下时执行动作，才直接判断 ACTION_DOWN。

## ACTION_DOWN 的常见用途

| 用途 | 按下时执行的动作 |
| --- | --- |
| 记录触摸起点 | 保存 event.x 和 event.y，供后续计算位移；这两个值使用接收事件的 View 的局部坐标，单位为像素。 |
| 初始化交互状态 | 记录开始时间，初始化本次拖动所需的状态。 |
| 停止已有动画 | 立即取消正在执行的自动动画，让后续交互响应用户操作。 |

按下时只执行需求要求的动作。例如只需停止动画时，监听器只取消动画，拖动和点击识别留给其他代码或默认行为。动画的取消与结束回调规则见 [ValueAnimator、插值器与滚动动画](valueanimator-and-interpolation.md#启动取消与结束回调)。

## OnTouchListener 与 onTouchEvent 的调用关系

`OnTouchListener` 是通过 setOnTouchListener() 给 View 注册的外部回调。Android 调用监听器的 onTouch(view, event)，传入视图和事件。`onTouchEvent(event)` 是 View 自身处理触摸事件的方法，系统控件有默认实现，自定义 View 可以重写该方法。

`dispatchTouchEvent(event)` 是 View 分发触摸事件的方法，决定调用哪个处理入口。下面的流程适用于已经收到事件、处于启用状态、使用默认分发逻辑的 View，并省略了与本主题无关的过滤步骤：

```text
View.dispatchTouchEvent(event)
    ↓
是否注册了 OnTouchListener？
    ├─ 没有 → 调用 View.onTouchEvent(event)
    └─ 有 → 调用监听器的 onTouch(view, event)
                ├─ 返回 true → 该事件由监听器消费，不再调用 onTouchEvent
                └─ 返回 false → 继续调用 View.onTouchEvent(event)
```

消费事件指处理者返回 true，声明当前事件已经处理完毕。监听器和 onTouchEvent() 是两个处理入口，监听器先执行，监听器的返回值决定同一个 View 的 onTouchEvent() 是否还会收到当前事件。注册监听器后，onTouchEvent() 的实现保持不变，监听器只是获得优先处理事件的机会。

## 监听按下事件的最小示例

以下函数给 Button 注册触摸监听器，animator 是调用方传入的 ValueAnimator。用户按下按钮时，监听器取消动画，然后返回 false，让按钮继续执行默认的触摸处理。

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

OnTouchListener 返回 false，表示监听器没有消费当前事件，Android 接着调用同一个 View 的 onTouchEvent()。监听器返回 true，表示监听器已经消费当前事件，这个 View 的 onTouchEvent() 不再收到当前事件。

监听器和 onTouchEvent() 都不消费 ACTION_DOWN 时，这个 View 通常收不到同一触摸序列的后续事件。所以自行实现拖动时，通常要在 ACTION_DOWN 时消费事件并记录起点，在 ACTION_MOVE 中计算位移，在 ACTION_UP 时结束拖动。

当前 View 消费 ACTION_DOWN 后，父容器仍可能中途接管触摸，接管方式见下文[ViewGroup 拦截事件：onInterceptTouchEvent](#viewgroup-拦截事件onintercepttouchevent)。所以 View 还要在收到 ACTION_CANCEL 时清理拖动状态、取消尚未完成的操作，并跳过只应在正常抬起时执行的点击动作。

## ViewGroup 拦截事件：onInterceptTouchEvent

触摸事件从 Activity 的 Window 开始，沿 View 树从父 View 向子 View 逐层传递，完整路径见下文[一次触摸的完整分发路径](#一次触摸的完整分发路径)。ViewGroup 收到事件后，先调用自己的 `onInterceptTouchEvent(event)` 决定是否拦截：

```text
ViewGroup.dispatchTouchEvent(event)
    ↓
调用 onInterceptTouchEvent(event)
    ├─ 返回 false → 把事件交给手指所在位置的子 View
    └─ 返回 true  → 拦截事件，交给 ViewGroup 自己的 onTouchEvent(event) 处理
```

ViewGroup 在手指已经按在子 View 上之后才开始拦截时，系统先向这个子 View 发送 `ACTION_CANCEL`，之后同一触摸序列的事件都交给 ViewGroup 处理。

例如 ScrollView 中放了一个按钮：

1. 手指按下时，ScrollView 的 `onInterceptTouchEvent()` 返回 false，按钮收到 `ACTION_DOWN`，显示按下状态。
2. 手指开始滑动，移动距离超过系统的滑动阈值后，ScrollView 的 `onInterceptTouchEvent()` 返回 true。
3. 按钮收到 `ACTION_CANCEL`，取消按下状态，不触发点击。
4. 后续的 `ACTION_MOVE` 交给 ScrollView 的 `onTouchEvent()`，ScrollView 开始滚动。

子 View 可以调用 `parent.requestDisallowInterceptTouchEvent(true)`，请求父 View 在当前触摸序列中不再拦截。例如可以横向拖动的控件放在纵向列表中，控件确认用户在横向拖动后，调用这个方法阻止列表接管事件。

UIKit 中，滚动视图与按钮之间的冲突通过手势识别器的识别和取消规则解决，见[UIKit 触摸事件、手势与键盘输入](../ios/touch-events-and-gestures.md)。

## 一次触摸的完整分发路径

以 Activity → DecorView → 内容区域 → LinearLayout → Button 的结构为例，手指按下按钮时，ACTION_DOWN 按以下顺序传递：

```text
系统按触摸坐标和窗口层级选出 Activity 的窗口，把事件交给窗口
→ DecorView 把事件交给 Activity.dispatchTouchEvent()
   （Activity 可以在这里处理整个页面的触摸）
→ Activity 把事件交回窗口，DecorView 以 ViewGroup 身份开始分发
→ 每个 ViewGroup 的 dispatchTouchEvent() 依次执行三步：
     ① 调用 onInterceptTouchEvent()，返回 false 表示不拦截
     ② 从最上层的子 View 开始，查找触摸坐标落在哪个子 View 范围内
     ③ 调用这个子 View 的 dispatchTouchEvent()
   DecorView → 内容区域 → LinearLayout 都执行这三步
→ Button.dispatchTouchEvent() 按上文的监听器与 onTouchEvent() 规则处理，
   Button 的 onTouchEvent() 显示按下状态并返回 true
```

手指抬起时，Button 的 `onTouchEvent()` 判定一次完整点击，调用 `OnClickListener`。窗口的作用见[窗口的作用：显示、输入与窗口规则](activity-fragment-and-navigation.md#窗口的作用显示输入与窗口规则)。

目标 View 不处理事件时，事件从子 View 向父 View 逐层退回。例如把 Button 换成不可点击的 TextView：

1. TextView 的 `onTouchEvent()` 返回 false。
2. LinearLayout 的 `dispatchTouchEvent()` 收到 false，转而调用 LinearLayout 自己的 `onTouchEvent()`。
3. LinearLayout 也返回 false 时，再交给 LinearLayout 的父 View，依次向上。
4. 所有 View 都不处理时，最后调用 `Activity.onTouchEvent()`。

ACTION_DOWN 决定整个触摸序列的接收者。ViewGroup 记录在 ACTION_DOWN 时返回 true 的子 View；同一序列后续的 ACTION_MOVE 和 ACTION_UP 不再按坐标查找子 View，直接交给记录的子 View。所以手指按在按钮上再滑出按钮范围，ACTION_MOVE 仍然交给按钮。父 ViewGroup 每次转发后续事件前仍会调用 `onInterceptTouchEvent()`，中途拦截的规则见上文。ACTION_DOWN 时没有任何子 View 返回 true，同一序列的后续事件就不再分发给这些子 View。

### onInterceptTouchEvent 与 onTouchEvent 的分工

`onTouchEvent()` 的返回值决定 View 是否处理当前事件。`onInterceptTouchEvent()` 决定 ViewGroup 是否把事件留给自己、不再交给子 View；ViewGroup 拦截后，是否处理事件仍由 ViewGroup 自己的 `onTouchEvent()` 决定。

| 方法 | 所属对象 | 回答的问题 | 返回 true | 返回 false |
| --- | --- | --- | --- | --- |
| `onInterceptTouchEvent()` | 只有 ViewGroup | 事件还要不要交给子 View？ | 不再交给子 View，改为调用自己的 `onTouchEvent()` | 继续交给触摸位置下的子 View |
| `onTouchEvent()` | 所有 View | 当前 View 是否处理事件？ | 处理事件，事件不再向上退回 | 不处理，事件交回父 View 的 `onTouchEvent()` |

例如某个自定义容器在 ACTION_DOWN 时让 `onInterceptTouchEvent()` 返回 true，同时自己的 `onTouchEvent()` 返回 false：

1. 容器拦截事件，容器内的子 View 收不到这次触摸。
2. 容器的 `onTouchEvent()` 不处理事件。
3. 事件交回容器的父 View，由父 View 的 `onTouchEvent()` 决定是否处理。

## 与 UIKit 触摸回调的对照

按单指触摸的阶段类比，ACTION_DOWN 接近 UIKit 的 touchesBegan，ACTION_MOVE 接近 touchesMoved，ACTION_UP 接近 touchesEnded，ACTION_CANCEL 接近 touchesCancelled。两端只在开始、移动、结束和取消四个阶段上对应，事件分发和多指处理规则各不相同。

两端分发规则的对照如下，UIKit 的命中测试和响应链见[UIKit 触摸事件、手势与键盘输入](../ios/touch-events-and-gestures.md)：

| 对比项 | UIKit | Android |
| --- | --- | --- |
| 查找目标视图 | 先用 `hitTest(_:with:)` 从窗口一次找到最深的视图，再投递事件 | 没有单独的查找阶段；ACTION_DOWN 逐层分发，分发过程中确定目标 |
| 表示已处理 | `touchesBegan` 中不再调用 `super` | `onTouchEvent()` 返回 true |
| 未处理的事件 | 沿响应链交给父视图、控制器、窗口、`UIApplication` | 交回父 View 的 `onTouchEvent()`，最后到 `Activity.onTouchEvent()` |
| 按下时由父视图接收 | 父视图重写 `hitTest(_:with:)` 返回自己 | 父 ViewGroup 在 ACTION_DOWN 时让 `onInterceptTouchEvent()` 返回 true |
| 父视图中途接管 | 手势识别器识别成功后取消视图的触摸，视图收到 `touchesCancelled` | 父 ViewGroup 在后续事件中让 `onInterceptTouchEvent()` 返回 true，子 View 收到 ACTION_CANCEL |
| 手势识别 | 系统的 `UIGestureRecognizer` 先于视图收到触摸 | 没有系统级手势识别器；`GestureDetector` 是工具类，需要在 `onTouchEvent()` 中手动传入事件 |
