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

ViewHolder 规范了列表项的管理，并减少重复的控件查找；复用机制减少视图创建。布局计算、图片解码、绑定逻辑和动画仍然有开销，所以 RecyclerView 并非在所有场景下都比 ListView 快。
