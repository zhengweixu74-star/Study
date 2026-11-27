# Android 列表复用：ListView、RecyclerView 与 ViewHolder

[返回 Android 分类目录](README.md)

`ListView` 和 `RecyclerView` 都是展示数据集合的 Android 视图容器，都支持列表项视图复用。复用指把已经创建的列表项视图拿来展示其他数据，从而少创建视图。两者的主要区别在于布局职责、适配器接口和更新机制。本文讨论传统 Android View 体系。

## 主要区别

适配器（`Adapter`）告诉列表一共有多少个数据项，并把每个数据项填入对应视图。布局管理器（`LayoutManager`）决定 RecyclerView 中列表项的排列、尺寸、位置和滚动布局。

| 对比项 | ListView | RecyclerView |
| --- | --- | --- |
| 排列方式 | 主要提供垂直列表。 | 通过 LayoutManager 支持垂直列表、水平列表、网格和瀑布流等排列。 |
| 创建与配置 | 在 `getView(position, convertView, parent)` 中复用或创建视图，再配置数据。 | 通过 `onCreateViewHolder` 创建视图和 ViewHolder，通过 `onBindViewHolder` 配置数据。 |
| ViewHolder | 可以自行使用 ViewHolder 模式，接口不强制要求。 | Adapter 接口以 `RecyclerView.ViewHolder` 为列表项管理对象。 |
| 数据更新 | 常用 `notifyDataSetChanged()` 通知数据集变化。 | 可通过 `notifyItemChanged`、`notifyItemInserted`、`notifyItemRemoved` 等描述具体变化。 |
| 扩展职责 | 列表排列、分隔线等功能主要由 ListView 自身提供。 | LayoutManager 管布局，ItemDecoration 提供间距或装饰绘制，ItemAnimator 管列表项变化动画。 |

ListView 通过 `convertView` 复用视图，两者都会复用。RecyclerView 的关键改进是把创建、绑定、布局和动画拆给不同对象负责，并为每项职责提供明确的扩展接口。

## Adapter 的职责与 UITableViewDataSource 对照

适配器模式在两个接口不匹配的对象之间放一个转换对象，让双方能够协作。RecyclerView 只管理一行行 View，不知道如何显示 `List<Note>` 中的一条 `Note`。Adapter 把数据转换成 RecyclerView 可以显示的行，用三个方法回答 RecyclerView 的问题：

| 方法 | 回答的问题 | 示例 |
| --- | --- | --- |
| `getItemCount()` | 一共有几行 | 返回 `notes.size` |
| `onCreateViewHolder()` | 每一行使用什么 View | 加载 `item_note.xml`，再包装成 `NoteViewHolder` |
| `onBindViewHolder()` | 第 N 行显示什么内容 | 把第 N 条 `Note` 的标题写入标题控件 |

UIKit 中承担相同职责的是 `UITableViewDataSource`，但两者的类型不同：

| 对比项 | UITableViewDataSource | RecyclerView.Adapter |
| --- | --- | --- |
| 类型 | Objective-C 协议 | 抽象类 |
| 实现方式 | 某个类声明遵守协议，并实现协议方法 | 编写子类继承 `RecyclerView.Adapter` |
| 常见写法 | ViewController 遵守协议，自己充当数据源 | 单独编写 `NotesAdapter`，页面创建 `NotesAdapter` 对象后交给 RecyclerView |
| 方法实现 | 协议不提供方法实现 | 抽象类自带 `notifyItemChanged()` 等通知刷新的方法 |

协议只规定需要实现哪些方法，协议本身不能创建对象。抽象类不能直接创建对象，需要由子类继承；抽象类可以包含已实现的方法。`UITableViewDataSource` 继承自 `NSObjectProtocol`，只有类可以遵守，结构体不能遵守。

```swift
class NotesViewController: UIViewController, UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { notes.count }
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell { ... }
}
// tableView.dataSource = self
```

```kotlin
class NotesAdapter : RecyclerView.Adapter<NoteViewHolder>() {
    override fun getItemCount(): Int = notes.size
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): NoteViewHolder { ... }
    override fun onBindViewHolder(holder: NoteViewHolder, position: Int) { ... }
}
// recyclerView.adapter = NotesAdapter()
```

## ViewHolder 的职责与收益

`RecyclerView.ViewHolder` 持有列表项的根视图，并保存这一项的管理信息。根视图通过 `itemView` 访问。开发者通常在自定义 ViewHolder 中再保存标题、图片等内部控件的引用。对象引用是程序访问某个对象时使用的值。

`findViewById` 根据资源 ID 在一个视图及其子视图中查找控件。创建 ViewHolder 时查找控件并保存引用，之后绑定数据时直接使用引用，就省去了重复查找。例如创建时把标题控件保存为 `holder.title`，绑定新数据时直接执行 `holder.title.text = newTitle`。

RecyclerView 强制使用 ViewHolder，但不会替开发者给内部控件生成缓存字段。如果每次绑定都调用 `itemView.findViewById`，查找开销依然存在。ListView 自行使用 ViewHolder 模式并保存控件引用，也能省下同样的查找开销。

ViewHolder 还是 RecyclerView 管理列表项的统一单位。RecyclerView 以 ViewHolder 为单位管理绑定、复用和动画状态。创建视图与绑定数据分成两步后，固定的视图结构只在创建时搭建一次，数据变化时只更新展示内容。

在 ViewHolder 中保存内部控件引用，类似在自定义 `UITableViewCell` 或 `UICollectionViewCell` 中保存 `titleLabel`、`imageView` 属性。两者只在保存引用的方式上相似：RecyclerView.ViewHolder 是普通对象，UIKit 的 Cell 是视图。

## ViewHolder 与 ViewGroup 的区别

ViewGroup 是一种 View，显示在屏幕上，负责容纳并摆放子 View。ViewHolder 是普通对象，不显示在屏幕上，只保存某一行中各个 View 的引用。两者名称都含有 View，但属于不同类别的对象。

一行列表项的根视图通常是 ViewGroup，这一行同时由一个 ViewHolder 管理。例如 `item_note.xml` 用 LinearLayout 上下排列标题和日期：

```xml
<LinearLayout android:orientation="vertical" ...>
    <TextView android:id="@+id/title" ... />
    <TextView android:id="@+id/date" ... />
</LinearLayout>
```

```kotlin
class NoteViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    val titleView: TextView = itemView.findViewById(R.id.title)
    val dateView: TextView = itemView.findViewById(R.id.date)
}
```

屏幕上显示两行笔记时，对象关系如下：

```text
View 树（显示在屏幕上）                    View 树之外的普通对象

RecyclerView（ViewGroup）
├── LinearLayout（第 1 行根视图）   ◀── NoteViewHolder 1 的 itemView
│   ├── TextView 标题               ◀── NoteViewHolder 1 的 titleView
│   └── TextView 日期               ◀── NoteViewHolder 1 的 dateView
└── LinearLayout（第 2 行根视图）   ◀── NoteViewHolder 2 的 itemView
    ├── TextView 标题               ◀── NoteViewHolder 2 的 titleView
    └── TextView 日期               ◀── NoteViewHolder 2 的 dateView
```

| 对比项 | ViewGroup | ViewHolder |
| --- | --- | --- |
| 是否为 View | 是，继承自 View | 否，是普通对象 |
| 是否显示在屏幕上 | 是 | 否 |
| 是否属于 View 树 | 是 | 否 |
| 职责 | 容纳子 View，决定子 View 的尺寸和位置 | 保存一行中各个 View 的引用，作为 RecyclerView 复用的单位 |
| 示例 | `LinearLayout`、`RecyclerView` | `NoteViewHolder` |

每一行都有 id 为 `title` 的控件，所以 `NoteViewHolder` 调用 `itemView.findViewById()`，把查找范围限定在当前行，见[从资源 ID 获取内容](resources-and-r-class.md#从资源-id-获取内容)。

## 创建、绑定与复用流程

1. RecyclerView 需要新的 ViewHolder 时，调用 Adapter 的 `onCreateViewHolder`，传入父容器和视图类型。Adapter 创建列表项根视图和 ViewHolder，返回 ViewHolder。
2. RecyclerView 要显示某个位置的数据时，调用 Adapter 的 `onBindViewHolder`，传入 ViewHolder 和数据位置。Adapter 读取该位置的数据，更新控件。
3. 列表滚动后，移出屏幕的 ViewHolder 进入缓存或回收池。之后 RecyclerView 取出视图类型相同的 ViewHolder 重新使用，不再新建对象。
4. 同一个 ViewHolder 改为展示另一项数据时，Adapter 必须重设所有随数据变化的展示状态，包括文字、图片、选中状态和显隐状态。漏设的状态会保留上一项的内容。

一个 ViewHolder 可以先展示数据 A，再展示数据 B。所以 ViewHolder 的数量通常少于数据总数，绑定时也不一定新建 ViewHolder。如果 ViewHolder 从缓存中取出且内容仍然有效，RecyclerView 可能直接显示，不调用 `onBindViewHolder`；列表项重新可见时不一定触发完整绑定。

## 局部更新与性能边界

局部更新指告诉列表哪些位置发生了哪种变化。例如先修改 Adapter 使用的数据，再调用 `notifyItemChanged(position)`，通知该位置的内容已改变。通知方法只发出通知，数据要由开发者自己修改。`DiffUtil` 比较新旧两个数据列表并计算变化，开发者需要提供两条规则：如何判断两项是同一条数据，如何判断两项内容相同。

`ListAdapter` 是封装了 `DiffUtil` 的 `RecyclerView.Adapter` 子类。调用 `submitList(newList)` 提交新列表后，`ListAdapter` 在后台线程比较新旧列表，再在主线程把具体变化通知 RecyclerView，只刷新有变化的行。比较规则由开发者传入的 `DiffUtil.ItemCallback` 决定。

`notifyItemChanged` 通知的是整个列表项的内容变化，Adapter 默认会重新绑定这一项的全部内容。只想更新部分内容时，可以传入 payload。payload 是随更新通知一起传递的变化说明，Adapter 在带 payload 参数的 `onBindViewHolder` 中读取 payload，只更新对应控件。

提交数据和创建行 View 发生在不同时刻。调用 `submitList()` 或 `notifyDataSetChanged()` 后，Adapter 只保存新数据，并通知 RecyclerView 在下一帧重新布局。到下一帧布局时，RecyclerView 才为可见位置调用 `onCreateViewHolder` 和 `onBindViewHolder`。所以提交数据后立即查询某一行的 ViewHolder，结果可能是 `null`；要读取行的位置和高度，需等下一次布局完成，例如在 `doOnPreDraw` 回调中读取。`doOnPreDraw` 在 View 下一次绘制前调用传入的代码，这时布局已经完成。

ViewHolder 规范了列表项的管理，并减少重复的控件查找；复用机制减少视图创建。布局计算、图片解码、绑定逻辑和动画仍然有开销，所以 RecyclerView 并非在所有场景下都比 ListView 快。

## RecyclerView 的滚动过程：回收、填充与平移

RecyclerView 只为可见位置准备行 View，所以滚动时不能只修改一个滚动偏移，还要拆掉移出屏幕的行、补上新露出的行。`recyclerView.scrollBy(0, dy)` 的工作由 LayoutManager 完成；竖直排列的 `LinearLayoutManager` 在 `scrollVerticallyBy()` 中按顺序做三步：

1. 回收：按“移动 dy 之后”的位置判断，把完全移出列表顶部的行从 RecyclerView 上拆下，放进缓存。
2. 填充：给底部新露出的空位补行。先从缓存取出或调用 `onCreateViewHolder` 新建行 View，再调用 `onBindViewHolder` 绑定数据，最后测量并布局这一行。
3. 平移：对每一行调用 `offsetTopAndBottom(-dy)`，把行的 top 和 bottom 各减去 dy。

`offsetTopAndBottom()` 只更新这一行 RenderNode 中的位置，不重新测量、布局或执行 `onDraw()`，开销与修改 `translationY` 同级。RenderNode 见[硬件加速与 RenderNode](view-drawing-and-frame-refresh.md#硬件加速与-rendernode)。

例如列表高 1000 px，每行高 200 px，第 0～4 行刚好填满列表。执行 `scrollBy(0, 150)`：

| 步骤 | 结果 |
| --- | --- |
| 回收 | 第 0 行移动后为 top −150、bottom 50，仍露出 50 px，不回收 |
| 填充 | 底部空出 850～1000，补上第 5 行，布局在 top 1000、bottom 1200 |
| 平移 | 第 0～5 行全部上移 150 px，第 5 行变为 top 850、bottom 1050 |

下一帧再滚 100 px 时，第 0 行的 bottom 变为 −50，整行不可见，第 0 行的 View 进入缓存，之后可以复用给其他位置。

三步中填充最耗时：绑定数据执行业务代码，测量和布局还要遍历这一行的 View 树。所以滚动开销主要取决于这一帧有多少新行露出。上例每滚过 200 px 补一行，连续滚动 2000 px 约绑定并布局 10 行。

## 跳转到指定位置：scrollToPositionWithOffset

`LinearLayoutManager.scrollToPositionWithOffset(position, offset)` 不带动画，把第 `position` 项直接排到距离列表顶部 `offset` 像素的位置。调用时列表不会立即移动：LayoutManager 记下目标位置，下一次布局时从第 `position` 项开始摆放。

例如屏幕上显示第 0～4 行，调用 `scrollToPositionWithOffset(50, 400)`。下一次布局后，第 50 行的 top 为 400，第 48、49 行排在上方，第 51、52 行排在下方；第 5～47 行不创建，也不绑定。

目标行不在屏幕上时，RecyclerView 没有目标行的 View，查询目标行 ViewHolder 的结果是 `null`，代码读不到目标行的位置，也就算不出平滑滚动的距离。从第 0 行平滑滚到第 50 行还要逐行完成填充，行高不一时总距离也无法预先算出。常见做法是：

1. 调用 `scrollToPositionWithOffset()`，把目标行先排到附近。
2. 等下一次布局完成，例如在 `doOnPreDraw` 回调中执行后续代码。
3. 读取目标行的 top 和高度，计算剩余距离，再播放平滑滚动动画。

第 3 步要读取目标行的实际位置，因为 LayoutManager 会修正请求的位置：按请求摆放后列表底部会留出空白时，`LinearLayoutManager` 把所有行整体下移，用上方的行填满空白。例如列表高 1800 px，共 50 行，目标是最后一行第 49 行，行高 200 px：

1. 调用 `scrollToPositionWithOffset(49, 760)`，按请求摆放时第 49 行的 top 为 760、bottom 为 960。
2. 第 49 行下方没有更多行，960～1800 之间会空出 840 px。
3. `LinearLayoutManager` 把所有行下移 840 px，第 49 行的 top 实际为 1600。

所以计算时使用布局完成后读出的位置。读取 View 在屏幕上的位置见[屏幕坐标与目标滚动位置](view-drawing-and-frame-refresh.md#屏幕坐标与目标滚动位置)。

## 滚动状态：IDLE、DRAGGING 与 SETTLING

RecyclerView 用滚动状态描述列表是否在滚动、由谁驱动：

| 常量 | 含义 | 例子 |
| --- | --- | --- |
| `SCROLL_STATE_IDLE` | 列表静止 | 没有触摸，也没有自动滚动 |
| `SCROLL_STATE_DRAGGING` | 手指按在列表上拖动列表 | 用户正在拖动 |
| `SCROLL_STATE_SETTLING` | 手指已经离开，列表自己继续滚向停止位置 | 抛动后的惯性滑动、`smoothScrollToPosition()` 的自动滚动 |

列表处于 `SETTLING` 时用户按下手指，RecyclerView 切换到 `DRAGGING`，并停止正在进行的惯性滑动或自动滚动。

RecyclerView 通过 `OnScrollListener` 通知滚动状态的变化。需要在滚动停下后执行代码时，在状态变为 `IDLE` 时执行：

```kotlin
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        if (newState == RecyclerView.SCROLL_STATE_IDLE) {
            // 滚动已经停下
        }
    }
})
```

## 平滑滚动到指定位置：smoothScrollToPosition

`recyclerView.smoothScrollToPosition(position)` 让列表带动画滚到第 `position` 项。开始滚动时，目标行通常还没有 View，中间各行的高度也没有测量过，所以 RecyclerView 不知道目标行有多远。iOS 的 `UITableView.scrollToRow(at:at:animated:)` 用预估行高计算未显示行的位置；RecyclerView 改用“先朝目标方向滚，看到目标行后再精确停下”的方式。

### 执行过程

`LinearSmoothScroller` 是负责逐帧滚向目标行的对象。例如屏幕上显示第 0～4 行，调用 `smoothScrollToPosition(30)`：

1. `LinearLayoutManager` 创建一个 `LinearSmoothScroller`，把目标设为第 30 项并启动，列表进入 `SETTLING` 状态。
2. 找目标阶段：`LinearSmoothScroller` 比较 30 与屏幕上第一行的位置 0，判断目标在下方，开始向下匀速滚动，每帧推进一段。
3. 每帧滚动都经历回收、填充、平移三步，见上文[RecyclerView 的滚动过程](#recyclerview-的滚动过程回收填充与平移)。每填充一行，`LinearSmoothScroller` 检查新行是不是第 30 项。
4. 第 30 项被布局出来时，系统调用 `LinearSmoothScroller` 的 `onTargetFound()`。这时第 30 项已有真实位置，`LinearSmoothScroller` 算出剩余的精确距离。
5. `LinearSmoothScroller` 用减速插值器滚完剩余距离，列表停下，状态回到 `IDLE`。

`smoothScrollToPosition()` 由 LayoutManager 驱动，滚动的是 RecyclerView 自身的内容。外层还有滚动容器、需要外层和列表按同一进度接力时，可以先用上文的 `scrollToPositionWithOffset()` 排好目标行，再用一个 ValueAnimator 分配两层的滚动距离，见[把动画进度转换成滚动距离](valueanimator-and-interpolation.md#把动画进度转换成滚动距离)。

### 默认停靠位置

`LinearSmoothScroller` 的 `getVerticalSnapPreference()` 决定目标行最后停在哪里。默认实现按滚动方向返回：

| 情况 | 返回值 | 目标行停靠的位置 |
| --- | --- | --- |
| 目标在下方，向下找到目标 | `SNAP_TO_END` | 目标行刚好完整出现在列表底部 |
| 目标在上方，向上找到目标 | `SNAP_TO_START` | 目标行停在列表顶部 |
| 开始时目标行已经布局出来 | `SNAP_TO_ANY` | 已完整显示时不滚动；部分显示时只滚动到刚好完整显示 |

例如从第 0 行滚到第 30 行，默认结果是第 30 行贴着列表底边显示。

### 滚动速度与自定义

`LinearSmoothScroller` 的 `calculateSpeedPerPixel(displayMetrics)` 返回滚过 1 px 需要的毫秒数。默认值是 `25 ÷ densityDpi`，即每滚过 1 英寸用 25 ms。`densityDpi` 与 `DisplayMetrics` 见[尺寸单位：px、dp 与 sp](view-system-and-layout.md#尺寸单位pxdp-与-sp)。例如 `densityDpi` 为 440 时，找目标阶段滚过 3000 px 约 6.8 英寸，用时约 170 ms，所以长距离的平滑滚动看起来很快。

要改变停靠位置或速度，创建 `LinearSmoothScroller` 的子类并交给 LayoutManager 启动。`object : 父类(...) { ... }` 的写法见[对象表达式](../language/kotlin-basics.md#对象表达式创建匿名子类的对象)。

```kotlin
val scroller = object : LinearSmoothScroller(recyclerView.context) {
    // 目标行总是停在列表顶部
    override fun getVerticalSnapPreference(): Int = SNAP_TO_START

    // 每滚过 1 英寸用 100 ms，比默认值慢 4 倍
    override fun calculateSpeedPerPixel(displayMetrics: DisplayMetrics): Float =
        100f / displayMetrics.densityDpi
}
scroller.targetPosition = 30
recyclerView.layoutManager?.startSmoothScroll(scroller)
```

`smoothScrollToPosition(30)` 内部做的是同样的事，只是使用默认的 `LinearSmoothScroller`。
