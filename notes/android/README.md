# Android 界面与动画

[返回笔记目录](../README.md)

| 文档 | 内容范围 |
| --- | --- |
| [Android View、ViewGroup 与 XML 布局](view-system-and-layout.md) | View 与 ViewGroup 的职责、容器嵌套、常见布局容器及与 UIKit 的对照、XML 布局与布局预览的示例数据 |
| [Activity、Fragment 与页面组织](activity-fragment-and-navigation.md) | Activity 与 Fragment 的职责、宿主和窗口、Activity 与 Window、DecorView 的持有关系、页面组合、清单声明、启动流程及 UIViewController 对照 |
| [Android 页面生命周期与状态保存](activity-lifecycle-and-state.md) | Activity 的基本回调、配置变化与重建、Jetpack ViewModel、进程结束后的状态恢复、SavedStateHandle、Fragment 实例与视图的生命周期 |
| [ViewModel 状态暴露与页面收集：StateFlow 与 repeatOnLifecycle](stateflow-and-lifecycle-collection.md) | StateFlow 与 MutableStateFlow、私有可变与公开只读状态、viewModelScope 与 Dispatchers.IO、lifecycleScope 与 repeatOnLifecycle、页面不可见时停止收集 |
| [Android 列表复用：ListView、RecyclerView 与 ViewHolder](listview-recyclerview-and-viewholder.md) | ListView 与 RecyclerView、Adapter 与 UITableViewDataSource 对照、ViewHolder、创建与绑定流程、局部更新、ListAdapter |
| [Android 触摸事件与 View 事件处理](touch-events-and-view-dispatch.md) | MotionEvent 与触摸序列、ACTION_DOWN、监听器返回值、OnTouchListener 与 onTouchEvent 的调用关系、取消处理与 UIKit 对照 |
| [ValueAnimator、插值器与滚动动画](valueanimator-and-interpolation.md) | 每帧执行顺序、插值器与动画进度、动画值、滚动距离与取消行为 |
| [Android 资源与 R 类](resources-and-r-class.md) | 资源 ID 与命名规则、XML 与代码引用、findViewById 的查找范围、资源限定符、多语言、多模块中的 R 类 |
