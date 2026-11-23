# Swift / Objective-C 运行时与方法派发

[返回笔记目录](../README.md)

## 本文目录

- [Objective-C 分类与类扩展](#objective-c-分类与类扩展)
- [Swift Runtime 与 Objective-C Runtime](#swift-runtime-与-objective-c-runtime)
- [自动释放池与 AutoreleasePoolPage](#自动释放池与-autoreleasepoolpage)
  - [线程、Page 与池边界](#线程page-与池边界)
  - [嵌套池的进入与退出](#嵌套池的进入与退出)
  - [Swift 使用场景与按批释放](#swift-使用场景与按批释放)
- [Objective-C +load：调用时机与自动注册](#objective-c-load调用时机与自动注册)
  - [示例：图片解析组件自动注册](#示例图片解析组件自动注册)
  - [与 App 启动耗时的关系](#与-app-启动耗时的关系)
  - [+initialize 的触发时机与执行限制](#initialize-的触发时机与执行限制)
  - [按功能需要初始化](#按功能需要初始化)
- [关联对象、稳定 key 与持有关系](#关联对象稳定-key-与持有关系)
  - [关联闭包与循环引用](#关联闭包与循环引用)
  - [通过包装对象保存弱引用](#通过包装对象保存弱引用)
- [实例布局、静态存储与类型元数据](#实例布局静态存储与类型元数据)
- [对象引用与 UnsafePointer](#对象引用与-unsafepointer)
- [isa 与类型元数据指针](#isa-与类型元数据指针)
- [静态派发、动态派发与虚方法表](#静态派发动态派发与虚方法表)
- [编译阶段、机器码与内联](#编译阶段机器码与内联)
  - [Swift 编译优化选项与体积](#swift-编译优化选项与体积)
- [协议见证表与遵循关系](#协议见证表与遵循关系)
  - [扩展方法与协议要求的派发](#扩展方法与协议要求的派发)
- [Objective-C 互操作、Selector 与 IMP](#objective-c-互操作selector-与-imp)
  - [消息转发的接收者与执行过程](#消息转发的接收者与执行过程)
- [Method Swizzling 与使用场景](#method-swizzling-与使用场景)
  - [适用场景与直接修改方法的区别](#适用场景与直接修改方法的区别)
- [Selector 字符串与方法映射](#selector-字符串与方法映射)
- [@objcMembers 与 Objective-C 兼容限制](#objcmembers-与-objective-c-兼容限制)
- [KVC 按键访问属性](#kvc-按键访问属性)
- [KVO 观察关系与属性访问](#kvo-观察关系与属性访问)
- [自动 KVO 的动态子类机制](#自动-kvo-的动态子类机制)
- [KVO 通知选项与自动通知](#kvo-通知选项与自动通知)
- [dynamic 对 KVO 访问路径的影响](#dynamic-对-kvo-访问路径的影响)

## Objective-C 分类与类扩展

Category（分类）可以给已有类添加方法，也可以把一个类的实现拆到多个文件中，但不能添加实例变量。在 Category 中声明属性时，编译器不会生成对应的实例变量和 getter／setter，需要自己实现访问方法。

Class Extension（类扩展）与原类一起编译，通常写在 .m 文件中，用来声明只在类内部使用的方法和属性。Class Extension 中声明的属性会生成实例变量和 getter／setter。“只在内部使用”只是编译期的可见性：其他代码拿不到声明，但运行时仍然可以通过发送消息调用这些方法。

## Swift Runtime 与 Objective-C Runtime

“运行时”有两种含义：作为时间阶段，指程序正在执行的阶段；作为组件（Runtime），指程序运行期间提供支持的库。Swift Runtime 为运行中的 Swift 程序提供类型元数据管理、动态类型转换、泛型支持和内存管理等能力；也有一部分工作由编译器生成的代码直接完成，不经过 Runtime 函数。

Swift Runtime 和 Objective-C Runtime 是两个不同的库，在 Apple 平台上可以互相配合。Objective-C Runtime 支撑 OC 的对象系统，可以在运行时查询、添加、替换方法，并管理关联对象。Swift 代码调用 `@objc dynamic` 方法，或 Objective-C 代码调用暴露给 OC 的方法时，才经过 Objective-C Runtime 的消息派发；其他 Swift 方法调用不经过它。动态派发是在运行时从已经编译好的多个实现中选择一个，不会重新编译代码。

Objective-C 的动态性，是程序可以在运行时识别对象的实际类、选择方法实现，并查询或修改类的信息。常见表现包括：

- 动态类型：Objective-C 的 id 类型变量可以保存任意类的对象引用，对象在运行时仍有确定的类。Swift 的 AnyObject 作用类似，但编译器对 AnyObject 的检查更严格。
- 动态绑定：Objective-C 消息调用在运行时根据接收者的实际类选择方法实现，具体过程见 [Objective-C 互操作、Selector 与 IMP](#objective-c-互操作selector-与-imp)。
- 动态加载与类注册：程序可以在运行时加载包含类的已编译代码，也可以通过 Runtime API 创建并注册新类。两者都只使用已经编译好的代码，不会在运行时编译源码。

## 自动释放池与 AutoreleasePoolPage

retain 让对象的引用计数加 1；release 让引用计数减 1，减到 0 时对象销毁；autorelease 不立即减计数，而是把对象登记到当前的自动释放池，等池结束时再对对象执行一次 release。登记本身不改变引用计数。同一个对象登记几次，池结束时就 release 几次。

自动释放池只处理登记过的对象，不会扫描代码块里的变量。只有框架代码或编译器生成的代码对对象调用 autorelease 时，运行时才把对象地址写入池。这次调用可以发生在 autoreleasepool 闭包内任意深度的同步函数调用中。下面只讨论对象实际被登记到池中的情况，不展开编译器的返回值优化。

### 线程、Page 与池边界

AutoreleasePoolPage 是运行时用来存放登记记录的内存块，记录内容是对象地址和池边界标记，对象本身不会被移动到 Page 中。每个 Page 的 next 指针指向下一个可写入的位置；parent 和 child 指针把多个 Page 连成链表；hotPage 是当前正在写入的 Page。当前 Page 写满时，运行时切换到下一个 Page，没有就新分配一个。

线程局部存储（TLS）是让每个线程各自保存一份数据的机制。运行时用 TLS 为每个线程保存各自的 hotPage，所以每个线程有自己的自动释放池。对象登记到哪个线程的池，取决于调用 autorelease 时代码在哪个线程执行，与对象在哪个线程创建无关。一个线程清空自己的池，不影响其他线程的池。

池边界是标记一个自动释放池起点的特殊记录。一个 Page 中可以有多个池的记录，一个池的记录也可以跨越多个 Page。parent 和 child 只表示 Page 的先后存储顺序，与池的嵌套无关；Page 写满也不会产生新的池。池结束后，运行时可能保留空 Page 供之后复用，不一定立即释放。

### 嵌套池的进入与退出

在同一线程上，一个池还没结束时又进入另一个池，就形成外层池和内层池。两层池使用同一套记录，只是边界标记的位置不同。假设依次发生以下操作：

```text
进入外层池 → 登记 A
进入内层池 → 登记 B → 登记 C
退出内层池
登记 D → 退出外层池
```

登记 C 后，记录依次是“外层边界、A、内层边界、B、C”，可能在同一个 Page 中，也可能跨越多个 Page。退出内层池时，运行时从最新的记录往前，依次对 C、B 执行 release，直到遇到内层边界，并移除这个边界；外层边界和 A 保持不变。随后登记 D；退出外层池时，再依次对 D、A 执行 release。这里假设 release 过程中没有登记新的对象。

池清空只是对登记的对象各执行一次 release。如果对象还有其他强引用，引用计数不会归零，对象继续存活。ARC 的基本规则见 [ARC 与 Android GC](swift-basics.md#arc-与-android-gc引用计数可达性与引用环)。

### Swift 使用场景与按批释放

纯 Swift 类实例由 ARC 直接管理，不经过自动释放池。但 Swift 调用 Foundation、UIKit 等 Objective-C 框架时，框架内部仍会产生登记到自动释放池的临时对象。UIKit 和 AppKit 已经在主线程每次事件循环的前后创建和清空自动释放池；在一次事件处理中执行大量同步工作时，可以再加一层范围较小的池，让临时对象尽早释放。

下面的示例同步读取 1,000 个图片路径，每批处理 100 张，前提是 paths.count 等于 1,000。每批的池结束时，只释放这一批登记过的对象，UIImage 是否在此时销毁还要看是否有其他强引用。

```swift
import UIKit

func inspectImages(at paths: [String]) {
    precondition(paths.count == 1_000)

    autoreleasepool {
        for batch in 0..<10 {
            autoreleasepool {
                for offset in 0..<100 {
                    let index = batch * 100 + offset
                    if let image = UIImage(contentsOfFile: paths[index]) {
                        print(image.size)
                    }
                }
            }
        }
    }
}
```

执行顺序是：处理第 1～100 张 → 清空本批的池 → 处理第 101～200 张 → 清空 → …… 直到第 10 批。10 个内层池先后执行，都嵌套在同一个外层池中。也可以每张图片用一个池，100 只是选定的批次大小。

使用时注意以下几点：

- 只多写一层循环不会产生自动释放池，必须用 autoreleasepool 包住代码。
- 池中没有登记对象时，池结束时什么也不做。
- 数组、缓存等对象仍持有图片时，清空池不会让图片释放。
- 把启动异步任务的代码包进池，任务之后在其他线程登记的对象不属于这个池。

自动释放池用于尽早释放对象；[UITableView 重用机制](../ios/uikit-layout-and-mvvm.md#uitableview-的重用机制)则保留视图对象供之后重复使用，两者目的相反。

## Objective-C +load：调用时机与自动注册

+load 是 Objective-C 的类方法。类或分类被加载到 Objective-C Runtime 时，Runtime 自动调用这个类或分类自己实现的 +load。开发者可以在 +load 中编写加载时就要执行的准备代码，不需要创建对象，也不需要在业务代码中调用。

随 App 启动加载的类和分类，+load 在 main 函数之前调用。程序启动后才动态加载的库，库中的 +load 在加载这个库时调用，此时 main 早已执行。

### 示例：图片解析组件自动注册

假设开发者编写了 WebPDecoder，负责把 WebP 图片数据解码为可显示的图像。图片管理器保存一张“图片格式 → 解析器类”的映射表。注册是把解析器类及其支持的格式加入这张表，供图片管理器后续查找。

1. 开发者在 WebPDecoder 的 +load 中写入注册代码：向图片管理器登记“WebP 格式由 WebPDecoder 解析”。图片管理器及其注册接口需要支持在这个加载阶段使用。
2. WebPDecoder 被加载到 Objective-C Runtime。Runtime 自动调用 WebPDecoder 的 +load。注册代码随之执行，映射表中增加“WebP → WebPDecoder”这一项。这个阶段只登记解析能力，还没有解码具体图片。
3. 业务之后请求加载一张 WebP 图片。图片管理器识别图片格式，查询映射表，找到 WebPDecoder，再调用 WebPDecoder 的解码逻辑。

**+load 在这个例子中的作用，是提供自动执行注册代码的时机。真正完成注册的是开发者写在 +load 中的代码。**

只定义 WebPDecoder 类，图片管理器并不知道“WebP → WebPDecoder”的对应关系；+load 中没有注册代码，也不会完成注册。如果把注册代码放到其他初始化入口，就要保证这个入口在第一次查找解析器之前执行。

### 与 App 启动耗时的关系

启动阶段执行的 `+load` 会占用启动时间。`+load` 中应只放确实需要在加载阶段完成的少量工作，避免同步读取大文件、同步等待网络结果或执行大量计算。优化看的是首屏显示前要执行的工作量，而不是 `+load` 方法的个数。

### +initialize 的触发时机与执行限制

`+initialize` 是 Objective-C 的类初始化方法。一个类第一次收到 Objective-C 消息（即第一次被调用方法）之前，运行时先调用这个类的 `+initialize`；如果是子类，父类先初始化。开发者一般不直接调用 `+initialize`。`+initialize` 每个类执行一次，与每个对象各执行一次的实例初始化方法 `-init` 不同。

例如，第一次调用 `[ExportManager version]` 时，运行时先初始化 `ExportManager`，再执行 `version`。如果 `+initialize` 中读取导出配置，那么只是查询版本号，也会触发这次读取。也就是说，“第一次使用这个类”可能早于“第一次使用导出功能”。

`+initialize` 在第一次调用这个类的线程上同步执行，原来的调用要等 `+initialize` 返回才继续。其他线程在此期间调用这个类，也要等待初始化完成。所以 `+initialize` 中只适合做少量、简单的初始化，不应把 `+load` 中的耗时工作原样搬进来。

`+initialize` 的执行时间可能很早。如果其他类的 `+load` 调用了这个类，`+initialize` 会在 `main` 之前执行。只有类一直没有被调用，`+initialize` 才一直不执行。

运行时为每个类单独记录是否已初始化。子类没有实现 `+initialize` 时，子类初始化会调用继承来的父类实现，所以父类的 `+initialize` 方法体可能执行多次，每次的接收类不同。如果某项准备工作只属于父类，要在方法中判断 `self == [Parent class]`，只在接收类是父类时执行。

### 按功能需要初始化

按需初始化是开发者在具体功能第一次需要资源时，再准备该资源。假设 `ExportManager` 导出文件前需要读取配置，读取耗时 50 毫秒，而首页不需要导出功能。50 毫秒只是示例值。

| 初始化位置 | 执行过程 | 对用户等待的影响 |
| --- | --- | --- |
| 启动时加载类的 `+load` | 启动 → 读取导出配置 → 进入 `main` → 显示首页 | 不导出文件也会增加启动等待 |
| `+initialize` | 首次调用该类方法 → 读取配置 → 执行原方法 | 首次使用类足够晚时可推迟开销；首页查询版本也可能提前触发 |
| 导出功能入口 | 显示首页 → 用户点击导出 → 首次准备配置 → 导出 | 首页不承担配置读取，但首次导出仍需要准备资源 |

只需要准备一次的配置，第一次准备成功后要保存结果，之后直接使用。推迟准备不会改变执行线程：如果配置仍在主线程同步读取，用户第一次点击导出时仍要等待。是否改为异步准备，取决于组件对线程的要求，以及导出是否必须等配置就绪。

注册类工作有先后要求，不能随意推迟。前面的图片解析器必须在第一次查找解析器之前完成注册，如果把注册改到“解析器第一次被使用时”，查找时就找不到解析器。

判断优化是否有效，要看这项工作是否真的移出了首屏显示前的执行过程。启动阶段与测量边界见[iOS 应用生命周期、启动与窗口](../ios/app-lifecycle-and-events.md#启动优化的三个方向)。

## 关联对象、稳定 key 与持有关系

class、struct、enum 都支持 extension，但扩展不能添加实例存储属性。如果要给已有的 UIView 附加一个 businessID，可以在扩展中声明计算属性，在 getter／setter 中调用关联对象 API。关联对象 API 由 Objective-C Runtime 提供，Runtime 把附加值保存在对象之外的一张表中，UIView 实例本身的内存布局不变。

```text
view.businessID = 新值 → 计算属性 setter → objc_setAssociatedObject
读取 view.businessID   → 计算属性 getter → objc_getAssociatedObject
查找关系：宿主对象身份 + key → 附加值
```

**key 是一个指针。** 关联对象用 key 的地址区分不同的附加值，所以通常使用地址固定不变的文件级变量：

```swift
// 类型和函数之外；private 在此限制在当前文件
private var businessIDKey: UInt8 = 0
// 读写都使用同一个 &businessIDKey
```

起作用的是变量的地址，变量的值 0 没有意义。同一个 key 可以用于不同的 view；同一个 view 上的不同附加属性要用不同的 key。不能用临时局部变量的地址作 key，因为每次调用时局部变量的地址可能不同。在 Swift 6 中，这个全局 var 还需要满足并发检查，例如标记为 `nonisolated(unsafe)` 或放在 `@MainActor` 隔离范围内。

**关联策略决定宿主怎样持有附加值。** 宿主是附加值所挂靠的对象。retain 让宿主强持有附加对象；copy 先复制附加对象，再强持有复制结果，不可变对象的 copy 可能直接返回原对象。assign 不持有附加对象，附加对象销毁后也不会自动置 nil，读取时可能访问已释放的内存，所以不能用 assign 代替 weak。NONATOMIC 只表示这次关联读写不加锁，与业务逻辑是否线程安全无关。

持有方向是“宿主 → 附加对象”。只有附加对象（例如闭包）又强持有宿主时，才会形成引用环。关联被移除、被替换或宿主销毁时，宿主对附加对象的持有随之解除；附加对象是否释放，还要看是否有其他强引用。

```swift
objc_setAssociatedObject(
    self,                              // 宿主
    &businessIDKey,                     // 稳定 key
    newValue,                          // 附加值
    .OBJC_ASSOCIATION_RETAIN_NONATOMIC  // 保存策略
)
```

newValue 是 setter 的参数名。附加值是 class 实例时，传入的就是对象引用；附加值是 String 等值类型时，Swift 把值桥接为 NSString 等 Objective-C 对象再传入，retain 策略持有的是桥接得到的对象。传入 nil 会移除这个 key 对应的关联。

**值类型可以作附加值，但不能作宿主。** 关联对象靠宿主的对象身份查找附加值。class 实例有固定的身份；struct、enum 在赋值和传参时会复制，没有固定身份。如果把值类型作为第一个参数传入，Swift 每次都会把它临时包装成一个新对象：设置时用的是对象 A，读取时用的是对象 B，于是找不回附加值。作为第三个参数（附加值）时，包装得到的对象被宿主持有，所以没有这个问题。

### 关联闭包与循环引用

下面的示例在 `@MainActor` 类中用一个 static 属性作 key，static 属性的地址固定不变。`@convention(block)` 把回调声明为 Objective-C 的 Block 类型，Block 是支持 copy 的 Objective-C 对象，因此可以使用 COPY 策略关联。

```swift
import Foundation
import ObjectiveC

@MainActor
final class Example: NSObject {
    private static var callbackKey: UInt8 = 0

    func installCallback() {
        let callback: @convention(block) () -> Void = {
            self.doSomething()
        }
        objc_setAssociatedObject(
            self, &Self.callbackKey, callback,
            .OBJC_ASSOCIATION_COPY_NONATOMIC
        )
    }

    func invokeCallback() {
        let callback = objc_getAssociatedObject(
            self, &Self.callbackKey
        ) as? (@convention(block) () -> Void)
        callback?()
    }

    func clearCallback() {
        objc_setAssociatedObject(
            self, &Self.callbackKey, nil,
            .OBJC_ASSOCIATION_COPY_NONATOMIC
        )
    }

    private func doSomething() {
        print("执行回调")
    }
}
```

调用 `example.installCallback()` 时，宿主是 `self` 指向的 `Example` 实例，不是 `Example` 类。key 是 `callbackKey` 的地址，关联值是 `callback` 的复制结果。`invokeCallback()` 使用同一宿主和同一 key 取回回调，再调用回调。

这个示例存在强引用环：`Example 实例 → 关联回调 → 同一个 Example 实例`，因为闭包中的 `self.doSomething()` 强捕获了 `self`。把 COPY 换成 RETAIN 也无法消除这个环：复制 Block 时，Block 捕获的 `self` 仍是强引用。

如果回调不应延长宿主生命周期，创建回调时使用下面的替代写法：

```swift
let callback: @convention(block) () -> Void = { [weak self] in
    self?.doSomething()
}
```

`[weak self]` 不让宿主保持存活，宿主销毁后闭包中的 `self` 为 `nil`。另一种做法是在业务结束时，由外部代码调用 `clearCallback()` 解除关联。不能把解除关联的代码只放在 `deinit` 中：强引用环存在时，实例的引用计数不会归零，`deinit` 永远不会执行。Swift 闭包与捕获环境的区别见[捕获存储与函数返回](swift-basics.md#捕获存储与函数返回)。

### 通过包装对象保存弱引用

关联策略中没有 weak 选项。如果需要关联一个弱引用，可以让宿主强持有一个包装对象，再由包装对象弱引用目标对象：

```swift
final class WeakBox<T: AnyObject> {
    weak var value: T?

    init(_ value: T) {
        self.value = value
    }
}
```

把 `WeakBox(target)` 以 retain 策略设为关联值后，关系是 `宿主 → WeakBox ⇢ target`，其中最后一条是弱引用。取回包装对象后读取 `value`。目标对象没有其他强引用而销毁后，`value` 自动变为 `nil`。包装对象本身由宿主强持有，所以不会被提前释放。

## 实例布局、静态存储与类型元数据

实例布局是一个实例在内存中的组织方式：有哪些字段，每个字段占多少空间、从哪个偏移开始，以及为了对齐而插入的填充。class 实例除了存储属性，还有一个运行时使用的对象头；具体布局随平台和实现不同而不同。

纯 Swift 类实例的布局可以简化为：

```text
对象头：类型元数据指针、引用计数等管理信息
业务字段：age、name 等实例存储
必要的对齐／填充
```

对象头中只有一个指向类元数据的指针，类型信息和方法代码不在每个实例中重复保存，所有实例通过这个指针找到同一份类元数据。OC 对象的对象头是 isa，编码方式与 Swift 对象头不同。

关联对象保存在实例之外，计算属性只是访问代码，static 存储属性不属于任何实例；这三种方式都不需要给实例增加字段。

**static 的存储：**

普通非泛型类型的 static 存储属性，由编译器分配在一块单独的全局存储中，通过编译器生成的访问代码读写。static 存储属性属于类型本身，但不保存在类元数据中。

```text
person1 实例：自己的 age
person2 实例：自己的 age

Person.adultAge：独立的类型属性存储或优化后的常量表示
Person 类元数据：描述这个类型的信息
```

这块全局存储位于程序映像的数据区；需要运行代码计算初值时，在第一次访问时初始化。像 `static let adultAge = 18` 这样的常量，编译器还可能直接把 18 写进使用它的代码，不再占用单独的内存。所以要区分三样东西：static 属性的全局存储、类型元数据、每个实例自己的字段。

**类型元数据与方法：**

类型元数据是 Runtime 使用的类型描述信息，包括类型的种类、大小和布局、父类、泛型参数、方法入口等，用于类型转换、泛型代码执行和反射。不同种类的类型（class、struct、enum 等），元数据的结构不同。

方法的实现是编译后的机器码，位于程序的代码区。类元数据中保存的是 Swift 虚方法表中的函数地址，或者指向 OC 方法列表的信息，机器码本身不在元数据中。只有可以被子类重写的方法才会进入虚方法表。

```text
对象 → 类元数据 → 某个方法的实现入口 → 执行代码
```

类型元数据是数据本身，类型元数据指针是存放这份数据地址的值，两者是不同的东西。

## 对象引用与 UnsafePointer

class 引用在底层保存的是对象地址，可以理解为“指向对象的值”。但 Swift 的对象引用受类型系统和 ARC 管理：编译器知道它指向什么类型，ARC 管理它的引用计数，代码也不能对它做地址运算。这些都是普通裸指针没有的。

```swift
let person: Person = Person()
```

`: Person` 告诉编译器 person 可以怎样使用。person 在运行时只保存一个地址，类型信息记录在编译器中，编译器据此检查代码可以访问哪些成员；对象头中的类元数据指针则让程序在运行时识别对象的实际类型，并完成动态派发。

`UnsafePointer<T>` 是 Swift 中只能读取的类型化指针，通过 pointee 读取指向的 T 值；`UnsafeMutablePointer<T>` 还可以写入。这两种指针虽然带类型参数，但不保证地址有效、内存已初始化或访问不越界，保存指针也不会延长指向的对象或内存的生命周期，这些都由调用者自己保证。通过 withUnsafePointer 得到的指针只能在闭包内使用。

## isa 与类型元数据指针

isa 是对象头中的一个指针，对象通过 isa 找到自己所属的类。isa 只用于查找类，不决定方法调用采用哪种派发方式。

纯 Swift 类实例即使没有继承 NSObject，对象头中也有指向类元数据的指针。在 Apple 平台上，Swift 类元数据的开头部分与 OC 的 Class 对象结构相同，所以这个指针同时就是 isa：

```text
同一个 Swift 对象
       │
       └─ 对象头中的类型入口
                    │
                    ▼
             Swift 类元数据
             同时兼容 OC Class
```

也就是说，Swift 对象一创建就带着可以当作 isa 使用的指针，不是传给 OC 时才补上；isa 直接指向类元数据，中间没有第二层指针。

以上只适用于 Apple 平台。不支持 OC 互操作的平台（如 Linux）上，Swift 对象仍有元数据指针，但不需要兼容 isa。反过来，纯 OC 类的 Class 对象也不是完整的 Swift 类元数据。另外，OC 对象的 isa 中还编码了引用计数等其他信息，需要按规则解码才能得到类的地址。

每个对象都有这个指针，与属性是否标记 dynamic 无关。有这个指针，也不表示 Swift 方法调用都走 OC 消息派发，或 Swift 成员都自动暴露为 selector、支持 KVO。

## 静态派发、动态派发与虚方法表

动态派发是在运行时根据对象的实际类型，选择这次调用执行哪个方法实现。

```swift
class Animal {
    func speak() { print("声音") }
}

class Dog: Animal {
    override func speak() { print("汪") }
}

func makeSound(_ animal: Animal) {
    animal.speak()
}
```

参数的声明类型是 Animal，传入的实际对象可以是 Dog，这时必须调用 Dog 重写的实现，这就是多态。如果编译器在某个调用位置能确定对象一定是 Dog，可以把这次调用优化成直接调用 Dog.speak，甚至内联；优化只改变调用方式，执行结果不变。

**静态派发：**

静态派发是编译时就确定调用哪个实现，生成的代码直接调用这个实现，运行时不再根据对象类型选择。编译时确定的只是“调用哪个实现”，方法仍在运行时执行，参数和返回值也是运行时才得到。

通过具体类型调用 struct 的方法、调用 final 方法时，通常是静态派发。但同一个 struct 方法，如果通过协议类型或泛型调用，就可能经过见证表动态派发。

没有写 final 的类方法，如果编译器能确定某个调用位置只可能调用一个实现，也可以优化为直接调用，这叫去虚拟化。另外，`static` 关键字声明的是类型成员，与“静态派发”是两回事。

**虚方法表：**

Swift 类的虚方法表是一张按固定位置（槽位）保存方法实现地址的表，每个可重写的方法占一个槽位。父类和子类的同一个槽位可以指向不同的实现：

```text
Animal 的表：speak 槽位 → Animal.speak
Dog 的表：   speak 槽位 → Dog.speak
```

动态派发时，程序通过对象的实际类型找到对应的表，从 speak 槽位取出实现地址，再调用这个地址。表中保存的是地址，不是方法代码。编译器能确定调用目标时，可以省掉查表，直接调用。Swift 的虚方法表派发与 OC 按 selector 查找方法是两套不同的机制。

## 编译阶段、机器码与内联

Swift 编译大致经过源码解析、类型检查、SIL 等中间表示、优化和目标代码生成，最后通过链接形成程序。机器码是 CPU 能执行的二进制指令，与 CPU 架构有关；十六进制和汇编是帮助人阅读的表示。

例如 ARM64 的 `ret`（函数返回）指令编码是 `0xD65F03C0`，按小端顺序存入内存后，四个字节依次是 `C0 03 5F D6`。其他 CPU 架构的返回指令编码不同。

内联是编译器把被调用函数的代码直接复制到调用位置，省掉一次函数调用的开销，也让常量折叠等进一步优化成为可能。内联发生在编译器生成的代码中，源文件不变；内联过多会使代码体积增大。静态派发的调用更容易被内联，但编译器不一定会内联。

### Swift 编译优化选项与体积

Swift 的优化选项决定编译器如何生成代码。常用选项的目标如下。

| 选项 | 主要目标 |
| --- | --- |
| `-Onone` | 不执行常规优化，便于设置断点和查看变量。 |
| `-O` | 优先优化运行性能。 |
| `-Osize` | 优先减少生成代码的大小，同时保留优化能力。 |

`-Osize` 会更谨慎地选择内联。较大的函数被多个调用位置内联时，生成代码可能重复，导致体积增加；保留独立函数并让多个位置调用，可能减少这类重复。

`-Osize` 不会禁止所有内联。小函数内联后，编译器可能消除调用开销，并把已知常量参与的计算提前算出，从而进一步减少代码。是否内联由编译器根据具体代码决定。

`-Osize` 只影响编译生成的代码大小，与图片、视频等资源无关。实际能减少多少体积，取决于代码结构和编译器版本，运行性能也可能随之下降。比较效果时，保持其他构建条件不变，同时检查可执行文件大小和关键功能的性能。

## 协议见证表与遵循关系

协议见证表对应的是一条遵循关系，即“某个类型遵循某个协议”，而不是对应某个类型或某个协议。见证表记录在这条遵循关系中，每项协议要求由哪个实现满足。

```swift
protocol Speaker {
    func speak()
}

struct Dog: Speaker {
    func speak() { print("汪") }
}

struct Cat: Speaker {
    func speak() { print("喵") }
}
```

```text
Dog 遵循 Speaker：speak 要求 → 对应 Dog 实现的入口
Cat 遵循 Speaker：speak 要求 → 对应 Cat 实现的入口
```

同一个类型遵循两个协议，就有两张见证表；两个类型遵循同一个协议，也各有一张见证表。泛型和继承还会影响见证表的生成方式，二进制中实际有多少张表不能简单推算。

通过泛型或协议存在类型调用协议要求时，程序从见证表中找到实现地址再调用；表中的地址有时指向一段转接代码，由转接代码再调用实际实现。编译器对泛型函数做特化（为具体类型生成专用版本）后，可以省掉部分查表。协议要求与扩展新增方法的区别见下节。

### 扩展方法与协议要求的派发

派发也称分发，指一次方法调用如何确定要执行的实现。静态派发在编译时确定实现；动态派发在运行时通过方法表或运行时查找确定实现。具体机制与优化见[静态派发、动态派发与虚方法表](#静态派发动态派发与虚方法表)。

方法写在 `extension` 中，并不决定派发方式。派发方式取决于：扩展的是哪种类型、方法是否属于协议要求、方法有哪些修饰符，以及调用位置的静态类型。下表只讨论普通 Swift 调用。

| 方法与调用条件 | 派发规则 |
| --- | --- |
| `struct`、`enum` 或 `class` 扩展中的普通方法，通过具体类型直接调用，且不涉及 `@objc`、`dynamic` 或协议要求调用 | 静态派发；类扩展中的普通非 `@objc` 方法不能被子类重写 |
| 方法只在协议扩展中添加，不属于协议要求；通过协议类型或只以该协议为约束的泛型调用 | 编译器选择可见的协议扩展实现，不通过协议见证表选择具体类型的同名方法 |
| 方法属于协议要求；通过协议类型或受协议约束的泛型调用 | 可以经协议见证表动态派发；满足要求的实现可以写在具体类型、具体类型扩展或协议扩展中 |
| 类扩展中的方法标记为 `@objc dynamic`，运行环境支持 Objective-C 互操作 | 通过 Objective-C 运行时动态派发 |

**协议要求与扩展新增方法。** 协议要求是协议声明中规定的成员要求。下面的 `speak()` 属于协议要求，`describe()` 只由协议扩展添加，不属于协议要求。

```swift
protocol Speaker {
    func speak()
}

extension Speaker {
    func speak() {
        print("默认 speak")
    }

    func describe() {
        print("默认 describe")
    }
}

struct Person {}

extension Person: Speaker {
    func speak() {
        print("Person speak")
    }

    func describe() {
        print("Person describe")
    }
}

let person = Person()
let speaker: any Speaker = person

person.speak()      // Person speak
person.describe()   // Person describe
speaker.speak()     // Person speak
speaker.describe()  // 默认 describe
```

`person` 的编译时类型是 `Person`，编译器可以直接选择 `Person` 扩展中的两个方法。`speaker` 的类型是 `any Speaker`，表示可以保存任意遵循 `Speaker` 的值。`speaker.speak()` 是协议要求，程序通过 `Person: Speaker` 的见证表找到 `Person.speak()`。`speaker.describe()` 不是协议要求，编译器在编译时就选定 `Speaker` 扩展中的实现，运行时不会去查找 `Person.describe()`。

| `Person: Speaker` 中的方法要求 | 见证表中的概念性对应关系 |
| --- | --- |
| `speak()` | 有方法条目，调用最终进入 `Person.speak()` |
| `describe()` | 不是协议要求，没有用于派发 `describe()` 的方法条目 |

如果删除 `Person` 自己的 `speak()`，`Person: Speaker` 的见证表中仍有 `speak()` 条目，只是条目指向协议扩展中的默认实现。见证表中有没有某个方法的条目，只取决于协议是否声明了这项要求，与实现写在哪里无关。

同样的区别也适用于只以 `Speaker` 为约束的泛型函数：

```swift
func inspect<T: Speaker>(_ value: T) {
    value.speak()
    value.describe()
}

inspect(Person())
// 依次打印：Person speak、默认 describe
```

`inspect` 的函数体只知道 `T` 遵循 `Speaker`。`describe()` 在见证表中没有条目，所以 `inspect(Person())` 调用的是协议扩展中的 `describe()`。编译器即使优化掉查表，也不会改变这个结果。

同一个 `Person.speak()`，通过 `Person` 类型调用时是静态派发，通过 `any Speaker` 调用时经过见证表。判断派发方式要看每次调用的方式，不能只看方法写在哪里。

## Objective-C 互操作、Selector 与 IMP

`@objc` 把 Swift 声明暴露给 Objective-C，前提是声明中的类型都能在 OC 中表示。Selector 是 OC 方法的名称标识；`#selector(...)` 让编译器检查方法是否存在并得到这个标识，这一步不会执行方法。

UIKit 的 target-action API 保存目标对象和 selector，事件发生时通过 OC 消息派发调用目标方法。所以按钮的回调方法只需标记 @objc，不需要 dynamic。

@objc 只保证 OC 代码可以调用这个成员；Swift 代码调用这个成员时，仍可能使用 Swift 自己的派发方式。加上 dynamic 后，Swift 代码的调用也必须走 OC 消息派发。KVO 等需要在运行时替换方法实现的机制依赖这一点。

**selector 与方法实现：**

selector 只是方法的名称标识，不包含方法代码，也不是函数地址。IMP 是方法实现的函数地址。每个类在运行时保存一份方法列表，记录 selector 到 IMP 的对应关系；同一个 selector 在不同类中可以对应不同的 IMP。

```text
接收对象：person
方法标识：setAge:
参数：20
      ↓
isa 确定从哪个类开始找
      ↓
selector 确定找哪个方法
      ↓
找到对应 IMP，调用其实现
      ↓
setter 修改 age
```

“查找 setAge: → 执行 setter”是一次调用的两个阶段：先用 selector 找到 IMP，再执行 IMP 指向的 setter。

Objective-C 消息调用经过 objc_msgSend 等运行时函数。接收者不为 nil 时，Runtime 先查这个类的方法缓存；缓存未命中，再依次查当前类和父类的方法列表。找到 IMP 后，Runtime 调用这个实现，并传入接收者、selector 和参数，同时把结果写入缓存。有了缓存，大多数调用不需要遍历继承链。

当正常查找找不到实现时，系统还提供以下处理机会：

1. 动态方法解析：Runtime 调用类的 `+resolveInstanceMethod:`（类方法对应 `+resolveClassMethod:`），类可以在其中动态添加缺失的方法实现。
2. 消息转发：对象可以把消息交给另一个接收者；也可以提供方法签名，由 Runtime 把这次调用包装成 NSInvocation 对象交给对象处理。NSInvocation 保存接收者、selector 和参数等调用信息。
3. 以上步骤都没有处理这条消息时，Runtime 报告 unrecognized selector 并抛出异常。

消息转发只是把调用交给其他对象处理，与线程无关。这套机制只适用于 Objective-C 消息派发，普通 Swift 方法调用找不到实现时在编译阶段就会报错。

| 查找起点 | selector | 对应实现 |
|-|-|-|
| Person | setAge: | 原 setter |
| KVO 动态子类 | setAge: | 带通知逻辑的 setter |

KVO 改变的是查找起点（对象的 isa 指向动态子类），以及该类中 setAge: 对应的实现；selector 始终是同一个 setAge:，selector 本身没有任何变化。

暴露给 OC 的 name 属性，如果没有自定义访问器名称，getter 的 selector 是 `name`，setter 的 selector 是 **`setName:`**。末尾的冒号是名称的一部分，表示有一个参数（要设置的新值），所以 `setName` 与 `setName:` 是两个不同的 selector。

### 消息转发的接收者与执行过程

消息转发发生在方法查找和动态方法解析都没有找到实现之后，由接收者决定如何处理这条消息。快速转发直接把消息交给另一个对象；完整转发把调用包装成 `NSInvocation`，由接收者自行处理。

假设对象 A 没有实现 `speak`，A 的 `speaker` 属性引用对象 B，且 B 实现了 `speak`。A 可以提供下面的方法，把 `speak` 消息交给 B。

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(speak)) {
        return self.speaker;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

调用方给 A 发送 `speak` 消息。正常查找和动态方法解析没有提供实现后，运行时调用 A 的 `forwardingTargetForSelector:`，传入 `speak` 对应的 Selector。A 返回 B，运行时把这条消息发给 B，执行 B 的 `speak`。接收者由 A 自己指定，运行时不会在程序中搜索其他实现了 `speak` 的对象。

如果 `forwardingTargetForSelector:` 返回 nil，运行时进入完整转发：先调用 `methodSignatureForSelector:` 获取方法的参数和返回值类型，再把调用包装成 NSInvocation，传给 `forwardInvocation:` 处理。仍未处理时，抛出 unrecognized selector 异常。代理对象常用消息转发，把自己没有实现的方法交给实际工作的对象。

## Method Swizzling 与使用场景

Method Swizzling 是在运行时修改 selector 与 IMP 的对应关系，让同一个 selector 调用到另一个实现。`method_exchangeImplementations` 可以交换两个方法的实现。Selector 是调用时使用的方法标识，IMP 是实际实现的函数入口，定义见[Objective-C 互操作、Selector 与 IMP](#objective-c-互操作selector-与-imp)。

假设同一个类中已有 `speak` 和 `trackedSpeak` 两个方法，其参数和返回类型一致。交换前后可以表示为：

| Selector | 交换前的实现 | 交换后的实现 |
| --- | --- | --- |
| `speak` | 原始发声逻辑 | 记录调用并调用原始逻辑 |
| `trackedSpeak` | 记录调用并调用原始逻辑 | 原始发声逻辑 |

下面只展示交换完成后，包装方法中的逻辑，不包含执行交换的代码。

```objective-c
- (void)trackedSpeak {
    NSLog(@"记录一次 speak 调用");
    [self trackedSpeak]; // 交换后，这个 Selector 对应原始 speak 实现
}
```

交换完成后，调用方仍发送 `speak` 消息。运行时找到的是包装实现：先记录调用，再调用 `trackedSpeak`，而交换后 `trackedSpeak` 对应的是原始的 speak 实现，所以执行原始逻辑。如果在交换之前执行这个包装方法，`[self trackedSpeak]` 会调用自己，形成无限递归。

消息转发处理的是“接收者找不到实现”的消息，改变的是由谁接收；Swizzling 处理的是正常能找到实现的消息，改变的是 selector 对应哪个实现。

### 适用场景与直接修改方法的区别

如果原方法的源码由自己维护，直接在方法中加入记录逻辑就能完成需求；有共同父类时，也可以在父类中统一实现。只是为了加统计，不需要使用 Swizzling。

| 场景 | 需要补充的行为 | 采用 Swizzling 的原因 |
| --- | --- | --- |
| 通用页面统计组件 | 在页面出现等方法执行时记录事件 | 组件无法逐个修改接入方页面的源码，且相关调用经过目标方法 |
| 第三方集成组件 | 在应用回调发生时记录或转交信息 | 组件需要参与宿主已有回调，但不直接维护宿主实现 |
| 方法调用诊断 | 记录调用次数、参数或同步执行耗时 | 目标实现无法直接编辑，需要在相应运行时入口补充逻辑 |

两种方式都不用改调用方的代码。区别在于：直接修改改的是方法的源码，Swizzling 改的是运行时的 selector 与 IMP 对应关系。系统框架和已编译的库没有源码可改，这时才需要 Swizzling。

使用 Swizzling 要注意：

- 交换作用于整个类，这个类的所有实例都受影响。
- 子类重写了这个方法、且重写的实现不调用父类实现时，子类实例不受影响。
- 多个组件交换同一个方法，交换的先后顺序会影响最终的调用顺序。

所以使用前要确认目标类、目标方法、调用是否经过 OC 消息派发，以及在什么时候执行交换。

Swizzling 只影响 Objective-C 消息派发。Swift 的静态调用和虚方法表调用不经过 selector 查找，交换 OC 方法实现对这些调用不起作用；`@objc` 与 `dynamic` 的作用见[Objective-C 互操作、Selector 与 IMP](#objective-c-互操作selector-与-imp)。

## Selector 字符串与方法映射

`NSStringFromSelector` 返回 selector 对应的 Objective-C 方法名字符串，不执行方法。以前文的[ProductModule](../cross-platform/webview-and-lynx-bridge.md#lynx-原生模块注册与回调)为例：

```swift
let selector = #selector(ProductModule.getProductName(_:callback:))
let methodName = NSStringFromSelector(selector)
// methodName 为 "getProductName:callback:"
```

selector 名中的两个冒号对应两个参数。如果用 `@objc(...)` 自定义了暴露给 OC 的方法名，得到的就是自定义的名称。

Lynx 的 `methodLookup` 是 `[String: String]` 字典，键是 JS 方法名，值是 OC 方法名，所以需要把 selector 转成字符串，例如 `"getProductName" → "getProductName:callback:"`。用 `#selector` 生成字符串，编译器会检查方法是否存在、是否暴露给 OC；方法改名后，编译直接报错。手写字符串则要到运行时才会发现错误。

## @objcMembers 与 Objective-C 兼容限制

**`@objcMembers` 是标注在类上的 Swift 特性，编译器会给这个类中所有能用 Objective-C 表示的成员自动加上 `@objc`。** 类的扩展和子类中的成员也受影响。无法用 OC 表示的成员会被跳过，不会报错。`@objcMembers` 只负责暴露成员，不会把类注册为 Lynx 模块。

成员能否暴露给 OC，取决于属性类型、参数类型和返回类型能否用 OC 表示。常见情况：

| 类型或声明 | 能否用于暴露给 OC 的普通成员 |
|-|-|
| `Int`、`Bool`、`Double` 等常用标量 | 可以，对应 OC 可表示的基础类型 |
| `String`、`String?` | 可以，通过与 `NSString` 的桥接表示 |
| `NSObject`、`UIView` 等 OC 可见的类及其可选引用 | 可以 |
| 元素（以及字典的键和值）能用 OC 表示的数组、字典 | 可以，对应 `NSArray`、`NSDictionary` |
| 标记为 `@objc`、原始值是整数类型的枚举 | 可以；枚举不能带关联值 |
| 标记为 `@objc` 的协议类型 | 可以；没有 `@objc` 的 Swift 协议不行 |
| 参数和返回类型都能用 OC 表示的闭包 | 可以，对应 OC 的 Block；Block 是 OC 中可调用的代码块对象 |
| 自定义 Swift `struct`、元组、带关联值的枚举 | 通常不能直接作为这类成员的类型 |
| 泛型方法，如 `func echo<T>(_ value: T)` | 不能直接导出为 OC 方法 |

示例：

```swift
import Foundation

struct ProductInfo {
    let name: String
}

@objcMembers
final class ProductBox: NSObject {
    var title: String = "运动鞋"       // 可以暴露给 OC
    var count: Int = 1                // 可以暴露给 OC

    func show(_ productId: String) {} // 可以暴露给 OC

    var info = ProductInfo(name: "运动鞋") // 自定义 struct，不自动暴露
    func echo<T>(_ value: T) {}           // 泛型方法，不自动暴露
}
```

`info` 和 `echo` 在 Swift 中照常使用，只是 OC 看不到。如果在这两个声明上显式写 `@objc`，编译器会报错。

部分 Swift 值类型可以暴露给 OC，例如 `String` 会桥接为 `NSString`。但能桥接的类型，换一种组合形式就不一定能暴露：`Int?` 属性无法暴露为 OC 属性，因为 OC 的整数没有“空值”，而 `String?` 可以，因为 OC 对象指针可以是 nil。

`@objcMembers` 只加 `@objc`，不加 `dynamic`。对派发的影响见[@objc、selector 与 IMP](#objective-c-互操作selector-与-imp)。

## KVC 按键访问属性

KVC（Key-Value Coding，键值编码）是通过字符串键在运行时读取或设置对象属性的机制。`NSObject` 提供 `value(forKey:)` 和 `setValue(_:forKey:)` 的默认实现。自定义的 Swift 属性要支持 KVC，必须暴露给 Objective-C，通常显式标记 `@objc`，并且属性类型能用 OC 表示。只使用 KVC 不需要 `dynamic`；KVO 的要求见下文。

```swift
import Foundation

class Person: NSObject {
    @objc var name: String = ""
    @objc var age: Int = 0
}

let person = Person()
person.setValue("小明", forKey: "name")
let name = person.value(forKey: "name") as? String
```

`setValue(_:forKey:)` 收到 `"name"` 后，按命名规则查找 setter，这里找到的是 `setName:`。`value(forKey:)` 按规则查找 getter，结果以 `Any?` 返回，调用方需要转换类型。找不到访问方法时，如果类允许直接访问实例变量，还会查找名称对应的实例变量。键完全找不到时，KVC 抛出“未定义键”异常，而不是返回 `nil`；给 Int 这类不能为空的属性设置 `nil`，也会抛出异常。

`key` 是单个属性名，`keyPath` 是用点连接的属性路径，例如 `"address.city"`，`value(forKeyPath:)` 沿路径逐级读取。`#keyPath(Person.name)` 在编译时检查属性存在并生成字符串 `"name"`，手写字符串没有这项检查。Swift 原生的 `\Person.name` 是另一种东西：它是带根类型和值类型的 [KeyPath](swift-basics.md#mapkeypathsequencecollection-与-set) 值，不是字符串，也可以用于 struct。

KVC 负责按键读写属性；KVO 负责在属性变化时调用已注册的观察回调。Swift 中使用 KVO，被观察的属性需要标记 `@objc dynamic`。

## KVO 观察关系与属性访问

```swift
import Foundation

class Person: NSObject {
    @objc dynamic var age: Int = 18
}

let person = Person()
let observation = person.observe(\.age) { _, _ in
    print("收到了 age 的变化通知")
}

person.age = 20
observation.invalidate()
```

- Person 继承 NSObject，NSObject 提供 KVO 所需的实现。
- @objc 把 age 的 getter／setter 暴露给 OC；dynamic 让 Swift 代码对 age 的读写也走 OC 消息派发，KVO 才能拦截。
- observe 为 person 这一个实例的 age 注册观察。
- `person.age = 20` 调用 setter 修改属性值，KVO 在 setter 中调用观察闭包。
- observe 返回的 observation 必须在观察期间保持存活，页面中通常保存为属性；调用 invalidate 停止观察。

只写 @objc dynamic 不会注册观察，也不会产生回调；必须调用 observe。这里讲的是 Foundation 的 KVO，不是 Swift 的 Observation 框架。

## 自动 KVO 的动态子类机制

注册观察时，KVO 在运行时创建（或复用）Person 的一个子类，在子类中重写被观察属性的 setter，再把被观察对象的 isa 改为指向这个子类，于是消息查找从子类开始。下面的 KVO_Person 只是示意名称，实际类名由系统决定。

```text
未注册观察：
person → isa → Person → 原 setter

注册观察后：
person → isa → KVO_Person → 带通知逻辑的 setter → Person 原 setter
```

对象还是原来那个 person，没有被复制；变量在 Swift 中的声明类型也仍是 Person。变化的只是 isa，也就是方法查找的起点。只有注册了观察的对象的 isa 会改变，其他 Person 实例不受影响。

系统会复用已经创建的动态子类，不会每次 observe 都新建一个类。注销观察后 isa 何时恢复，属于系统内部实现，代码不能依赖。

## KVO 通知选项与自动通知

对前文的 Person.age 调用 observe 注册观察后，KVO 在属性变化时调用注册时传入的闭包，并传入被观察对象和变化信息。这里说的“通知”就是这次闭包调用，与 NotificationCenter 无关。

属性赋值时，先进入 KVO 子类重写的 setter，再由这个 setter 调用 Person 原来的 setter。下面以 age 从 18 改成 19、未指定 .prior 为例，展示调用层级。这只是效果示意，不是系统源码，开发者也不需要自己写 KVO 子类。

```text
person.age = 19
    ↓
进入 KVO 子类重写的 age setter
    ├─ willChangeValue(forKey: "age")
    │      告诉 KVO 即将修改，此时 age = 18
    ├─ 调用 Person 原来的 age setter
    │      将 age 写成 19，原 setter 返回
    ├─ didChangeValue(forKey: "age")
    │      KVO 触发修改后的通知
    │      调用观察闭包，闭包返回
    └─ KVO setter 返回
    ↓
继续执行赋值之后的代码
```

willChangeValue 和 didChangeValue 分别报告修改开始和结束，它们不是观察闭包；虽然分前后两步，默认只在 didChangeValue 中调用一次闭包。观察闭包在执行赋值的线程上同步调用：赋值语句要等闭包返回后才继续，闭包也不会被切到主线程执行。

options 是注册观察时传入的选项集合。下面沿用前文的 Person 定义；observation 需要在观察期间保留。

```swift
let person = Person() // age 初始为 18
let observation = person.observe(
    \.age,
    options: [.old, .new]
) { person, change in
    print("当前值：", person.age)
    print("旧值：", change.oldValue as Any)
    print("新值：", change.newValue as Any)
}

person.age = 19
// 闭包调用一次：当前值 19、旧值 Optional(18)、新值 Optional(19)
observation.invalidate()
```

| 选项 | 注册时 | 一次 18 → 19 修改中的闭包调用 |
| --- | --- | --- |
| [] | 不调用 | 修改后一次；不要求 change 提供旧值和新值 |
| [.old, .new] | 不调用 | 修改后一次；oldValue 为 18，newValue 为 19 |
| [.prior, .old, .new] | 不调用 | 修改前一次、修改后一次 |
| [.initial, .new] | 立即调用一次，newValue 为当前值 18 | 修改后一次，newValue 为 19 |

.old 和 .new 只决定 change 中是否包含旧值和新值，不改变通知次数。不请求这两个值时，change 中没有旧值和新值，但仍可以通过闭包参数 person 读取当前的 age。

.prior 让 KVO 在修改前多通知一次。修改前那次通知中，change.isPrior 为 true，person.age 仍是 18；修改后那次通知中，isPrior 为 false，person.age 是 19。修改前的通知没有 newValue，指定了 .old 时有旧值。

.initial 让 KVO 在 observe 返回前立即调用一次闭包，这时属性还没有被修改。这次初始通知没有 oldValue；要通过 newValue 拿到当前值，还要指定 .new。.initial 与 .prior 可以一起使用：注册时通知一次，之后每次修改前后各通知一次。以上次数都以一次普通赋值为前提。

自动通知是指系统在 setter 中自动发送修改前后的通知，前文的 @objc dynamic 属性直接赋值即可触发。

手动通知是指属性的实现者自己在修改前调用 willChangeValue(forKey:)、修改后调用 didChangeValue(forKey:)。改为手动通知时，要重写 automaticallyNotifiesObservers(forKey:)，对这个属性返回 false，其他属性交给父类实现决定。如果自动通知仍开着，又手动发送同一组通知，观察者会收到重复的通知。

## dynamic 对 KVO 访问路径的影响

**有 @objc dynamic，但这个对象还没有注册任何 KVO 观察：** 赋值仍走 OC 消息派发。isa 指向原类 Person，查找 setAge: 找到的是原 setter，所以只修改值，不发通知。dynamic 只决定“通过消息查找实现”，查到哪个实现由 isa 决定。本例的原 setter 由编译器生成；如果自己写了 setter，执行的就是自己写的逻辑。

**去掉 dynamic、只保留 @objc：** Swift 代码对 age 的赋值不再走 OC 消息派发，可能通过 Swift 虚方法表调用，也可能直接调用或内联。属性照常读写，只是不经过 OC 的方法查找。

```text
有 @objc dynamic 且注册观察：
Swift 赋值 → OC 消息派发 → KVO setter → 原 setter + 通知

只有 @objc 的一种可能路径：
Swift 赋值 → 直接调用／Swift 虚方法表 → 原 Swift setter
```

第二条路径不经过 KVO 子类的 setter，所以会出现“值改变了，观察者却没收到通知”。不过，OC 代码调用 setAge: 时，即使属性没有 dynamic，这次调用也走 OC 消息派发，仍会经过 KVO 子类。

还要注意：age 没有被观察，不代表对象的 isa 还指向原类。同一个对象的其他属性被观察后，对象的 isa 已经指向 KVO 子类。
