# Core Animation 渲染与性能分析工具

[返回笔记目录](../README.md)

## 本文目录

- [图层动画与渲染管线](#图层动画与渲染管线)
  - [UIView 与 CALayer 的对应关系](#uiview-与-calayer-的对应关系)
  - [多个子图层的用途与形状](#多个子图层的用途与形状)
  - [图层重用时的状态管理](#图层重用时的状态管理)
  - [UIViewPropertyAnimator 的正向、反向与暂停](#uiviewpropertyanimator-的正向反向与暂停)
- [anchorPoint、position 与图层几何](#anchorpointposition-与图层几何)
- [圆角、边界裁剪与离屏渲染](#圆角边界裁剪与离屏渲染)
  - [cornerRadius 与 masksToBounds 的职责](#cornerradius-与-maskstobounds-的职责)
  - [clipsToBounds 与 masksToBounds 的关系](#clipstobounds-与-maskstobounds-的关系)
  - [离屏渲染的含义与常见场景](#离屏渲染的含义与常见场景)
  - [shouldRasterize 与位图缓存](#shouldrasterize-与位图缓存)
  - [静态图片的圆角预处理](#静态图片的圆角预处理)
  - [shadowPath 与圆角阴影的组合](#shadowpath-与圆角阴影的组合)
  - [性能判断与优化边界](#性能判断与优化边界)
- [回调 FPS 与卡顿监测](#回调-fps-与卡顿监测)
  - [CADisplayLink 帧时间与卡顿时长统计](#cadisplaylink-帧时间与卡顿时长统计)
- [Time Profiler 的采样与统计边界](#time-profiler-的采样与统计边界)
- [Animation Hitches 的使用流程](#animation-hitches-的使用流程)
  - [同次录制中的时间线关联](#同次录制中的时间线关联)
- [MetricKit 与 Xcode Organizer](#metrickit-与-xcode-organizer)

## 图层动画与渲染管线

**与 UIView 动画的关系：** UIView 动画是 UIKit 提供的高层接口，由 Core Animation 的图层动画实现。每个 UIView 背后都有一个 CALayer。UIView 主要管理[视图层级](uikit-layout-and-mvvm.md#视图层级与父子关系)、布局与交互；CALayer 保存位置、大小、透明度、圆角等视觉状态；Core Animation 根据动画配置计算中间状态，并参与渲染与合成。UIView 自身也提供绘制和动画 API，所以 UIView 与 CALayer 的职责有部分重叠。

例如让视图在 0.3 秒内淡出：使用 UIKit 时，在 `UIView.animate` 中把 `view.alpha` 设为 `0`；直接使用 Core Animation 时，为图层的 `opacity` 创建并添加 `CABasicAnimation`，同时把模型属性设为最终值。使用 UIKit 时，UIKit 为图层创建对应的动画，业务代码不直接创建 `CAAnimation` 对象。两种写法都由 Core Animation 计算中间状态，业务代码不需要逐帧修改透明度。

**动画原理：** CALayer 管理视觉内容，以及位置、变换、透明度等属性。应用描述起点和终点、时长、关键帧或弹簧规则，系统按时间计算中间状态。位移、缩放、淡出只改变图层属性，通常能复用已有的位图／纹理，不必每帧重画内容；改变图层内容本身（例如重绘文字或图片）仍需要 CPU 重新绘制。

- 模型树保存代码设置的状态；呈现树反映动画中的当前状态，可通过 presentation() 读取；渲染树是系统内部结构。
- 简单数值属性的当前值 = 起点 + 差值 × 经时间曲线换算后的进度。系统按当前时间直接算出当前值，不是每次回调累加一段固定距离。
- CABasicAnimation、CAKeyframeAnimation、CASpringAnimation 和 CAAnimationGroup 分别表达基本插值、关键帧／路径、弹簧与动画组合。
- 显式添加 CAAnimation 只影响呈现树，模型属性保持原值。代码需要把模型属性设为预期的最终值，否则动画移除后图层回到原状态。UIView.animate 是更上层的接口：在动画块内修改的普通 UIView 属性才会产生过渡，在动画块外修改通常立即生效。
- CATransaction 把一批图层修改合并为一次提交。代码没有显式创建事务时，系统创建隐式事务，并借助 RunLoop 在本轮循环末尾自动提交。事务负责提交修改，动画类型负责描述变化方式；显式动画同样要经过事务提交。

**典型渲染管线：**

| 阶段 | 位置与工作 |
|-|-|
| Event | App 处理触摸、Timer、数据回调，修改视图／图层状态 |
| Commit | App 完成必要的布局、内容绘制，整理并提交图层变化 |
| Render Prepare | Render Server 的 CPU 处理图层和效果，准备 GPU 命令与资源 |
| Render Execute | GPU 完成绘制、采样、变换、混合等工作，生成画面 |
| Display | 显示系统在相应时机呈现已准备好的画面 |

App 在 Commit 阶段调用 `draw(_:)` 生成图层内容；之后渲染服务与 GPU 按图层属性把这些内容合成为最终画面。内容绘制和 GPU 合成属于不同阶段。Render Prepare 阶段在 Render Server 的 CPU 上执行，因此 Core Animation 的渲染过程同样使用 CPU。

流水线的各阶段可以同时处理不同的帧：App 准备下一次更新时，GPU 可以在处理之前提交的内容。60／120 Hz 对应约 16.67／8.33 ms 的显示间隔。一次更新从输入到显示要依次经过多个阶段，通常跨越不止一个显示间隔；具体延迟几帧，因设备和系统而异。

Commit hitch 指 App 没有按时完成事件处理、布局／绘制和提交；Render hitch 指渲染准备或渲染执行超时。主线程流畅时，渲染服务或 GPU 仍可能成为瓶颈。新画面没有按时准备好时，屏幕继续显示旧画面。RunLoop 是单个线程的事件循环；Render Loop 是从 App 提交、经渲染服务的 CPU 与 GPU、到最终显示的跨进程完整流程。

统计 FMP（首次有效绘制，指页面主要内容第一次显示到屏幕上的时刻）时，`willDisplay` 回调、布局完成和事务提交完成都只说明 App 侧的工作已完成；内容真正显示在屏幕上，还要经过渲染和显示阶段。

### UIView 与 CALayer 的对应关系

`CALayer` 是 Core Animation 中保存视觉内容和几何、外观属性的对象。支撑图层是 UIKit 为 `UIView` 创建并管理的图层，通过 `view.layer` 访问。每个 `UIView` 有一个支撑图层；支撑图层可以包含多个子图层。

子图层是加入另一个图层内部的 `CALayer` 对象。`superlayer` 表示直接父图层，`sublayers` 表示直接子图层数组。调用 `view.layer.addSublayer(layer)` 把 `layer` 添加为子图层。每个 `UIView` 对应一个支撑图层；反过来，代码额外创建的 `CALayer` 可以没有对应的 `UIView`，所以 `UIView` 与 `CALayer` 并不一一对应。

调用 `parentView.addSubview(childView)` 时，UIKit 同时把 `childView` 的支撑图层加入 `parentView.layer` 的图层层级。直接向 `parentView.layer` 添加独立图层时，UIKit 不会创建子视图，`parentView.subviews` 也不会增加元素。

### 多个子图层的用途与形状

多个子图层可以把一个控件的不同视觉部分分开管理。例如，一个进度控件可以用 `CAGradientLayer` 显示渐变背景，用 `CAShapeLayer` 显示进度线，再用另一个图层显示装饰。各图层可以分别改变颜色、位置、透明度或动画，不必把全部内容合并成同一张图片。

`CAShapeLayer` 根据 `path` 指定的几何路径绘制形状。路径是一组描述直线、曲线和轮廓的数据，可以表示矩形、圆、折线或自定义轮廓。`CAShapeLayer` 画出的形状完全由 `path` 决定；例如要得到圆形轮廓，需要用宽高相等的矩形生成椭圆路径，再把路径赋给 `path`。

只负责视觉展示和动画的部分可以使用独立图层；需要 UIKit 的触摸事件、手势或无障碍交互的部分，通常使用 `UIView`。原因是独立 `CALayer` 不在 UIKit 的响应者链（UIKit 依次传递事件的对象链）中：图层提供几何命中检测方法，但不具备 `UIView` 的事件处理能力。

额外添加的图层不支持 UIView 的 Auto Layout 约束。父视图尺寸变化时，代码需要按新的布局手动调整子图层的尺寸和路径。拆分图层便于分别控制各个视觉部分；图层数量增加也会增加合成工作，渲染不一定更快。

### 图层重用时的状态管理

重用已有 `CALayer` 时，图层保留上一次设置的 `contents`、`opacity`、`isHidden`、`transform` 等属性。为新数据配置图层时，代码必须重新设置所有随数据变化的属性，包括图片为空和隐藏状态。例如，上一次把 `opacity` 设为 `0`，这一次只替换 `contents`，新图片仍不可见。固定的装饰图层应只创建一次，之后只更新属性；每次配置都添加新的子图层，会让图层叠加显示并增加开销。一个图层同一时间只能属于一个父图层。

上一次添加的显式动画仍在运行时，显式动画会继续作用于新内容。动画移除后，图层按模型层属性显示；显式动画没有同步更新模型层时，外观会跳变。`removeAllAnimations()` 只移除当前图层上的动画：图层属性保持不变，子图层上的动画也保留。配置新数据时，如需属性立即生效，可以用 `CATransaction.setDisableActions(true)` 关闭本次修改的隐式动画。支撑图层由 UIKit 管理，普通属性赋值通常不会触发支撑图层的隐式动画。

父视图尺寸改变后，代码需要按新尺寸更新自建子图层的 `frame`，以及依赖尺寸的 `CAShapeLayer.path`。异步加载完成、准备把结果写入图层前，持有图层的 Cell 或视图要确认当前数据标识与发起请求时的标识一致。已经取消旧任务时，这项归属检查也要保留，因为旧结果可能在取消生效前已经返回。Cell 重用与异步结果的一般规则见 [长列表性能与异步图片复用](uikit-layout-and-mvvm.md#长列表性能与异步图片复用)。

### UIViewPropertyAnimator 的正向、反向与暂停

`UIViewPropertyAnimator` 是 UIKit 中的视图动画对象，可以控制播放方向、暂停状态和进度。例如图片先从小变大，再从当前大小缩小，可以用同一个 `UIViewPropertyAnimator` 控制 `transform`。transform 的几何含义见 [transform 与 bounds、center、frame](uikit-layout-and-mvvm.md#transform-与-boundscenterframe)。

下面的示例适用于 iOS 11 及以上，用线性曲线在两秒内把图片从 50% 大小放大到原始大小。使用方式：页面创建一个 `ImageScaleAnimation` 并用属性持有；三个按钮分别调用 `playForward`、`playBackward` 和 `pause`。

```swift
import UIKit

@MainActor
final class ImageScaleAnimation {
    private let animator: UIViewPropertyAnimator

    init(imageView: UIImageView) {
        imageView.transform = CGAffineTransform(scaleX: 0.5, y: 0.5)
        animator = UIViewPropertyAnimator(duration: 2, curve: .linear) {
            imageView.transform = .identity
        }
        animator.pausesOnCompletion = true
    }

    func playForward() {
        play(reversed: false)
    }

    func playBackward() {
        play(reversed: true)
    }

    func pause() {
        if animator.isRunning {
            animator.pauseAnimation()
        }
    }

    private func play(reversed: Bool) {
        pause()
        animator.isReversed = reversed
        animator.startAnimation()
    }
}
```

`pauseAnimation()` 让动画停在当前位置。`isReversed = false` 时，动画向终点（原始大小）播放；`isReversed = true` 时，动画向起点（50% 大小）播放。第一次调用 `startAnimation()` 时启动动画；动画处于激活且暂停的状态时，调用 `startAnimation()` 从当前位置继续播放。

`pausesOnCompletion = true` 让动画到达起点或终点后保持激活并暂停，之后仍可以改变方向继续播放。没有这项设置时，动画正常结束后变为非激活状态，原来的 `UIViewPropertyAnimator` 无法从端点继续播放原动画。示例只使用暂停和继续，不调用 `stopAnimation` 或 `finishAnimation`；页面退出、不再需要动画时，应先结束动画，再释放 `ImageScaleAnimation`。

例如，正向播放到中间大小时点击暂停，图片停在当前大小；再点击反向，图片从当前大小开始缩小，不会先跳到最大尺寸。播放方向完全由按钮决定，动画不会自动往返。`transform` 只改变显示缩放，布局约束和周围视图的排版保持不变。

## anchorPoint、position 与图层几何

`anchorPoint` 是 `CALayer` 在自身 `bounds` 中选定的参考点，使用相对宽高的单位坐标，默认值为 `(0.5, 0.5)`。单位坐标表示比例，不是像素或点数；常见取值在 `0...1`，但 API 也允许超出这个范围。

在 iOS 默认坐标方向下，`(0, 0)` 对应左上角，`(0.5, 0.5)` 对应中心，`(1, 1)` 对应右下角，`(0, 0.5)` 对应左边中点。图层围绕锚点旋转和缩放。下面把视图支撑图层的锚点改为左边中点：

```swift
view.layer.anchorPoint = CGPoint(x: 0, y: 0.5)
```

`anchorPoint` 和 `position` 共同决定图层的位置。对于没有变换的图层，`anchorPoint` 选定图层自身的一个点，`position` 指定这个点在父图层坐标系中的位置。

| 属性 | 描述的内容 |
| --- | --- |
| `bounds` | 对象在自身坐标系中的边界矩形 |
| `UIView.frame` | 无变换时，视图在父视图坐标系中的位置和大小；非单位变换下不应用它直接推算或设置原始几何信息 |
| `UIView.center` | 视图中心在父视图坐标系中的位置；默认锚点时，与支撑图层的 `position` 对应 |
| `CALayer.anchorPoint` | 在自身 `bounds` 内选定参考点的相对坐标 |
| `CALayer.position` | 图层在父图层坐标系中的定位值；无变换时，指定锚点应处的位置 |

对于未设置变换的图层，可以按两个坐标分量分别计算：`frame.origin = position - anchorPoint × bounds.size`。例如，`bounds = (0, 0, 100, 100)`，`position = (200, 200)`：

| anchorPoint | frame 左上角 |
| --- | --- |
| `(0.5, 0.5)` | `(150, 150)` |
| `(0, 0)` | `(200, 200)` |

`position` 保持不变、只把锚点从中心改为左上角时，图层向右、向下各移动 50 点。原因是父坐标中的 `(200, 200)` 原来对齐图层中心，改后对齐图层左上角。改变锚点时要保持图层原来的可见位置，需要同步调整 `position`；图层有旋转、缩放等变换时，调整量还要经过变换计算，与这个无变换示例中的 50 点不同。

`CALayer.frame` 是根据 `bounds`、`position`、`anchorPoint` 和变换计算出的、在父坐标系中的边界框。图层旋转后，`frame` 是包住旋转后图层的边界框，尺寸一般与未旋转时不同。视图布局与约束的关系见 [frame、Auto Layout 与 SnapKit](uikit-layout-and-mvvm.md#frameauto-layout-与-snapkit)。

## 圆角、边界裁剪与离屏渲染

### cornerRadius 与 masksToBounds 的职责

`cornerRadius` 是 `CALayer` 绘制背景和边框时使用的圆角半径，单位为点。`masksToBounds` 决定图层是否按自身边界裁剪内容及子图层；开启后，裁剪形状带有 `cornerRadius` 指定的圆角。裁剪只隐藏边界以外的显示区域，原图片数据保持不变。

`bounds` 表示图层在自身坐标系中的边界矩形。设置 `cornerRadius` 后，`bounds` 仍是原来的矩形，布局尺寸也不变。基础 `CALayer` 的 `cornerRadius` 默认是 `0`，`masksToBounds` 默认是 `false`。

```swift
imageView.layer.cornerRadius = 12
imageView.layer.masksToBounds = false
```

假设 `imageView` 显示一张不透明矩形图片，图片覆盖到视图的四个角，并且没有其他蒙版或父视图裁剪。此时背景和边框按 12 点半径绘制圆角，图片内容不受这个圆角裁剪，图片通常仍显示为直角矩形。

改成下面的代码后，图片会按圆角边界裁剪：

```swift
imageView.layer.cornerRadius = 12
imageView.layer.masksToBounds = true
```

两种写法的圆角半径都已生效，区别只在于图片内容是否被裁剪。原图四角本来透明，或者图片没有铺满四个角时，两种配置的最终外观可能相同。

| 配置 | 背景与边框 | 图像内容与子图层 |
| --- | --- | --- |
| `cornerRadius = 12`，`masksToBounds = false` | 可以呈现圆角 | 不因该圆角被裁剪；仍可能受其他裁剪影响 |
| `cornerRadius = 12`，`masksToBounds = true` | 可以呈现圆角 | 按包含圆角的边界裁剪 |

图层只有普通背景和边框、没有图片、子图层或其他复杂效果时，即使开启裁剪，现代系统通常也能直接绘制，一般不需要离屏渲染。这种情况下关闭裁剪，理由只是需求用不到裁剪；开关前后的渲染工作可能相同，关闭裁剪不一定带来可测量的性能收益，也不一定减少掉帧。

需求只要求背景和边框圆角时，图层上仍可能有图片或子图层，开启裁剪会同时裁剪这些内容。是否开启裁剪、性能会受什么影响，要根据图层的实际内容判断。

### clipsToBounds 与 masksToBounds 的关系

`clipsToBounds` 是 `UIView` 提供的裁剪开关，`masksToBounds` 是 `CALayer` 提供的裁剪开关。对普通 UIKit 视图，设置视图的 `clipsToBounds` 会同步设置支撑图层的边界裁剪。设置图片圆角时，两个开关选一个设置即可，例如：

```swift
imageView.layer.cornerRadius = 12
imageView.clipsToBounds = true
```

`UIView` 的 `clipsToBounds` 默认值是 `false`；部分子类会修改默认值，使用前应确认具体视图的初始值。开启裁剪后，超出边界的子视图内容也会被隐藏。

### 离屏渲染的含义与常见场景

离屏渲染指 GPU 为完成某个图层效果，先把参与效果的内容绘制到一张中间图像，再用中间图像合成最终画面。GPU 是执行图形计算的处理器；中间图像是保存临时像素结果的渲染目标。

“离屏”指绘制结果先写入中间图像，而不是直接写入最终显示的画面；“离屏”与代码运行在哪个线程、视图是否在屏幕可见范围内无关。Swift 代码只负责设置图层属性；是否需要中间绘制，由渲染系统根据图层属性决定。

| 场景 | 可能需要中间结果的原因 |
| --- | --- |
| 可见阴影，且未设置 `shadowPath` | 渲染器可能需要先得到内容合成后的透明度轮廓，再生成阴影 |
| 图层蒙版 `mask` | 蒙版用自身各像素的透明度控制目标内容可见程度，可能需要先绘制被遮罩的内容与蒙版 |
| 多个子图层的圆角整体裁剪 | 渲染器可能需要先合成子图层，再保留圆角范围内的结果 |
| `UIVisualEffectView` 的模糊等视觉效果 | 效果可能依赖背景图像，需要采样、模糊和额外的中间绘制 |
| 组透明度，且子图层存在需要整体合成的内容 | 当父层透明度介于 0 和 1、并启用 `allowsGroupOpacity` 等条件成立时，可能先合成一组内容，再统一应用透明度 |
| `shouldRasterize = true` | 要求把图层及子图层绘制成位图；首次生成或缓存失效时产生中间绘制，后续可能复用缓存 |

`cornerRadius` 和 `masksToBounds` 是否触发离屏渲染，取决于图层结构、效果组合、系统和设备。系统对简单的圆角裁剪有优化，通常不需要中间绘制；多个子图层与其他效果组合时，才可能需要额外绘制。

额外的中间绘制会增加像素计算、临时图像和读写开销。这些开销是否造成卡顿，取决于绘制面积、发生频率和当时的渲染负载；要确认离屏绘制是卡顿原因，还需要测量渲染耗时。

衡量离屏渲染的影响时，看每一帧需要完成多少额外工作，而不是程序累计触发了多少次。新画面没有赶在显示时刻前完成时，屏幕继续显示旧画面，用户看到滚动或动画掉帧。主线程按时提交变化后，渲染服务或 GPU 仍可能没有按时完成画面。

较大的临时图像和较多的像素读写会增加内存与数据传输压力，持续的绘制负载还可能增加耗电和发热。系统会复用或释放临时缓冲区，这部分内存占用属于正常现象，不是内存泄漏。设备过热导致处理器运行频率下降时，流畅度可能进一步下降。固定 60／120 Hz 的显示间隔约为 16.67／8.33 ms，完整流水线的时序见 [图层动画与渲染管线](#图层动画与渲染管线)。

合成指按层级、位置和透明度把多个图像组合为一个结果。例如组透明度先把重叠的子图层合成一张图，再对整张图应用一次透明度；逐个设置子图层透明度时，重叠区域会叠加两次半透明，颜色与组透明度的结果不同。因此关闭组透明度会改变外观，不能直接替代组透明度。只设置 `alpha < 1` 或 `opacity < 1` 时，是否发生离屏绘制，还取决于上表中的其他条件。

`draw(_:)` 和图像绘制接口在 CPU 上生成图像，与本节讨论的 Core Animation GPU 离屏绘制属于不同阶段。视图实现 `draw(_:)`，只说明视图在 CPU 上绘制内容；是否触发 GPU 离屏绘制，仍取决于上表中的图层效果。

### shouldRasterize 与位图缓存

栅格化是把图层内容转换为像素位图。`shouldRasterize = true` 让 Core Animation 把图层及子图层的绘制结果生成一张位图，并在后续帧尝试复用这张位图。`shouldRasterize` 本身不产生圆角；图层已有复杂但稳定的圆角组合时，位图会包含裁剪后的效果，缓存能复用时就有机会减少重复绘制和合成。

生成位图本身就是一次离屏绘制，`shouldRasterize` 的收益来自后续帧复用位图。系统可能让缓存失效或回收缓存，位图可能被生成多次。图片、文字、尺寸或内部效果频繁变化时，位图需要反复重新生成，开销反而增加，因此不应默认开启。

例如，图片和文字固定的卡片仅随列表移动时，缓存可能有收益；卡片不断替换图片或更新布局时，缓存需要反复重建。缓存能否复用，要在目标设备上用实际的动画和尺寸验证。

开启时还应设置 `rasterizationScale`，让缓存位图中每个 point 对应的像素数与目标屏幕一致。比例低于屏幕比例时内容模糊；比例越高，位图越大，占用内存越多。位图缓存由 Core Animation 管理，App 无法取得缓存位图并作为 UIImage 保存。

### 静态图片的圆角预处理

测量确认静态图片的重复裁剪是渲染超时的重要来源时，可以提前生成圆角图片并缓存，显示时关闭 `imageView` 图层的圆角裁剪。位图是由像素组成的图像；生成的圆角图片仍是矩形位图，四个角的像素是透明的，因此看起来是圆角。

后台线程按最终显示尺寸生成圆角图片，再回到主线程设置 `imageView.image`。缓存标识应包含原图标识、目标尺寸、圆角半径和像素比例。缓存大小和复用时的过期结果校验见 [iOS 图片加载、解码与内存](image-loading-decoding-and-memory.md)。

下面的函数只负责生成图片，切换线程和缓存由调用方完成。调用要求：输入图片和 `size` 的宽高、`scale` 都大于零；`radius` 以 point 为单位，不超过目标矩形宽高较小值的一半。`scale` 是每个 point 对应的像素数，由调用方传入目标屏幕的比例。

```swift
func makeRoundedImage(
    _ image: UIImage,
    size: CGSize,
    radius: CGFloat,
    scale: CGFloat
) -> UIImage {
    let format = UIGraphicsImageRendererFormat()
    format.scale = scale
    format.opaque = false
    let renderer = UIGraphicsImageRenderer(size: size, format: format)

    return renderer.image { _ in
        let bounds = CGRect(origin: .zero, size: size)
        UIBezierPath(roundedRect: bounds, cornerRadius: radius).addClip()

        let ratio = max(
            size.width / image.size.width,
            size.height / image.size.height
        )
        let drawSize = CGSize(
            width: image.size.width * ratio,
            height: image.size.height * ratio
        )
        let drawRect = CGRect(
            x: (size.width - drawSize.width) / 2,
            y: (size.height - drawSize.height) / 2,
            width: drawSize.width,
            height: drawSize.height
        )
        image.draw(in: drawRect)
    }
}
```

`renderer.image` 在调用线程上同步执行绘图闭包。执行闭包前，`renderer` 创建绘图上下文并设为当前上下文；绘图上下文是保存绘制目标、坐标变换和裁剪区域等状态的对象。`UIBezierPath(roundedRect:cornerRadius:)` 描述圆角矩形的几何路径；`addClip()` 把当前上下文的裁剪区域与路径内部取交集，之后的绘制只出现在交集区域内。

`image.draw(in:)` 在同一个当前上下文中绘制图片，因此受已设置的裁剪区域限制。`addClip()` 把路径写入了上下文的裁剪区域，`draw(in:)` 读取的是上下文状态，所以 `draw(in:)` 不需要接收 `UIBezierPath` 参数。需要在绘制过程中临时改变裁剪、之后恢复原状态时，配对调用 `CGContext` 的 `saveGState()` 与 `restoreGState()`。

`ratio` 取宽、高两个缩放比例中的较大值，让原图等比例缩放后填满目标区域；`drawRect` 让图片居中，超出目标区域的部分被裁掉。`opaque = false` 让位图保留透明通道，四角保持透明。函数返回一张已包含圆角像素的新 `UIImage`，原图片保持不变。

`imageView` 没有其他裁剪需求时，在主线程显示结果：

```swift
imageView.layer.cornerRadius = 0
imageView.clipsToBounds = false
imageView.image = roundedImage
```

提前生成圆角图片同样消耗绘制时间和内存。这种优化的收益来自在后台生成一次、之后反复复用，避免每次 Cell 显示时在主线程重新绘图。显示阶段的裁剪成本发生在渲染阶段，只把 `cornerRadius` 的赋值移到后台线程无法消除这部分成本；UIView 的修改也应在主线程进行。

预处理适合静态头像和缩略图。视频帧或频繁变化的内容需要不断生成新位图，不适合预处理；这类内容应保留必要的裁剪，并减少被裁剪面积、无用子图层和额外效果。任何方案都要在目标设备上测量掉帧情况，配置本身无法保证完全不掉帧。

### shadowPath 与圆角阴影的组合

`shadowPath` 是显式指定阴影轮廓的路径。轮廓已知时，渲染器不必先从整个内容的合成结果推导轮廓，因此可以减少或避免这一步导致的离屏绘制。其他蒙版、裁剪或模糊效果仍可能有额外开销。

圆角卡片既要在外侧显示阴影、又要裁剪内部图片时，可以使用两层视图：外层 `shadowView` 显示阴影并关闭裁剪，内层 `imageView` 裁剪图片。原因是阴影画在图层边界之外；负责阴影的图层一旦开启边界裁剪，阴影也会被裁掉。

下面假设两个视图已经创建，并且 `imageView` 已添加为 `shadowView` 的子视图。先设置外观：

```swift
let radius: CGFloat = 12

shadowView.layer.cornerRadius = radius
shadowView.layer.masksToBounds = false
shadowView.layer.shadowColor = UIColor.black.cgColor
shadowView.layer.shadowOpacity = 0.2
shadowView.layer.shadowRadius = 6
shadowView.layer.shadowOffset = CGSize(width: 0, height: 3)

imageView.layer.cornerRadius = radius
imageView.clipsToBounds = true
```

布局得到有效尺寸后，再更新内层视图和阴影路径：

```swift
imageView.frame = shadowView.bounds
shadowView.layer.shadowPath = UIBezierPath(
    roundedRect: shadowView.bounds,
    cornerRadius: radius
).cgPath
```

`shadowPath` 使用图层自身的坐标系，因此示例用 `bounds` 生成路径。尺寸或圆角半径变化时，需要重新生成路径，例如在视图的布局回调中更新。`shadowRadius` 控制阴影的模糊程度，`cornerRadius` 控制圆角形状，两者是不同的半径。

### 性能判断与优化边界

优先使用系统提供的圆角属性；只需要普通圆角形状时，不必另建自定义 `mask` 图层。只有确认内容和子视图不会超出边界、也不需要裁掉图片四角时，才关闭裁剪；界面实际需要的裁剪必须保留。

在目标设备上录制实际的滑动或动画，再结合 [Animation Hitches](#animation-hitches-的使用流程) 和图层检查信息，判断是否存在额外的离屏绘制、发生在哪个图层，以及是否造成渲染超时。优化后重复相同操作，比较渲染时间与外观，用测量结果确认优化是否生效。

## 回调 FPS 与卡顿监测

`CADisplayLink` 回调可以用来做轻量的卡顿监测：主线程繁忙时，显示回调会延迟，回调频率下降、间隔变长。监测用的 DisplayLink 应注册到主线程 RunLoop 的 `.common` 模式，让列表拖动期间也能执行回调。只注册 `.default` 模式时，拖动期间 RunLoop 切换模式，回调暂停，监测结果会把这段暂停误判为卡顿。

计算方法：单调时钟是只向前走、不受系统时间修改影响的时钟。每次回调读取单调时钟；第一次回调保存基准时间，之后每次回调把间隔数加 1；约一秒后，用间隔数除以实际经过的时间，得到近似的回调 FPS。同时记录每一次的间隔，因为一秒的平均值会掩盖单次明显停顿。判断间隔是否异常时，以当前的调度帧率为准：按 60 FPS 调度时，正常间隔约 16.7 ms，突然出现 100 ms 的间隔值得记录排查；按 120 FPS 调度时，正常间隔约 8.3 ms。动态帧率下要结合实际调度节奏判断；系统主动降低帧率时回调间隔变长，这不是性能卡顿。

**回调 FPS 只反映 DisplayLink 回调的节奏，回调执行时新画面还未必成功上屏。** 主线程按时回调后，渲染服务或 GPU 仍可能错过显示时机。因此回调 FPS 是近似指标：回调 FPS 正常时仍可能有卡顿，回调 FPS 也无法说明卡顿的根因。

需要定位卡顿时，在真机上用 [Animation Hitches](#animation-hitches-的使用流程) 录制滑动，找到具体的停顿及其所在的提交／渲染阶段；再结合 [Time Profiler](#time-profiler-的采样与统计边界) 与线程状态，区分主线程是在执行耗时计算，还是在等待锁、资源等。线上卡顿可以结合 [MetricKit 与 Xcode Organizer](#metrickit-与-xcode-organizer) 的报告分析。

### CADisplayLink 帧时间与卡顿时长统计

基于帧时间戳的卡顿统计：用相邻两次回调提供的帧时间和预期显示时间计算延迟，再累计延迟时长。这是观察界面流畅度的近似方法。计算出的延迟衡量的是帧没有按预期时刻显示的时长，既不是某个方法的执行耗时，也不是完整的画面渲染耗时。

**回调来源与执行顺序。** `CADisplayLink` 是 iOS 提供的对象，按显示刷新节奏调用指定的方法。监控器创建 `CADisplayLink`，注册采样方法，并将 `CADisplayLink` 加入主线程的 `RunLoop`。`RunLoop` 是线程接收和处理事件的循环。系统调度显示回调时，调用采样方法并传入 `CADisplayLink` 对象；采样方法读取帧时间、计算延迟，并保存本次数据供下一次使用。

下文的“本次回调”和“上次回调”，指监控用的同一个 `CADisplayLink` 相邻两次调用采样方法。卡顿统计使用 `CADisplayLink` 提供的帧时间，而不是采样方法实际开始执行的时刻。

| 属性或表达式 | 含义 |
| --- | --- |
| `timestamp` | 系统提供的最近一帧已经显示的时间。 |
| `targetTimestamp` | 系统提供的下一帧预期显示时间。 |
| `targetTimestamp - timestamp` | 本次回调提供的这两个帧时间之间的间隔。 |
| `duration` | 屏幕按最大刷新率刷新时的帧间隔；不要把该属性直接当作所有调度频率下的实际回调间隔。 |

上述 API 的时间单位是秒。以毫秒统计时，把所有时间统一乘以 `1000`，避免秒与毫秒混算。

**下一帧预期显示时间直接读取 `targetTimestamp`。** 按每秒 60 帧稳定调度时，`targetTimestamp - timestamp` 约为 16.67 ms；按每秒 120 帧稳定调度时，约为 8.33 ms。动态帧率下这个差值会变化，所以监控器读取系统每次提供的 `targetTimestamp`，而不是固定用 `timestamp + 16.67 ms` 推算。

**逐帧计算。** 在同一段连续采样中，监控器保存上次的 `timestamp` 和 `targetTimestamp`。第一次回调只建立基准；从第二次回调开始，按下面的公式计算。以下公式统一使用毫秒。

| 计算量 | 公式 |
| --- | --- |
| 本次统计时长 | 本次 timestamp − 上次 timestamp |
| 原始延迟 | 本次 timestamp − 上次 targetTimestamp |
| 本次计入的卡顿时长 | 原始延迟 > 过滤阈值时，取原始延迟；否则取 0。 |

过滤阈值决定多大的延迟才计入累计值。例如以约 8.33 ms（即 `1000 / 120` ms）为过滤阈值：延迟小于或等于 8.33 ms 时不计入；延迟超过 8.33 ms 时，计入整段原始延迟，而不是减去阈值后的差值。阈值由指标定义者选择，iOS 没有为卡顿统计规定统一的阈值。

假设按每秒 60 帧稳定调度，相邻两次有效采样取得下列数值：

| 时间 | 数值 |
| --- | --- |
| 上次 timestamp | 1000.00 ms |
| 上次 targetTimestamp | 1016.67 ms |
| 本次 timestamp | 1050.00 ms |

本次统计时长为 `1050.00 − 1000.00 = 50.00 ms`。原始延迟为 `1050.00 − 1016.67 = 33.33 ms`。采用约 8.33 ms 的过滤阈值时，本次计入的卡顿时长为 33.33 ms；上次 `timestamp` 到上次 `targetTimestamp` 之间正常的约 16.67 ms 不计入卡顿时长。

本次计算完成后，监控器保存本次的 `timestamp` 和 `targetTimestamp`。下一次回调到来时，再用新的 `timestamp` 与这次保存的 `targetTimestamp` 比较。

**累计时长与丢帧时长率。** 本节将“丢帧时长率”定义为：同一采样范围内，累计计入的卡顿时长除以累计统计时长。

`累计统计时长 = Σ 本次统计时长`

`累计卡顿时长 = Σ 本次计入的卡顿时长`

`丢帧时长率 = 累计卡顿时长 / 累计统计时长`

例如累计统计时长为 1000 ms，累计卡顿时长为 100 ms，则比值为 0.1。以百分比展示是 10%；以每秒卡顿多少毫秒展示是 100 ms/s。比较两组数据时，两组的分母、采样范围、过滤阈值和展示单位必须一致。累计统计时长为 0 时，不计算比率。

合并多段记录时，先分别累加各段的卡顿时长和统计时长，再计算 `Σ 卡顿时长 / Σ 统计时长`。各段时长不相等时，直接对各段百分比取算术平均会放大短时段的权重，结果与总比率不同。

**采样范围与观测边界。** 采样应覆盖需要观察的动画或交互过程。停止采样、进入后台或暂停显示回调后，恢复采样时要重新建立时间基准，否则中间未采样的时间会被计为卡顿。拖动期间能否继续采样，取决于显示回调注册的 RunLoop 模式。

采样方法应只读取时间并更新少量统计值。逐帧执行复杂计算或同步上传数据，会增加主线程工作，影响被监测的界面。

这种方法只测量显示回调所提供帧时间的延迟。要证明某段业务代码造成卡顿，或覆盖渲染服务和 GPU 造成的全部显示延迟，还需要结合渲染时间线、线程状态及方法调用记录。渲染服务负责在应用提交图层之后组织后续的渲染工作。

统计“连续丢 N 帧”对应的时长率时，还要先定义三项：把延迟换算为帧数时使用的时间基准、筛选条件，以及分子累计的是延迟时长还是整个帧间隔。用固定 16.67 ms 换算与用当前帧间隔换算，分档结果可能不同；指标名称相同但定义不同时，数值不能直接比较。

## Time Profiler 的采样与统计边界

**Time Profiler 周期性采集正在 CPU 上执行的线程调用栈，再按采样权重估算各调用路径的 CPU 时间。** Time Profiler 不在方法的入口和出口记录时间。Call Tree 展示选定时间范围内所有样本的汇总结果；每个样本仍保留了采样时刻和调用栈，所以时间线上仍能看出各个时刻执行到了哪里。

以每 1 ms 采样一次为简化示例（实际采样间隔取决于录制设置）：

```text
估算 CPU 时间 ≈ 命中相应方法／调用路径的样本数 × 采样间隔
```

估算值按样本数计算，而不是用“最后一次采到该方法的时间 − 第一次采到的时间”。一个方法出现在调用栈中，可能是方法自身正在执行，也可能是方法调用的子方法正在执行。因此查看统计时要区分两个值：Weight 包含子调用的样本，Self Weight 只计方法自身的样本。父方法的 Weight 已经包含子方法的 Weight，汇总时不要再把两者相加。

| 一次调用的假设情况 | 数值 |
|-|-|
| 实际开始／结束 | 1.2 ms／4.8 ms |
| 实际经过时间 | 3.6 ms |
| 连续在 CPU 上执行，并在整数毫秒采样时命中 | 2、3、4 ms，共 3 个样本 |
| 按固定间隔估算 | 约 3 ms，而不是 `4 − 2 = 2 ms` |

**统计边界：**

- 采样不一定落在方法真实的入口与出口上。例如 1.2～1.5 ms 内执行完的短调用，可能完全落在两个采样点之间，一个样本都没有。
- 同一方法的多次调用会按所选调用树口径汇总在一起。显示约 20 ms，可能是多次调用之和；单次调用耗时和准确的调用次数都无法从这个值得出。
- 每个样本只说明“这个时刻执行到了哪里”，采样间隔内的全部调用无法完整还原。例如第 1、2、3 ms 分别采到 A、B、A，两次看到的 A 可能是同一次调用，也可能是两次不同的调用；各方法的精确起止时间也无法还原。
- Time Profiler 统计的是 CPU 执行时间，与调用的实际经过时间不同。线程等待锁、I/O 时不在 CPU 上执行，等待会延长调用的返回时间，但这段时间不计入该方法的 CPU 时间。Time Profiler 也不直接测量 GPU 耗时。

| 想回答的问题 | 适合的方式 |
|-|-|
| 某个时间段 CPU 主要花在哪些线程、方法及调用路径 | Time Profiler，选中目标时间范围后看调用树 |
| 这一次快照调用从开始到返回经过多久 | 成对 Signpost 记录调用区间 |
| 每次已标记调用的开始、结束、持续时间及顺序 | Signpost + Instruments 时间线；只覆盖主动标记的操作 |
| 动画是否错过显示时机，问题位于提交还是渲染阶段 | Animation Hitches，再结合调用栈和阶段标记定位 |

## Animation Hitches 的使用流程

**Animation Hitches 是 Apple 的 Instruments 中用于分析动画、滑动卡顿的模板。** Hitch 指运动中的画面没有按预期时刻更新。例如滑动评论列表时，新位置的画面没有及时准备好，屏幕继续显示旧画面，用户就看到画面顿了一下。相关阶段见[Core Animation 渲染管线](#图层动画与渲染管线)。

使用流程：打开 Instruments，选择 Animation Hitches 模板和目标 App，开始录制，再在设备上执行要排查的滑动或动画。工具在时间线上标出卡顿，并提供提交、渲染、GPU、帧生命周期等轨道，帮助判断卡顿发生的时间、持续时长及主要耗时阶段。

以点击推送后直达评论为例：录制一次自动滚动到目标评论的过程，选中停顿所在的时间段，查看延迟出现在 App 的事件处理／布局／提交，还是后续的渲染准备／GPU 执行。Animation Hitches 标出的是延迟所在的阶段；进一步定位具体业务代码，还要结合 [Time Profiler](#time-profiler-的采样与统计边界) 等工具中的调用栈和线程状态。

[CADisplayLink](runloop-timer-and-displaylink.md#cadisplaylink-的回调时机) 通过回调间隔观察流畅度的迹象；Animation Hitches 用于录制并分析一次具体交互的渲染时间线。

### 同次录制中的时间线关联

Animation Hitches 标出卡顿发生的时间，并展示持续时长和相关的渲染阶段。定位 CPU 上的方法时，需要在同一次录制中同时采集 Time Profiler，才能把帧的时间线与同一时段的线程调用栈对齐；两次独立录制中，相同的秒数对应的是不同的工作。

Animation Hitches 模板可以组合多个分析工具。开始录制前，检查当前模板是否已经包含 Time Profiler：已经包含时，直接录制即可；缺少时，把 Time Profiler 加入本次录制。

画面停顿被看到的时刻，通常晚于耗时代码开始执行的时刻，因为一帧要经过提交和渲染才会显示。所以应沿帧生命周期往前查找导致这一帧延迟的事件处理、布局和提交区间，再查看这些区间对应的 CPU 采样。Time Profiler 只反映采样到的 CPU 调用路径，GPU 执行瓶颈要看 GPU 相关轨道；某个方法出现在卡顿附近时，还要确认方法落在导致延迟的区间内，才能认定因果关系。

## MetricKit 与 Xcode Organizer

**MetricKit 是 App 中接收系统性能与诊断报告的框架；Xcode Organizer 是开发者在 Xcode 中查看 App 数据的窗口。** 下文只涉及 Organizer 的性能指标与诊断功能。

| 对比项 | MetricKit | Xcode Organizer |
|-|-|-|
| 使用方式 | 在 App 中接入，接收系统提供的性能指标和诊断报告 | 在 Xcode 的 Window → Organizer 中查看 |
| 数据处理 | App 按需保存、上传到自己的监控平台，再做统计或告警 | Apple 收集并汇总数据，开发者查看指标及可用的诊断报告 |
| 典型用途 | 接入公司的线上性能监控体系 | 比较版本、机型上的启动耗时、内存、卡顿及无响应表现 |

**MetricKit 只把报告交给 App，上传到公司平台由 App 自己完成。** 业务代码或监控 SDK 负责上传，完整链路如下：

```text
系统收集数据 → MetricKit 将报告交给 App
            → 公司监控 SDK／业务代码上传 → 平台统计、展示和告警
```

MetricKit 交付的是系统收集、汇总后的报告，不提供实时的逐帧回调。Organizer 的系统性能指标由 Apple 收集，App 通常不需要专门接入 MetricKit 就能在 Organizer 中看到这些指标；Organizer 的数据也不依赖 MetricKit 上传。

**线上性能数据的取得条件：** 以通过 App Store 发布的 App 为例，Organizer 显示的是同意共享数据的用户的匿名汇总性能数据。

1. App 上架 App Store 后，用户下载安装并使用。
2. 用户同意与开发者共享分析与诊断数据后，用户的设备把性能和诊断数据发送给 Apple。
3. Apple 汇总报告；开发者在 Xcode 登录有权限的团队账号后，通过 Organizer 查看。

例如，开发者可以比较新旧版本的启动耗时、内存和卡顿指标，并在支持的指标上按机型筛选。Organizer 只展示汇总统计结果，看不到单个用户的身份和实时运行情况。统计样本只包含同意共享数据的用户，一万名使用者中只有其中一部分进入统计。

Organizer 需要足够的使用样本才显示指标，而且从发布到报告可见有一段延迟；只给几个朋友安装测试时，可能看不到性能指标。Apple 的 Metrics 报告面向 App Store 分发的版本；TestFlight 版本可以提供崩溃等报告，但全部性能指标是否覆盖 TestFlight，不能由此推断。线上指标用于发现某个版本的性能退化，具体原因要结合诊断报告与复现分析。
