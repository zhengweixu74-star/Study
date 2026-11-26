# UICollectionView 与布局

[返回笔记目录](../README.md)

## 本文目录

- [UICollectionView 的职责与复用](#uicollectionview-的职责与复用)
- [分区、项目与 IndexPath](#分区项目与-indexpath)
- [数据源与代理的五个重点回调](#数据源与代理的五个重点回调)
- [UICollectionViewFlowLayout 的排列规则](#uicollectionviewflowlayout-的排列规则)
  - [按行排列与滚动方向](#按行排列与滚动方向)
  - [行间距与项目间距](#行间距与项目间距)
  - [分区边距](#分区边距)
  - [代理协议的继承与布局回调](#代理协议的继承与布局回调)
- [UICollectionViewCompositionalLayout 的层级组合](#uicollectionviewcompositionallayout-的层级组合)

## UICollectionView 的职责与复用

UICollectionView 显示一组可滚动的内容。各部分分工如下：Cell 显示一项内容；DataSource 提供数量和配置好的 Cell；Delegate 处理点击等交互；Layout 提供每一项的位置和尺寸。数据按 section／item 组织，IndexPath 只表示当前位置，不能当作数据的唯一标识。

使用流程：创建并设置布局 → 注册 Cell → 设置数据源和代理 → 按需取出并配置 Cell → 按布局显示。滚动时重用 Cell，不会为所有数据一次性创建视图；每次配置都要设置全部状态，避免残留旧内容。

SnapKit 负责 UICollectionView 在页面中的位置，以及 Cell 内部控件的约束；Cell 在列表中怎么排列，由 UICollectionViewLayout 负责。UICollectionView 自身不计算 Cell 的位置：布局对象为每个项目计算 frame，UICollectionView 再按这些 frame 放置 Cell。UIKit 提供 UICollectionViewFlowLayout 和 UICollectionViewCompositionalLayout 两种布局类，见下文 [UICollectionViewFlowLayout 的排列规则](#uicollectionviewflowlayout-的排列规则) 和 [UICollectionViewCompositionalLayout 的层级组合](#uicollectionviewcompositionallayout-的层级组合)；也可以继承 UICollectionViewLayout 自定义布局。

## 分区、项目与 IndexPath

分区（section）是对数据的一层分组；项目（item）是分区中的一项数据，Cell 是显示这个项目的可重用视图。一个分区可以显示多行 Cell，可以有头部、尾部，也可以没有。

例如，同一个图片列表可以把“今天”和“昨天”各作为一个分区：

```text
同一个 UICollectionView
├─ section 0：今天
│  ├─ Header：今天（可选）
│  ├─ 第一行：[item 0] [item 1] [item 2]
│  ├─ 第二行：[item 3] [item 4]
│  └─ Footer（可选）
└─ section 1：昨天
   ├─ Header：昨天（可选）
   └─ 第一行：[item 0] [item 1]
```

每个分区的 item 索引独立从 0 开始。IndexPath(item: 1, section: 0) 表示第一个分区中的第二项，不是整个列表的第二行。

## 数据源与代理的五个重点回调

数据源向 UICollectionView 提供数量和配置好的 Cell；代理接收点击、显示变化等通知，也可以通过部分方法的返回值做出决定（例如是否允许选中）。下表是直接实现 UICollectionViewDataSource 时最常用的方法。

| 方法 | 所属协议 | 使用场景 |
| --- | --- | --- |
| collectionView(_:numberOfItemsInSection:) | UICollectionViewDataSource | 返回指定分区的项目数，例如当前分区有 20 张图片就返回 20 |
| collectionView(_:cellForItemAt:) | UICollectionViewDataSource | 获取复用 Cell，根据 indexPath 对应的数据设置图片、标题及其他展示状态，再返回 Cell |
| collectionView(_:didSelectItemAt:) | UICollectionViewDelegate | 用户选中项目后，读取对应数据并执行选择动作，例如打开详情 |
| collectionView(_:didEndDisplaying:forItemAt:) | UICollectionViewDelegate | Cell 结束显示时，停止该 Cell 的动画等显示相关工作 |
| scrollViewDidScroll(_:) | UIScrollViewDelegate | 滚动位置变化时读取 contentOffset，例如按位置调整导航栏透明度 |

前两个方法是 UICollectionViewDataSource 的必需方法。多个分区还需要通过 numberOfSections(in:) 提供分区数量；未实现该可选方法时，默认按一个分区处理。

cellForItemAt 只是准备 Cell，用户不一定看到了；UIKit 可能为了预取提前调用它，所以不能在这里统计曝光。willDisplay 表示 Cell 即将显示，也不能说明可见面积和停留时间已经达到曝光标准。

didEndDisplaying 传入的是结束显示的 Cell。此时数据可能已经插入、删除或重排，要找到对应的数据，应使用 Cell 上保存的数据 ID，而不是按 indexPath 读取当前数组。

用户拖动、松手后的减速滚动、代码修改滚动位置，都会触发 scrollViewDidScroll。contentOffset 是当前的滚动偏移。这个回调调用非常频繁，里面只能做轻量的界面更新。

## UICollectionViewFlowLayout 的排列规则

UICollectionViewFlowLayout 是 UIKit 提供的流式布局类：按顺序把项目放进一行，当前行放不满就继续放，放不下就换行。本章依次说明行的方向、项目之间的两种间距、分区四周的边距，以及按项目或分区返回不同尺寸的代理回调。

### 按行排列与滚动方向

UICollectionViewFlowLayout 对每个分区按以下顺序计算项目位置：

1. 按 item 索引顺序读取每个项目的尺寸。尺寸来自 layout 的 itemSize，或代理方法 sizeForItemAt 的返回值。
2. 把项目依次放进当前行，相邻项目之间至少留 minimumInteritemSpacing。
3. 当前行剩余空间放不下下一个项目时，换到下一行，相邻两行之间至少留 minimumLineSpacing。
4. 一个分区排完后，下一个分区从新的一行开始，分区之间不共用行。

所以一个分区可以有多行，分区和行是两回事。

“行”的方向由 scrollDirection 决定：

| scrollDirection | 一行的方向 | 行的排列方向与滚动方向 |
| --- | --- | --- |
| .vertical（默认） | 横向一排 | 行从上往下排，列表竖向滚动 |
| .horizontal | 竖向一列 | 行从左往右排，列表横向滚动 |

竖向滚动时，项目在一排中从左往右放，排满换到下一排。横向滚动时，项目在一列中从上往下放，排满换到右边的下一列。

### 行间距与项目间距

minimumLineSpacing 是相邻两行之间的最小距离。minimumInteritemSpacing 是同一行中相邻两个项目之间的最小距离。两个值都是最小值，一行中的项目没有占满整行时，布局对象可以把项目之间的实际距离排得比 minimumInteritemSpacing 更大。

两个属性控制的方向随 scrollDirection 变化：

| scrollDirection | minimumLineSpacing 控制 | minimumInteritemSpacing 控制 |
| --- | --- | --- |
| .vertical | 上下两排之间的距离 | 同一排中左右相邻项目的距离 |
| .horizontal | 左右两列之间的距离 | 同一列中上下相邻项目的距离 |

示例：一个横向滚动的单行卡片列表。

```swift
let layout = UICollectionViewFlowLayout()
layout.scrollDirection = .horizontal
layout.itemSize = CGSize(width: 110, height: 160)
layout.minimumLineSpacing = 12
layout.minimumInteritemSpacing = 8
layout.sectionInset = UIEdgeInsets(top: 8, left: 16, bottom: 8, right: 16)
```

假设 UICollectionView 的高度是 176，排列结果如下：

1. 上下边距各占 8，留给项目的高度是 176 - 8 - 8 = 160。
2. 一张卡片高 160，一列只放得下一张卡片，所以每张卡片单独占一列。
3. 左右相邻两列之间按 minimumLineSpacing 留 12。第 1 张卡片的 x 是 16，第 2 张是 16 + 110 + 12 = 138，第 3 张是 138 + 110 + 12 = 260。
4. minimumInteritemSpacing 控制同一列中上下两个项目的距离。这个例子中每列只有一个项目，卡片之间的左右距离由 minimumLineSpacing 决定。

把 UICollectionView 的高度改成 344，留给项目的高度是 344 - 8 - 8 = 328，正好放下两张卡片和中间的间距（160 + 8 + 160 = 328）。这时每列放两张卡片：同一列上下两张卡片之间按 minimumInteritemSpacing 留 8，左右两列之间仍按 minimumLineSpacing 留 12。

### 分区边距

sectionInset 是分区内容四周的边距。在垂直布局中，top 是头部与第一行 Cell 之间的留白，bottom 是最后一行 Cell 与尾部之间的留白，left 和 right 是两侧的留白。没有头部或尾部时，top 和 bottom 就是分区的上下边缘留白。

```swift
layout.sectionInset = UIEdgeInsets(top: 10, left: 16, bottom: 10, right: 16)
```

这里的 layout 是 UICollectionViewFlowLayout。边距从哪里算起，还受 sectionInsetReference 影响。sectionInset 控制分区内容与分区边缘的距离，项目之间的距离由上文的 minimumLineSpacing 和 minimumInteritemSpacing 控制。

### 代理协议的继承与布局回调

UICollectionViewDelegate 继承 UIScrollViewDelegate，所以 collectionView.delegate 指向的对象，可以同时实现选中回调和滚动回调，不需要另外设置滚动代理。

UICollectionViewDelegateFlowLayout 又继承 UICollectionViewDelegate。使用 UICollectionViewFlowLayout 时，可以实现 sizeForItemAt 为每一项提供不同的尺寸，或实现 insetForSectionAt 为每个分区提供不同的边距。所有项目尺寸和边距都一样时，直接设置 layout 的 itemSize 和 sectionInset 即可。

这些回调只对 UICollectionViewFlowLayout 有效，其他布局用各自的接口配置。以上代理协议中的方法都是可选的，按需实现即可。

协议继承的通用规则见[Swift 基础语法与类型系统中的协议继承](../language/swift-basics.md#协议继承与类继承的区别)。

## UICollectionViewCompositionalLayout 的层级组合

UICollectionViewCompositionalLayout 是 iOS 13 起提供的布局类。布局由三层对象一层层组合：

- NSCollectionLayoutItem：描述一个项目的尺寸。
- NSCollectionLayoutGroup：把若干个 item 横向或竖向排在一起。group 的 subitems 中也可以放另一个 group。
- NSCollectionLayoutSection：把同一个 group 重复排列，直到用完分区中的全部项目，并设置分区边距和 group 之间的距离。

UICollectionViewCompositionalLayout 再用 section 生成整个布局。UICollectionViewFlowLayout 按行自动换行，适合尺寸规则的网格或单行列表；UICollectionViewCompositionalLayout 由开发者逐层描述项目如何组合，适合同一分区内大卡片和小卡片混排。

尺寸用 NSCollectionLayoutDimension 描述，常用三种：

- .absolute(值)：固定点数，例如 .absolute(144) 表示 144pt。
- .fractionalWidth(比例)、.fractionalHeight(比例)：相对外层容器宽度或高度的比例，例如 .fractionalHeight(1) 表示与外层 group 等高。
- .estimated(值)：预估尺寸，实际尺寸由 Cell 的自动布局计算。

示例：每组左边一张大卡片，右边上下两张小卡片，整个列表横向滚动。

```swift
let large = NSCollectionLayoutItem(layoutSize: NSCollectionLayoutSize(
    widthDimension: .absolute(144),
    heightDimension: .fractionalHeight(1)))

let small = NSCollectionLayoutItem(layoutSize: NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1),
    heightDimension: .absolute(88)))

let smallColumn = NSCollectionLayoutGroup.vertical(
    layoutSize: NSCollectionLayoutSize(
        widthDimension: .absolute(198),
        heightDimension: .fractionalHeight(1)),
    subitems: [small, small])
smallColumn.interItemSpacing = .fixed(8)

let group = NSCollectionLayoutGroup.horizontal(
    layoutSize: NSCollectionLayoutSize(
        widthDimension: .absolute(350),
        heightDimension: .absolute(184)),
    subitems: [large, smallColumn])
group.interItemSpacing = .fixed(8)

let section = NSCollectionLayoutSection(group: group)
section.interGroupSpacing = 8
section.contentInsets = NSDirectionalEdgeInsets(top: 8, leading: 16, bottom: 8, trailing: 16)

let configuration = UICollectionViewCompositionalLayoutConfiguration()
configuration.scrollDirection = .horizontal
let layout = UICollectionViewCompositionalLayout(section: section, configuration: configuration)
```

分区中有 6 个项目时，排列结果如下：

1. 第一个 group 的左边缘在 x = 16。item 0 是大卡片，位于 x = 16，宽 144、高 184。
2. smallColumn 位于大卡片右侧 8pt 处，x = 16 + 144 + 8 = 168。item 1 和 item 2 是两张小卡片，上下排列：item 1 的 y 是 8，item 2 的 y 是 8 + 88 + 8 = 104。
3. 一个 group 放完 3 个项目后，section 重复同一个 group。第二个 group 的左边缘在 x = 16 + 350 + 8 = 374，依次放 item 3、item 4、item 5。

group 的宽度 350 等于大卡片 144、间距 8 和小卡片列 198 之和；高度 184 等于两张小卡片 88 × 2 加上间距 8。
