# Android 列表复用：ListView、RecyclerView 与 ViewHolder

[返回 Android 分类目录](README.md)

`ListView` 和 `RecyclerView` 都是用于展示数据集合的 Android 视图容器，也都支持列表项视图复用。复用指把已经创建的列表项视图重新用于展示其他数据，以减少重复创建视图的成本。主要区别在于布局职责、适配器接口和更新机制。本文讨论传统 Android View 体系。

## 主要区别

适配器（`Adapter`）负责向列表提供数据项数量，并把数据配置到对应视图中。布局管理器（`LayoutManager`）负责决定 RecyclerView 中列表项的排列、尺寸、位置及滚动布局。

| 对比项 | ListView | RecyclerView |
| --- | --- | --- |
| 排列方式 | 主要提供垂直列表。 | 通过 LayoutManager 支持垂直列表、水平列表、网格和瀑布流等排列。 |
| 创建与配置 | 在 `getView(position, convertView, parent)` 中复用或创建视图，再配置数据。 | 通过 `onCreateViewHolder` 创建视图和 ViewHolder，通过 `onBindViewHolder` 配置数据。 |
| ViewHolder | 可以自行使用 ViewHolder 模式，接口不强制要求。 | Adapter 接口以 `RecyclerView.ViewHolder` 为列表项管理对象。 |
| 数据更新 | 常用 `notifyDataSetChanged()` 通知数据集变化。 | 可通过 `notifyItemChanged`、`notifyItemInserted`、`notifyItemRemoved` 等描述具体变化。 |
| 扩展职责 | 列表排列、分隔线等功能主要由 ListView 自身提供。 | LayoutManager 管布局，ItemDecoration 提供间距或装饰绘制，ItemAnimator 管列表项变化动画。 |

ListView 已经能够通过 `convertView` 复用视图。RecyclerView 的关键改进是把创建、绑定、布局和动画等职责分开，并提供明确的扩展接口。不能把两者区别概括为“ListView 不复用，RecyclerView 才复用”。

## ViewHolder 的职责与收益

`RecyclerView.ViewHolder` 是持有列表项根视图并保存该项管理信息的对象。根视图通过 `itemView` 访问。开发者通常在自定义 ViewHolder 中进一步保存标题、图片等内部控件的引用；对象引用是程序中用于访问某个对象的值。

`findViewById` 根据资源 ID 在视图及其子视图中查找控件。创建 ViewHolder 时查找并保存控件引用，后续绑定数据时直接使用这些引用，就能减少重复查找。例如先保存标题控件为 `holder.title`，绑定新数据时直接执行 `holder.title.text = newTitle`。

RecyclerView 要求使用 ViewHolder，但不会自动为每个内部控件生成缓存字段。如果每次绑定仍然调用 `itemView.findViewById`，这些重复查找仍然存在。ListView 也可以自行使用 ViewHolder 模式并保存控件引用，从而获得相同的查找成本收益。

ViewHolder 还为 RecyclerView 提供统一的列表项管理对象。RecyclerView 可以围绕同一个 ViewHolder 管理绑定、复用和动画状态。创建视图与绑定数据分开后，固定的视图结构只需在创建时建立，数据变化时更新展示内容。

保存内部控件引用的做法，类似在自定义 `UITableViewCell` 或 `UICollectionViewCell` 中保存 `titleLabel`、`imageView` 属性。这个类比只说明引用的保存方式；RecyclerView.ViewHolder 本身不是 View，也不等同于 UIKit 的 Cell。

## 创建、绑定与复用流程

1. RecyclerView 需要一个新 ViewHolder 时，调用 Adapter 的 `onCreateViewHolder`，传入父容器和视图类型。Adapter 创建列表项根视图及 ViewHolder，再返回 ViewHolder。
2. RecyclerView 需要配置某个数据位置的内容时，调用 Adapter 的 `onBindViewHolder`，传入 ViewHolder 和数据位置。Adapter 读取对应数据并更新控件。
3. 列表滚动后，不再用于当前显示的视图可以进入缓存或复用流程。RecyclerView 之后可以复用兼容视图类型的 ViewHolder，减少新建对象。
4. 同一个 ViewHolder 被用于另一项数据时，Adapter 必须更新该项依赖数据的展示状态，包括文字、图片、选中状态和显隐状态，避免保留上一项内容。

一个 ViewHolder 可以先展示数据 A，之后再展示数据 B。ViewHolder 数量不等于数据总数，也不是每次绑定都会新建 ViewHolder。缓存命中且状态仍有效时，RecyclerView 还可能直接使用已有内容；不能假定每次重新可见都会触发完整绑定。

## 局部更新与性能边界

局部更新指告诉列表哪些位置发生了何种变化。例如先修改适配器使用的数据，再调用 `notifyItemChanged(position)`，通知指定位置的内容已改变。通知方法不会替开发者修改数据。`DiffUtil` 可以比较新旧数据列表并计算变化，但需要明确数据项身份和内容比较规则。

`notifyItemChanged` 表示项目内容变化，不保证只更新项目内某一个控件。需要只绑定部分内容时，可以使用 payload；payload 是随更新通知传递的变化说明，Adapter 需要在相应绑定方法中读取并处理。

ViewHolder 主要帮助规范列表项管理并减少重复控件查找。复用机制主要减少视图创建。布局计算、图片解码、绑定逻辑和动画仍然可能产生开销，因此 RecyclerView 不保证在所有场景下都比 ListView 更快。
