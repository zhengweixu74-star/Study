# Android 应用开发

[返回笔记目录](../README.md)

| 文档 | 内容范围 |
| --- | --- |
| [Android 系统分层与应用启动](system-architecture-and-app-launch.md) | 系统分层、ART 与 DEX、字节码与机器码、NDK 与 .so、AOT 与 JIT、ART 的堆与 GC 触发时机、ART／Dalvik／JVM、进程与 UID 沙盒、Zygote 与 system_server、Binder、冷启动流程、Application 与系统创建 Activity |
| [Android 工程结构、依赖库与安装包](project-structure-and-build.md) | 工程目录与 iOS 对照、API level 与 minSdk、AndroidX 与 Jetpack、构建变体、APK 内容、AAB、R8 压缩与混淆 |
| [Android 资源与 R 类](resources-and-r-class.md) | 资源 ID 与命名规则、XML 与代码引用、findViewById 的查找范围、资源限定符、多语言、多模块中的 R 类 |
| [Android 应用组件、Context 与权限](app-components-and-context.md) | 四种应用组件、后台的两种含义、Service 的线程与启动方式、进程优先级与后台限制、前台 Service 与常驻通知、前台 Service 类型、WorkManager、BroadcastReceiver、ContentProvider、Context 与 Application、普通权限与运行时权限 |
| [Activity、Fragment 与页面组织](activity-fragment-and-navigation.md) | Activity 与 Fragment 的职责、宿主和窗口、Activity 与 Window、DecorView 的持有关系、窗口的显示／输入／窗口规则、页面组合、创建 Fragment 并放进 Activity、清单声明、启动流程、显式与隐式 Intent、页面参数与 Parcelable、任务栈与启动模式、外部入口打开内页后的返回目标、页面用 Activity 还是 Fragment、UIViewController 对照 |
| [Android 页面生命周期与状态保存](activity-lifecycle-and-state.md) | Activity 的基本回调、配置变化与重建、Jetpack ViewModel、进程结束后的状态恢复、SavedStateHandle、Fragment 实例与视图的生命周期 |
| [Android 主线程、Handler 与 ANR](main-thread-and-handler.md) | 主线程规则、Looper、MessageQueue 与 Handler、用 Handler 回到主线程、Dispatchers.Main、消息队列为空时的休眠、ANR 与阈值 |
| [ViewModel 状态暴露与页面收集：StateFlow 与 repeatOnLifecycle](stateflow-and-lifecycle-collection.md) | StateFlow 与 MutableStateFlow、私有可变与公开只读状态、viewModelScope 与 Dispatchers.IO、lifecycleScope 与 repeatOnLifecycle、页面不可见时停止收集、LiveData 与 StateFlow |
| [Android View、ViewGroup 与 XML 布局](view-system-and-layout.md) | View 与 ViewGroup 的职责、容器嵌套、常见布局容器及与 UIKit 的对照、XML 布局、Jetpack Compose、px、dp 与 sp、布局预览的示例数据 |
| [Android View 绘制流程与屏幕刷新](view-drawing-and-frame-refresh.md) | 测量、布局与绘制、MeasureSpec、requestLayout 与 invalidate、硬件加速与 RenderNode、属性修改的开销、滚动偏移与 scrollTo／scrollBy、VSYNC 与 Choreographer、掉帧 |
| [Android 触摸事件与 View 事件处理](touch-events-and-view-dispatch.md) | MotionEvent 与触摸序列、ACTION_DOWN、监听器返回值、OnTouchListener 与 onTouchEvent 的调用关系、取消处理、ViewGroup 拦截事件、一次触摸的完整分发路径、onInterceptTouchEvent 与 onTouchEvent 的分工、UIKit 对照 |
| [Android 列表复用：ListView、RecyclerView 与 ViewHolder](listview-recyclerview-and-viewholder.md) | ListView 与 RecyclerView、Adapter 与 UITableViewDataSource 对照、ViewHolder、创建与绑定流程、局部更新、ListAdapter、提交数据与行 View 创建的时机、RecyclerView 滚动的回收／填充／平移、scrollToPositionWithOffset |
| [ValueAnimator、插值器与滚动动画](valueanimator-and-interpolation.md) | 每帧执行顺序、插值器与动画进度、动画值、TypeEvaluator 与颜色渐变、滚动距离与取消行为、ValueAnimator／ObjectAnimator／ViewPropertyAnimator 的区别与性能 |
| [Android 内存泄漏与 OOM](memory-leaks-and-oom.md) | 可达引用链与泄漏、常见泄漏场景与修正、LeakCanary、OOM 与 Java 堆、Bitmap 内存计算 |
