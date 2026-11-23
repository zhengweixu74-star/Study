# UIKit 触摸事件、手势与键盘输入

[返回笔记目录](../README.md)

## 本文目录

- [触摸事件的分发路径](#触摸事件的分发路径)
- [命中测试](#命中测试)
  - [UIView 的局部坐标与 bounds](#uiview-的局部坐标与-bounds)
  - [point(inside:with:) 与 hitTest(\_:with:)](#pointinsidewith-与-hittest_with)
  - [事件穿透与重叠视图的触摸目标](#事件穿透与重叠视图的触摸目标)
  - [扩大按钮的命中区域](#扩大按钮的命中区域)
- [响应链与触摸回调](#响应链与触摸回调)
- [手势识别](#手势识别)
  - [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)
  - [同一按钮的点击与长按协调](#同一按钮的点击与长按协调)
  - [长按状态与取消后的恢复](#长按状态与取消后的恢复)
- [屏幕键盘：触摸位置与文字接收对象](#屏幕键盘触摸位置与文字接收对象)

## 触摸事件的分发路径

事件分发是把一个事件交给合适的对象处理。UIApplication 是代表整个应用的对象，UIWindow 是承载视图层级的根容器，两者的职责见 [UIApplication、AppDelegate、UIWindow 与 UIView 的职责](app-lifecycle-and-events.md#uiapplicationappdelegateuiwindow-与-uiview-的职责)。两者都参与触摸事件的分发：UIApplication 在应用内选择窗口，UIWindow 在窗口内选择视图。

```text
系统 → UIApplication → 触摸所在的 UIWindow → 窗口内的目标 UIView
```

找到目标视图，以及找到之后继续传递事件，分别由两套机制负责：

- 命中测试：UIKit 根据触摸位置和视图层级，找出触摸落在哪个视图上。命中测试找到的视图是第一个接收这次触摸的视图，称为目标视图。
- 响应链：由每个响应对象的 `next` 属性串起来的一条链。目标视图不继续处理的事件，可以沿响应链传给下一个对象。

手势识别器也会收到触摸，而且早于视图的触摸回调，见 [父子视图的手势识别与同时识别](#父子视图的手势识别与同时识别)。

## 命中测试

### UIView 的局部坐标与 bounds

命中测试使用的坐标都是视图自己的局部坐标。局部坐标系是以视图自身为参照的坐标系。默认情况下，`bounds.origin` 为 (0, 0)，视图可见区域的左上角就是局部坐标 (0, 0)，x 轴向右，y 轴向下，单位是点（pt）。

- `bounds` 是视图在自身坐标系中的矩形。
- `frame` 是视图在父视图坐标系中的矩形。

例如，按钮的 `frame` 是 `CGRect(x: 100, y: 200, width: 50, height: 50)`，`bounds` 是 `CGRect(x: 0, y: 0, width: 50, height: 50)`。在按钮的局部坐标系中，中心点是 (25, 25)，右下角是 (50, 50)；按钮的 `center` 使用父视图坐标系，值是 (125, 225)。

修改 `bounds.origin` 后，可见区域左上角的局部坐标随之改变，`UIScrollView` 滚动时改变的就是 `bounds.origin`。

### point(inside:with:) 与 hitTest(\_:with:)

两个方法的分工不同：

- `point(inside:with:)` 判断一个局部坐标点是否在当前视图的命中区域内，返回 `Bool`。命中区域是能让当前视图成为目标视图的范围，默认就是 `bounds`。
- `hitTest(_:with:)` 在当前视图及其所有子视图中找出目标视图，返回 `UIView?`；找不到时返回 `nil`。

UIWindow 收到触摸后，从自己开始调用 `hitTest(_:with:)`。UIView 默认的 `hitTest(_:with:)` 按以下顺序执行：

1. 当前视图被隐藏（`isHidden` 为 true）、关闭了用户交互（`isUserInteractionEnabled` 为 false），或 `alpha` 小于 0.01 时，直接返回 `nil`。
2. 调用当前视图的 `point(inside:with:)`。结果为 false 时，直接返回 `nil`，当前视图的子视图都不再检查。
3. 从最上层的子视图开始，依次检查子视图：先把触摸点转换成子视图的局部坐标，再调用子视图的 `hitTest(_:with:)`。最上层的子视图是 `subviews` 数组中的最后一个，所以检查顺序是数组的倒序。
4. 某个子视图返回非 `nil` 的结果时，当前视图返回这个结果，不再检查其余子视图。所有子视图都返回 `nil` 时，当前视图返回自己。

例如，窗口中有一个容器视图，容器视图中有一个按钮，用户点击按钮。容器视图的 `point(inside:with:)` 返回 true，容器视图接着检查按钮；按钮的 `point(inside:with:)` 返回 true，按钮没有子视图，于是返回自己。容器视图和窗口依次把按钮作为结果返回，按钮成为目标视图。

两个方法接收的 `point` 都是当前视图的局部坐标，所以重写命中区域时，要把 `point` 与同样使用局部坐标的 `bounds` 比较。目标视图在触摸开始时确定，之后这次触摸从开始到结束都交给这个目标视图，手指移动到其他视图上也一样。

**让容器接管子视图的触摸。** 重写容器的 `hitTest(_:with:)`，把原本返回子视图的结果改为返回容器自己：

```swift
final class InterceptingContainer: UIView {
    var interceptsTouches = false

    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        guard let target = super.hitTest(point, with: event) else {
            return nil
        }
        return interceptsTouches ? self : target
    }
}
```

`super.hitTest(point, with: event)` 先按默认规则查找目标，保留隐藏、交互关闭、命中区域等默认判断。`super` 返回 `nil` 时，`InterceptingContainer` 也返回 `nil`。`super` 返回非 `nil` 且 `interceptsTouches` 为 true 时，容器返回 `self`，子控件收不到触摸。容器成为目标视图后，要自己通过触摸方法或手势识别器执行动作；祖先视图上的手势识别器照样会收到这次触摸。

### 事件穿透与重叠视图的触摸目标

事件穿透是让上层视图在某些位置不成为目标视图，于是命中测试继续检查下层的视图。命中测试直接选中下层视图，上层视图从头到尾收不到这次触摸。

例如，父视图中有按钮 B，视图 A 盖在 B 上面，A 在 `subviews` 中排在 B 后面。父视图先检查 A：A 的 `hitTest(_:with:)` 返回 A 或 A 的子视图时，目标已经确定，B 收不到触摸；A 的 `hitTest(_:with:)` 返回 `nil` 时，父视图才接着检查 B。命中测试只看 `hitTest(_:with:)` 的返回值，所以背景透明的 A 仍会挡住触摸。

**方案一：空白处穿透，子按钮照常响应。**

```swift
final class PassthroughOverlay: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let target = super.hitTest(point, with: event)
        return target === self ? nil : target
    }
}
```

用户点击覆盖层中的按钮时，`super` 返回按钮，覆盖层原样返回按钮。用户点击覆盖层的空白处时，`super` 返回覆盖层自己，覆盖层改为返回 `nil`，父视图接着检查下层视图。覆盖层中只用于展示的子视图（例如一个装饰用的 UIView 色块）仍会挡住触摸，要把这些子视图的 `isUserInteractionEnabled` 设为 false。

**方案二：指定一个完全穿透的矩形区域。**

```swift
final class HoleOverlay: UIView {
    var passthroughRect: CGRect = .zero

    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        if passthroughRect.contains(point) {
            return false
        }
        return super.point(inside: point, with: event)
    }
}

let overlay = HoleOverlay()
overlay.passthroughRect = CGRect(x: 40, y: 100, width: 120, height: 60)
```

`passthroughRect` 使用 `overlay` 自己的坐标系。初始值 `CGRect.zero` 的宽高都为 0，`contains(_:)` 对任何点都返回 false，所以初始状态下没有穿透区域。设置后，局部点 (60, 120) 落在矩形内，`point(inside:with:)` 返回 false，命中测试跳过整个覆盖层及其子视图，触摸穿透到下层。孔洞范围内的覆盖层子按钮因此也收不到触摸；子按钮需要响应时，改用方案一。孔洞要跟随某个视图移动时，在布局变化后用 `overlay.convert(button.bounds, from: button)` 重新计算矩形。

### 扩大按钮的命中区域

重写 `point(inside:with:)` 可以扩大按钮的命中区域，按钮的显示尺寸和布局尺寸保持不变：

```swift
final class ExpandedHitButton: UIButton {
    override func point(
        inside point: CGPoint,
        with event: UIEvent?
    ) -> Bool {
        let hitBounds = bounds.insetBy(dx: -10, dy: -10)
        return hitBounds.contains(point)
    }
}
```

`bounds.insetBy(dx: -10, dy: -10)` 中的负数表示向外扩展。`bounds` 为 50×50 pt 时，命中区域是 `CGRect(x: -10, y: -10, width: 70, height: 70)`。

扩大的区域只有落在父视图范围内的部分才有效。命中测试从父视图向子视图逐层进行：触摸点在父视图范围之外时，父视图的 `point(inside:with:)` 返回 false，父视图直接返回 `nil`，按钮的 `point(inside:with:)` 没有被调用。这时还要扩大父视图的命中区域，或者重写父视图的 `hitTest(_:with:)` 把触摸转给按钮。同样的原因，父视图的 `clipsToBounds` 设为 false 后，子视图超出父视图的部分能显示出来，但落在这部分的触摸会被父视图排除。

## 响应链与触摸回调

触摸序列是一次触摸从按下、移动到抬起或取消的完整过程。触摸回调是 UIKit 在触摸序列各阶段调用的方法：`touchesBegan(_:with:)`、`touchesMoved(_:with:)`、`touchesEnded(_:with:)`、`touchesCancelled(_:with:)` 分别对应按下、移动、抬起和取消。

`UIResponder` 是所有能接收事件的对象的基类，UIView、UIViewController、UIWindow 和 UIApplication 都继承自 `UIResponder`。`next` 属性指向响应链中的下一个响应对象：普通视图的 `next` 是父视图；视图控制器根视图的 `next` 是这个视图控制器，事件到达视图控制器后再继续向上传递。

触摸回调只交给目标视图。父视图要收到目标视图的触摸回调，必须由目标视图转发：调用会转发的父类实现，或者显式调用 `next?.touchesBegan(touches, with: event)`。触摸方法没有返回值，UIKit 无法根据返回值判断事件是否“已经处理”。

`super` 指继承关系上的父类，`next` 指响应链上的下一个对象。`super.touchesBegan(touches, with: event)` 执行的是当前类的父类实现。`UIResponder` 的默认实现会把事件转给 `next`；父类是 `UIButton` 等控件或自定义类时，父类实现可能自己处理触摸并停止转发。

| 子视图的实现方式 | 对这个触摸回调的影响 |
|-|-|
| 重写触摸方法，不调用 `super`，也不显式转发 | 回调停在子视图 |
| 执行自定义逻辑后调用 `super` | 是否继续传递由父类实现决定 |
| 不重写触摸方法 | 使用继承来的实现；继承的是默认转发行为时，回调会继续传递 |

四个触摸方法要成对重写。重写 `touchesBegan(_:with:)` 且不调用 `super` 时，`touchesMoved(_:with:)`、`touchesEnded(_:with:)`、`touchesCancelled(_:with:)` 也要重写且不调用 `super`，即使方法体为空。只在 `touchesBegan(_:with:)` 中不转发时，后续三个阶段会通过继承来的默认实现转发出去，下一个响应对象收到的触摸序列缺少开始阶段。

## 手势识别

### 父子视图的手势识别与同时识别

`UIGestureRecognizer` 是手势识别器，根据触摸序列判断用户做了点击、拖动、长按等哪种手势。一个手势识别器同一时刻只能添加在一个视图上；把识别器添加到另一个视图时，识别器会从原来的视图上移除。

目标视图上的识别器，以及目标视图所有祖先视图上的识别器，都会收到这次触摸。祖先视图是沿 `superview` 逐层向上的各层父视图。与目标视图在画面上重叠的兄弟视图位于另一条视图分支上，兄弟视图上的识别器收不到这次触摸。

UIKit 先把触摸交给这些识别器，再把触摸回调交给视图。所以子视图在 `touchesBegan(_:with:)` 中是否调用 `super`，对父视图上的识别器没有影响。

例如，`parentView` 包含 `childView`，两者各有一个拖动识别器 `parentPan` 和 `childPan`。手指落在 `childView` 上时，两个识别器都会收到触摸，并各自判断手势；默认情况下，其中只有一个能识别成功。

同时识别是指允许两个识别器在同一次触摸中都识别成功。UIKit 会调用识别器代理的 `gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)`，询问两个识别器能否同时识别。下面的方法写在视图控制器中：视图控制器遵循 `UIGestureRecognizerDelegate`，用属性保存 `parentPan` 和 `childPan`，并把两个识别器的 `delegate` 都设为视图控制器。

```swift
func gestureRecognizer(
    _ gestureRecognizer: UIGestureRecognizer,
    shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
) -> Bool {
    (gestureRecognizer === parentPan && otherGestureRecognizer === childPan)
        || (gestureRecognizer === childPan && otherGestureRecognizer === parentPan)
}
```

返回 true 只表示允许 `parentPan` 和 `childPan` 同时识别，每个识别器仍要满足自己的识别条件，例如拖动距离。用 `require(toFail:)` 设置的“等另一个识别器失败后才能成功”的依赖同样生效。两个识别器都识别成功后，各自调用自己的动作方法，分别移动父视图和子视图。

### 同一按钮的点击与长按协调

按钮的点击用 `.touchUpInside` 事件处理，`.touchUpInside` 表示手指在按钮范围内抬起。长按用 `UILongPressGestureRecognizer` 识别。长按识别器添加在按钮上时，只有从按钮范围内（包括扩大后的命中区域）开始的按压才会被识别为长按；添加在页面根视图上时，从根视图任何位置开始的按压都可能被识别为长按。

以下代码写在 `UIViewController` 子类中。`button` 是已经创建好的 `UIButton` 属性，`setupInteractions()` 在界面初始化时调用一次。

```swift
private func setupInteractions() {
    button.addTarget(self, action: #selector(handleTap), for: .touchUpInside)

    let longPress = UILongPressGestureRecognizer(
        target: self,
        action: #selector(handleLongPress(_:))
    )
    longPress.minimumPressDuration = 0.5
    longPress.cancelsTouchesInView = true
    button.addGestureRecognizer(longPress)
}

@objc private func handleTap() {
    print("短按完成")
}

@objc private func handleLongPress(_ gesture: UILongPressGestureRecognizer) {
    switch gesture.state {
    case .began:
        print("长按开始")
    case .ended, .cancelled:
        print("长按结束，恢复临时状态")
    default:
        break
    }
}
```

用户的操作有两种结果：

- 按住不到 0.5 秒就在按钮上松手：长按识别失败，按钮触发 `.touchUpInside`，执行 `handleTap()`。
- 按住达到 0.5 秒，且手指移动没有超过允许距离：长按识别成功，进入 `.began` 状态，执行 `handleLongPress(_:)`。

`cancelsTouchesInView` 为 true 时，长按识别成功后，UIKit 取消按钮收到的触摸，按钮收到 `touchesCancelled(_:with:)`，所以这次松手不会再触发 `.touchUpInside`。长按识别器继续跟踪触摸，松手时进入 `.ended`。

如果按钮的动作绑定在 `.touchDown`（手指按下就触发）上，按下时动作已经执行，长按识别成功后再取消触摸，也撤销不了已经执行的动作。

### 长按状态与取消后的恢复

`UILongPressGestureRecognizer` 是连续手势识别器：识别成功后，识别器继续跟踪触摸，在手指移动、抬起或触摸被取消时再次调用动作方法。动作方法通过 `gesture.state` 区分当前阶段。

| 状态 | 含义 | 适合执行的操作 |
|-|-|-|
| `.began` | 达到长按条件，长按识别成功 | 保存原状态，开启临时效果 |
| `.changed` | 长按已经开始，手指位置等触摸信息发生变化 | 更新连续交互，例如跟随手指移动 |
| `.ended` | 已识别的长按正常结束，例如用户松手 | 恢复原状态，结束临时效果 |
| `.cancelled` | 已识别的长按被中断 | 同样恢复原状态并清理 |
| `.failed` | 识别成功之前条件就不满足，例如按住时间不够就松手 | 不执行只属于长按成功的操作 |

识别器只在触摸被系统实际中断时进入 `.cancelled`。例如，长按开始后来电界面弹出并接管了交互，UIKit 取消当前触摸，识别器进入 `.cancelled`。

长按开始后，手指移出按钮，长按仍会继续，识别器保持在 `.changed` 状态。需要“移出按钮就结束”时，在 `.changed` 状态中用 `gesture.location(in: button)` 读取手指位置，自己判断手指是否已经离开 `button.bounds`。

临时效果（例如长按时把按钮放大）要在 `.began` 时保存原状态，在 `.ended` 和 `.cancelled` 时都恢复。只在 `.ended` 中恢复的话，长按被取消时临时效果会一直留在界面上。

## 屏幕键盘：触摸位置与文字接收对象

第一响应者是当前优先接收键盘输入等事件的响应对象。在文字输入中，成为第一响应者的输入框就是文字的输入目标；触摸事件仍按触摸位置分发。

假设页面同时显示用户名框和密码框，用户使用屏幕键盘输入：

1. 用户点击密码框。UIKit 根据触摸位置找到密码框，密码框成为第一响应者，系统显示屏幕键盘。
2. 用户点击屏幕键盘上的 A 键。这次触摸有坐标，键盘用这个坐标判断用户按了哪个键。
3. 系统把字母 A 交给当前的第一响应者，也就是密码框。
4. 用户点击用户名框，用户名框成为第一响应者。用户再按同一个 A 键，字母 A 就输入到用户名框。键盘位置没变，变的是第一响应者。

所以，点击屏幕键盘时的触摸坐标只用来确定按键；文字写到哪个输入框，由第一响应者决定。外接键盘的按键事件没有触摸坐标，文字同样交给第一响应者。

关键窗口决定键盘输入交给哪个窗口，第一响应者决定交给窗口中的哪个对象。标准文本输入控件只有在所在窗口是关键窗口、并且自己成为第一响应者时，才能正常弹出屏幕键盘。关键窗口的设置方式见 [关键窗口与 makeKeyAndVisible()](app-lifecycle-and-events.md#关键窗口与-makekeyandvisible)。
