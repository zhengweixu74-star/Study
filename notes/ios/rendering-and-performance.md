# Core Animation 渲染与性能分析工具

[返回笔记目录](../README.md)

## 本文目录

- [图层动画与渲染管线](#图层动画与渲染管线)
- [回调 FPS 与卡顿监测](#回调-fps-与卡顿监测)
- [Time Profiler 的采样与统计边界](#time-profiler-的采样与统计边界)
- [Animation Hitches 的使用流程](#animation-hitches-的使用流程)
- [MetricKit 与 Xcode Organizer](#metrickit-与-xcode-organizer)

区分应用提交、渲染执行与显示阶段，并为 CPU 热点、单次耗时和动画卡顿选择对应观测工具。

## 图层动画与渲染管线

**与 UIView 动画的关系：** UIView 动画是 UIKit 提供的高层接口，底层利用 Core Animation 的图层动画能力。每个 UIView 背后都有 CALayer；UIView 管理[视图层级](uikit-layout-and-mvvm.md#视图层级与父子关系)、布局与交互，CALayer 保存位置、大小、透明度、圆角等视觉状态，Core Animation 根据动画配置计算中间状态并参与渲染、合成。UIView 本身也有绘制和动画 API，不能把这些职责理解为互不重叠。

例如让视图在 0.3 秒内淡出：使用 UIKit 时，在 `UIView.animate` 中把 `view.alpha` 设为 `0`；直接使用 Core Animation 时，为图层的 `opacity` 创建并添加 `CABasicAnimation`，同时处理模型属性的最终值。前者由 UIKit 组织相应的底层图层动画，无需业务逐帧修改透明度。

UIKit 动画依赖 Core Animation，不表示业务代码直接创建了显式图层动画对象。

**动画原理：** CALayer 管理视觉内容与位置、变换、透明度等属性。应用描述起终点、时长、关键帧或弹簧规则，系统依据时间计算中间状态。位移、缩放、淡出通常能复用已有位图／纹理，不必每帧重画内容；这不表示所有内容变化都没有 CPU 开销。

- 模型树保存代码设置的状态；呈现树反映动画中的当前状态，可通过 presentation() 读取；渲染树是系统内部结构。
- 对简单数值属性，可理解为“起点 + 差值 × 时间曲线处理后的进度”，不是每个回调固定增加一段距离。
- CABasicAnimation、CAKeyframeAnimation、CASpringAnimation 和 CAAnimationGroup 分别表达基本插值、关键帧／路径、弹簧与动画组合。
- 显式添加 CAAnimation 不会自动改变模型属性；需要将模型更新为预期最终状态，否则移除动画后可能回到原状态。UIView.animate 是更上层接口，普通 UIView 属性在动画块外修改通常不会自动过渡。
- CATransaction 将图层操作批量提交；隐式事务结合 RunLoop 自动提交。事务与动画类型要区分，显式动画也需要事务提交。

**典型渲染管线：**

| 阶段 | 位置与工作 |
|-|-|
| Event | App 处理触摸、Timer、数据回调，修改视图／图层状态 |
| Commit | App 完成必要的布局、内容绘制，整理并提交图层变化 |
| Render Prepare | Render Server 的 CPU 处理图层和效果，准备 GPU 命令与资源 |
| Render Execute | GPU 完成绘制、采样、变换、混合等工作，生成画面 |
| Display | 显示系统在相应时机呈现已准备好的画面 |

App 的 draw(\_:) 可生成供后续使用的图像内容；渲染服务与 GPU 再按图层属性合成最终画面。不能把 Commit 中的内容绘制和 GPU 合成混为同一阶段，也不能说 Core Animation 完全不使用 CPU。

流水线可以同时处理不同帧，App 准备后续更新时，GPU 可以处理之前提交的内容。60／120 Hz 对应约 16.67／8.33 ms 的显示间隔，不等于一次更新从输入到显示只需要一个间隔，也不能把固定几帧延迟当所有设备与系统的保证。

Commit hitch 是 App 未及时完成事件处理、布局／绘制和提交；Render hitch 是渲染准备或执行超时。主线程不卡不代表 GPU 或渲染服务没有瓶颈。未按时准备好新画面时，旧画面可能继续显示。RunLoop 是线程事件循环，Render Loop 是跨进程、CPU／GPU 到显示的完整流程。

FMP 边界仍适用：willDisplay、布局和事务提交完成，都不能直接等同指定内容真实上屏。

## 回调 FPS 与卡顿监测

可以做轻量监测：主线程忙不过来时，显示回调可能延迟，通过回调频率与间隔观察卡顿迹象。把 DisplayLink 注册到主线程 RunLoop 的 `.common` 模式，让常见列表拖动期间也能执行，避免只注册 `.default` 后因模式切换暂停回调而误判。

回调时记录单调时钟，先保存基准时间，随后累计回调间隔数；在约一秒的统计窗口内，用间隔数除以实际经过时间，得到近似回调 FPS。同时记录每次间隔，避免平均值掩盖某一次明显停顿。例如当前按 60 FPS 调度时正常间隔约 16.7 ms，突然出现 100 ms 间隔值得记录排查；120 FPS 约 8.3 ms，动态帧率下不能统一套用 16.7 ms。应结合实际调度节奏判断，主动降帧不等于发生性能卡顿。

**回调一次不代表新画面已成功上屏。** 这个统计反映 DisplayLink 回调节奏，是近似指标；后续渲染服务或 GPU 也可能错过显示时机，不能仅靠回调次数覆盖所有卡顿，更不能直接据此确定根因。[CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink)、[ProMotion 刷新时序](https://developer.apple.com/documentation/quartzcore/optimizing-iphone-and-ipad-apps-to-support-promotion-displays)。

不限工具时，在真机上用 [Animation Hitches](#animation-hitches-的使用流程) 录制滑动，定位具体停顿和提交／渲染阶段；再结合 [Time Profiler](#time-profiler-的采样与统计边界) 与线程状态，区分主线程在执行耗时计算，还是等待锁、资源等。线上情况可结合 [MetricKit 与 Xcode Organizer](#metrickit-与-xcode-organizer) 的报告分析。这是验证思路，不代表已实现监控或完成本项目实测。

## Time Profiler 的采样与统计边界

**Time Profiler 周期性采集正在 CPU 上执行的线程调用栈，再按采样权重估算各调用路径的 CPU 时间。** 它不是在每个方法的入口和出口自动记录时间。Call Tree 展示选定时间范围内的聚合结果；样本本身仍保留采样时刻和调用栈，因此并非完全没有时序信息。

以固定每 1 ms 采样一次为简化示例（不表示所有录制都固定使用该间隔）：

```text
估算 CPU 时间 ≈ 命中相应方法／调用路径的样本数 × 采样间隔
```

不是“最后一次采到该方法的时间 − 第一次采到的时间”。同一方法出现在调用栈中，可以是它自身正在执行，也可以是它调用的子方法正在执行；查看统计时要区分包含子调用的 Weight 与自身的 Self Weight，不能把父子累计值重复相加。

| 一次调用的假设情况 | 数值 |
|-|-|
| 实际开始／结束 | 1.2 ms／4.8 ms |
| 实际经过时间 | 3.6 ms |
| 连续在 CPU 上执行，并在整数毫秒采样时命中 | 2、3、4 ms，共 3 个样本 |
| 按固定间隔估算 | 约 3 ms，而不是 `4 − 2 = 2 ms` |

**统计边界：**

- 采样不保证碰到方法的真实入口与出口；例如 1.2～1.5 ms 执行完的短调用，可能完全落在两个采样点之间而漏采。
- 同一方法的多次调用会按所选调用树口径汇总。显示约 20 ms，不代表单次调用就是 20 ms，也不能据此得出准确调用次数。
- 采样点能说明“这个时刻执行到了哪里”，不能完整还原采样间隔内的全部调用。若第 1、2、3 ms 分别采到 A、B、A，不能仅凭这些样本判断两次看到的 A 是同一次调用还是两次不同调用，也不能还原各方法精确起止时间。
- Time Profiler 的 CPU 采样统计与调用的实际经过时间不同：锁等待、I/O 等等待可能延长返回时间，却不代表 CPU 持续执行该方法。它也不直接测量 GPU 耗时。

| 想回答的问题 | 适合的方式 |
|-|-|
| 某个时间段 CPU 主要花在哪些线程、方法及调用路径 | Time Profiler，选中目标时间范围后看调用树 |
| 这一次快照调用从开始到返回经过多久 | 成对 Signpost 记录调用区间 |
| 每次已标记调用的开始、结束、持续时间及顺序 | Signpost + Instruments 时间线；只覆盖主动标记的操作 |
| 动画是否错过显示时机，问题位于提交还是渲染阶段 | Animation Hitches，再结合调用栈和阶段标记定位 |

## Animation Hitches 的使用流程

**Animation Hitches 是 Apple 的 Instruments 中用于分析动画、滑动卡顿的模板。** Hitch 指运动中的画面没有按预期时机更新，例如滑动评论列表时，新位置对应的画面没及时准备好，旧画面继续停留，视觉上就顿了一下。相关阶段见[Core Animation 渲染管线](#图层动画与渲染管线)。

使用流程：打开 Instruments，选择 Animation Hitches 模板和目标 App，开始录制，再在设备上执行要排查的滑动或动画。工具在时间线上标出卡顿，并提供提交、渲染、GPU、帧生命周期等轨道，帮助判断卡顿发生的时间、持续时长及主要耗时阶段。

以 Push 评论直达为例，可以录制一次自动滚到目标评论的过程，选中停顿所在的时间段，查看 App 的事件处理／布局／提交，还是后续渲染准备／GPU 执行出现延迟。进一步定位具体业务代码时，还要结合 [Time Profiler](#time-profiler-的采样与统计边界) 等工具中的调用栈和线程状态；工具标出阶段，不等于自动给出完整根因。

[CADisplayLink](threads-and-scheduling.md#cadisplaylink-的原理与适用场景) 可通过回调间隔观察流畅度迹象；Animation Hitches 则用于录制、分析一次具体交互的渲染时间线。

## MetricKit 与 Xcode Organizer

**MetricKit 是 App 中接收系统性能与诊断报告的框架；Xcode Organizer 是开发者在 Xcode 中查看 App 数据的窗口。** 这里讨论的是 Organizer 的性能指标与诊断功能。

| 对比项 | MetricKit | Xcode Organizer |
|-|-|-|
| 使用方式 | 在 App 中接入，接收系统提供的性能指标和诊断报告 | 在 Xcode 的 Window → Organizer 中查看 |
| 数据处理 | App 按需保存、上传到自己的监控平台，再做统计或告警 | Apple 收集并汇总数据，开发者查看指标及可用的诊断报告 |
| 典型用途 | 接入公司的线上性能监控体系 | 比较版本、机型上的启动耗时、内存、卡顿及无响应表现 |

**MetricKit 不负责自动上传到公司的平台。** 上传需要业务代码或监控 SDK 实现，可以组成下面的链路；这是能力说明，不代表本项目已接入：

```text
系统收集数据 → MetricKit 将报告交给 App
            → 公司监控 SDK／业务代码上传 → 平台统计、展示和告警
```

MetricKit 提供系统收集的报告，不是实时逐帧回调。查看 Organizer 的系统性能指标通常不需要专门接入 MetricKit，两者也不是“MetricKit 上传后 Organizer 才有数据”的必需前后关系。

**线上性能数据的取得条件：** 以通过 App Store 发布的 App 为例，Organizer 可显示参与数据共享用户的匿名汇总性能表现。

1. App 上架 App Store 后，用户下载安装并使用。
2. 其中同意向开发者共享相应分析与诊断数据的用户，其设备向 Apple 提供相关数据。
3. Apple 汇总报告；开发者在 Xcode 登录有权限的团队账号后，通过 Organizer 查看。

例如可以比较新旧版本的启动耗时、内存和卡顿指标，并在支持的指标上按机型筛选。这里看到的是统计结果，不能据此逐个查看某个用户的身份及实时运行情况，也不能把一万名使用者理解成一万名都进入统计。

数据需要足够的使用样本，且发布后到报告可见存在延迟；只装给几个朋友测试，不能保证出现性能指标。Apple 的 Metrics 报告面向 App Store 分发；TestFlight 可提供崩溃等报告，不能由此推断全部性能指标都适用相同渠道。线上指标有助于发现某个版本发生性能退化，具体原因仍需结合诊断报告与复现分析。
