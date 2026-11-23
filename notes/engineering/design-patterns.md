# 设计模式与 Swift 实践

[返回笔记目录](../README.md)

## 本文目录

- [设计模式的分类与职责](#设计模式的分类与职责)
- [策略模式与评论排序](#策略模式与评论排序)
- [独立验证与页面验证](#独立验证与页面验证)
- [工厂方法、简单工厂与抽象要求](#工厂方法简单工厂与抽象要求)
- [单例、共享状态与依赖传入](#单例共享状态与依赖传入)
- [建造者与链式调用](#建造者与链式调用)
- [耦合与解耦](#耦合与解耦)
  - [实例一：通过协议隔离具体存储实现](#实例一通过协议隔离具体存储实现)
  - [实例二：通过回调隔离通用组件与具体页面](#实例二通过回调隔离通用组件与具体页面)
  - [实例三：个人页登录模块通过通知告知其他页面](#实例三个人页登录模块通过通知告知其他页面)
- [依赖反转与依赖注入](#依赖反转与依赖注入)
  - [源码依赖方向与运行时调用顺序](#源码依赖方向与运行时调用顺序)
  - [依赖反转与依赖注入的区别](#依赖反转与依赖注入的区别)

## 设计模式的分类与职责

设计模式是针对某类反复出现的软件设计问题，总结出的通用组织思路，描述代码分成哪些角色、各自负责什么以及如何协作。

例如，同一项任务有多种可替换算法，可以把算法分别封装，通过统一接口使用。这种组织方式叫策略模式；Swift 中可以用协议和具体类型实现，也可以在简单场景下用闭包表达。同一个模式可以用多种语言机制和代码结构实现。

| 分类 | 主要关注点 | 模式 |
|-|-|-|
| 创建型，5 种 | 对象怎样创建 | 单例、工厂方法、抽象工厂、建造者、原型 |
| 结构型，7 种 | 类和对象怎样组合 | 适配器、桥接、组合、装饰器、外观、享元、代理（Proxy） |
| 行为型，11 种 | 职责怎样分配、对象怎样协作 | 观察者、策略、模板方法、责任链、命令、迭代器、中介者、备忘录、状态、访问者、解释器 |

下文详细讨论策略、工厂方法、单例和建造者，其他模式仅作分类概览。

| 模式 | 直观含义 | 示例 |
|-|-|-|
| 观察者 | 状态变化时通知关注它的对象 | NotificationCenter、KVO |
| 策略 | 将不同算法封装起来，按需替换 | 评论按时间或点赞数排序 |
| 单例 | 统一访问一个共享实例 | App 级共享服务或配置管理对象 |
| 工厂方法 | 定义创建方法，由子类决定具体创建什么对象 | 图文和视频入口分别创建对应详情页 |
| 适配器 | 把已有接口转换成调用方需要的接口 | 将第三方 SDK 包装成项目统一协议 |
| 装饰器 | 包装原有对象，为其增加能力 | 给请求处理能力增加日志或缓存包装 |
| 外观 | 给复杂子系统提供简单入口 | “开始上传”内部协调压缩、鉴权与上传 |
| 责任链 | 多个处理者依次决定如何处理请求 | 请求依次经过不同校验器 |
| 状态 | 对象在不同状态下采用不同行为 | 播放器在播放、暂停、停止状态下处理操作 |
| 模板方法 | 父类规定流程，子类实现其中部分步骤 | 通用处理流程中的可定制步骤 |

- 闭包是 Swift 的语言机制，可以用来实现回调或策略；GoF 的 23 种模式中没有名为“闭包”的模式。
- MVC、MVVM 是架构或表现层的组织方式，不在上述 23 种之列。
- iOS 的 Delegate 是委托机制：一个对象把部分事件处理或决策交给另一个对象完成。Delegate 中文也常叫“代理”，与 GoF 的代理模式（Proxy）是两回事；Proxy 用一个替身对象控制对原对象的访问。

## 策略模式与评论排序

策略模式把完成同一任务的不同算法分别封装，通过统一接口使用，并允许替换。评论列表中的“最新优先”和“点赞优先”就是两种排序策略。

三个角色分别是：策略接口、具体策略、使用策略的对象。

```swift
struct Comment {
    let text: String
    let likeCount: Int
    let publishedAt: Int // 简化的时间戳
}

protocol CommentSortStrategy {
    func sort(_ comments: [Comment]) -> [Comment]
}

struct LatestFirstStrategy: CommentSortStrategy {
    func sort(_ comments: [Comment]) -> [Comment] {
        comments.sorted { $0.publishedAt > $1.publishedAt }
    }
}

struct MostLikedFirstStrategy: CommentSortStrategy {
    func sort(_ comments: [Comment]) -> [Comment] {
        comments.sorted { $0.likeCount > $1.likeCount }
    }
}

final class CommentList {
    var sortStrategy: any CommentSortStrategy

    init(sortStrategy: any CommentSortStrategy) {
        self.sortStrategy = sortStrategy
    }

    func sortedComments(_ comments: [Comment]) -> [Comment] {
        sortStrategy.sort(comments)
    }
}

let comments = [
    Comment(text: "评论A", likeCount: 10, publishedAt: 100),
    Comment(text: "评论B", likeCount: 30, publishedAt: 200),
    Comment(text: "评论C", likeCount: 20, publishedAt: 300)
]

let list = CommentList(sortStrategy: LatestFirstStrategy())
print(list.sortedComments(comments).map(\.text))
// ["评论C", "评论B", "评论A"]

list.sortStrategy = MostLikedFirstStrategy()
print(list.sortedComments(comments).map(\.text))
// ["评论B", "评论C", "评论A"]
```

CommentList 是使用策略的对象，只调用 sortStrategy.sort(_:)。给 sortStrategy 赋另一个策略，排序行为就随之改变。增加排序算法时，新增一个遵循 CommentSortStrategy 的类型即可。决定使用哪个策略的代码仍然可以用 switch。

简单算法也可以通过比较闭包传入：

```swift
var sortRule: (Comment, Comment) -> Bool = {
    $0.publishedAt > $1.publishedAt
}
let latestComments = comments.sorted(by: sortRule)

sortRule = { $0.likeCount > $1.likeCount }
let popularComments = comments.sorted(by: sortRule)
```

sorted(by:) 负责排序，闭包决定两个元素谁在前面。规则只有一两行时可以直接使用闭包；算法复杂、需要配置或依赖时，再考虑独立策略类型。策略模式的价值在于隔离算法变化、统一调用入口和方便替换算法。

## 独立验证与页面验证

独立验证和页面验证都要执行代码，两者的区别在于从哪里进入、依赖什么：

```text
页面：运行 App → 进入页面 → 生命周期／交互 → 排序 → 检查
测试：运行测试 → 直接调用排序 → 检查
```

沿用[策略示例](#策略模式与评论排序)的数据，可以直接输入并检查：

```swift
let strategy = LatestFirstStrategy()
let result = strategy.sort(comments)
assert(result.map(\.text) == ["评论C", "评论B", "评论A"])
assert(strategy.sort([]).isEmpty)
```

测试运行器也可直接执行：

```swift
import XCTest

final class CommentSortTests: XCTestCase {
    func testEmptyComments() {
        XCTAssertTrue(LatestFirstStrategy().sort([]).isEmpty)
    }
}
```

独立验证有三个好处：页面还没完成时就能验证算法；入口短，便于自动重复执行；容易构造空数组、单元素、发布时间相同等边界输入。独立验证失败时，只需检查输入、算法逻辑和预期值；独立验证通过但页面显示错误时，再检查页面的数据绑定和刷新。

在页面里构造模拟数据，同样可以不依赖服务端。模拟数据写在 viewDidLoad 中时，需要运行 App 并进入页面才会执行；页面方法本身不依赖 UI 时，也可以在测试中直接调用。

部分 iOS 测试仍会启动宿主 App。代码能否独立测试，取决于逻辑是否与 UI 分离、依赖和入口是否可控；普通函数也能满足这些条件。策略模式解决的是多种算法的统一组织与替换，与可测试性是两件事。算法验证通过后，仍需要页面或集成验证确认界面行为。测试运行器和 Xcode 配置见 [Xcode 工程与自动化测试](xcode-and-testing.md)。

## 工厂方法、简单工厂与抽象要求

工厂方法模式由父类定义创建方法，子类决定具体创建什么对象。父类的通用业务流程可以通过这个方法取得对象并继续工作。

假设已经有 ImagePostDetailViewController 和 VideoPostDetailViewController，两者继承 UIViewController，并支持 init(postID:)。下面的代码省略这两个页面的实现：

```swift
class PostDetailEntry {
    func makeDetailPage(postID: String) -> UIViewController {
        fatalError("子类必须实现这个方法")
    }

    func open(
        postID: String,
        in navigationController: UINavigationController
    ) {
        let page = makeDetailPage(postID: postID)
        navigationController.pushViewController(page, animated: true)
    }
}

final class ImagePostDetailEntry: PostDetailEntry {
    override func makeDetailPage(postID: String) -> UIViewController {
        ImagePostDetailViewController(postID: postID)
    }
}

final class VideoPostDetailEntry: PostDetailEntry {
    override func makeDetailPage(postID: String) -> UIViewController {
        VideoPostDetailViewController(postID: postID)
    }
}

let entry: PostDetailEntry = ImagePostDetailEntry()
entry.open(postID: "123", in: navigationController)
```

执行顺序：entry 调用父类提供的 open → open 调用 makeDetailPage → entry 的实际类型是 ImagePostDetailEntry，因此执行 ImagePostDetailEntry 重写的 makeDetailPage → 返回图文详情页 → open 把页面压入导航栈。

把 entry 换成 VideoPostDetailEntry 后，open 流程不变，得到的是视频详情页。

**价值与简单工厂：**

如果 open 直接创建 ImagePostDetailViewController，open 就和图文详情页的创建方式绑定。之后增加视频页，或者页面创建需要播放器等依赖时，这些创建分支可能不断加进 open 的通用流程。

工厂方法把会变化的创建步骤交给子类，带来三点好处：

- 不同页面复用同一个 open 流程。
- 视频页的创建方式变化时，主要修改 VideoPostDetailEntry。
- 新增页面类型时，新增一个入口子类，再在上层选择入口的地方接入；open 的通用流程不用改。

上层代码仍要决定使用哪个入口子类，这里通常仍有分支。工厂方法隔离的是创建细节；新增类型时，负责选择入口的上层代码仍需修改。

如果一个静态方法在内部集中判断：

```swift
static func makeDetailPage(post: Post) -> UIViewController {
    switch post.type {
    case .image:
        return ImagePostDetailViewController(postID: post.id)
    case .video:
        return VideoPostDetailViewController(postID: post.id)
    }
}
```

这种写法通常叫简单工厂。这里的 Post 是假设含有 type 和 id 的模型，与后文建造者小节的 Post 不是同一个类型。

| 写法 | 谁决定具体类型 |
|-|-|
| 简单工厂 | 一个工厂内部通过 switch 等逻辑决定 |
| 工厂方法模式 | 子类分别重写创建方法，各自决定 |

判断是否采用了 GoF 工厂方法模式，要看是否由子类重写创建方法来决定具体类型，与方法名是否叫 make、create 无关。页面类型少、构造简单时，直接初始化或简单工厂就够用，不必为套用模式增加继承层级。

**Swift 的抽象方法边界：**

抽象方法表示父类规定一个方法要求，但不提供具体实现，由具体子类补齐。有些语言可以使用 abstract 声明这种要求。

Swift 没有声明抽象方法的 abstract 关键字。上例在父类的 makeDetailPage 中调用 fatalError，用来模拟“子类必须重写”的约定：子类没有重写时，编译仍能通过；程序运行到父类的 makeDetailPage 时才崩溃。

Swift 协议可以在编译期表达能力要求：

```swift
protocol PostDetailCreating {
    func makeDetailPage(postID: String) -> UIViewController
}
```

遵循 PostDetailCreating 的类型如果没有实现 makeDetailPage，编译器直接报错。协议要求和抽象父类是两种机制：协议要求在编译期检查；fatalError 占位只在运行时暴露遗漏，属于示例写法。

## 单例、共享状态与依赖传入

单例让某个类型在一次 App 运行中通过统一入口提供一个共享实例，限制外部自行创建。示例让多个页面共享浏览记录：

```swift
final class PostHistoryManager {
    static let shared = PostHistoryManager()
    private init() {}

    private var viewedPostIDs: Set<String> = []

    func record(postID: String) {
        viewedPostIDs.insert(postID)
    }

    func hasViewed(postID: String) -> Bool {
        viewedPostIDs.contains(postID)
    }
}

PostHistoryManager.shared.record(postID: "123")
let viewed = PostHistoryManager.shared.hasViewed(postID: "123")
// true

let first = PostHistoryManager.shared
let second = PostHistoryManager.shared
print(first === second)
// true
```

- static let shared：类型级别保存固定的共享实例，首次访问时初始化。
- private init：外部不能直接 PostHistoryManager() 创建另一个实例。
- final：禁止子类继承。

这是单例的常见写法。private init 只阻止类型外部创建实例；PostHistoryManager 内部仍能调用 init，因此还需要类型自身不再创建其他实例。

使用单例时注意以下几点：

- static let 只保证 shared 初始化一次。viewedPostIDs 这类可变状态被多个线程同时读写时，仍需通过隔离或加锁等方式同步。
- 单例的数据只保存在内存中，进程结束后 viewedPostIDs 中的记录就会丢失。
- 不同账号或页面需要各自独立的状态时，不应使用单例。是否使用单例取决于状态是否真的需要全局共享，与类名是否叫 Manager 无关。
- 多个测试共用同一个单例时，前一个测试写入的状态可能影响后一个测试。

**解决的问题与替代方式：**

一份应该共享的状态，如果被各处分别创建成多个对象，就会分散在这些对象中。单例保证只有一个实例，所有调用方通过 shared 访问同一份状态。

用一个非单例的登录状态示例说明：

```swift
final class LoginState {
    var isLoggedIn = false
}

let loginPageState = LoginState()
let homePageState = LoginState()

// 假设登录页已收到登录成功结果
loginPageState.isLoggedIn = true
print(homePageState.isLoggedIn)
// false
```

loginPageState 和 homePageState 是两个不同的 LoginState 对象，各自保存 isLoggedIn。修改 loginPageState 不会影响 homePageState，因此可能出现“登录页已成功，首页仍显示请登录”。

问题的原因是应该共享的状态建了两份。只创建一个普通对象并传给两个页面，同样能解决：

```swift
let sharedState = LoginState()
let loginPageState = sharedState
let homePageState = sharedState

loginPageState.isLoggedIn = true
print(homePageState.isLoggedIn)
// true
```

单例只是共享状态的方式之一，更适合确实需要全局生命周期的服务。只在有限范围内共享的对象，可以由上层创建一次，再传给各个使用方。

## 建造者与链式调用

建造者把一个对象的构建过程拆成多个步骤，由专门对象收集和组装信息，最后生成结果。简化的帖子构建过程可以是：准备正文 → 添加图片 → 设置位置 → 生成帖子。

```swift
struct Post {
    let content: String
    let imageIDs: [String]
    let location: String?
}

final class PostBuilder {
    private let content: String
    private var imageIDs: [String] = []
    private var location: String?

    init(content: String) {
        self.content = content
    }

    func addImage(_ imageID: String) -> PostBuilder {
        imageIDs.append(imageID)
        return self
    }

    func setLocation(_ location: String) -> PostBuilder {
        self.location = location
        return self
    }

    func build() -> Post {
        Post(
            content: content,
            imageIDs: imageIDs,
            location: location
        )
    }
}

let post = PostBuilder(content: "今天去海边了")
    .addImage("image_001")
    .addImage("image_002")
    .setLocation("深圳")
    .build()

let textPost = PostBuilder(content: "今天心情不错")
    .build()
```

PostBuilder 在构建期间保存信息并提供各个步骤，Post 是最终结果。创建 PostBuilder 时先提供正文，之后按需补充图片或位置，最后调用 build 组装出 Post。

本例只有一个建造者。GoF 建造者模式还有两种扩展形式：由 Director 对象按固定顺序调用建造步骤；用多个建造者从同一组步骤生成不同表示。

**链式调用与适用条件：**

addImage 和 setLocation 都返回 self，也就是同一个 PostBuilder 对象，所以可以在返回值上继续调用 PostBuilder 的方法：

```swift
builder.addImage("image_001").setLocation("深圳")
```

同样可以分步骤写：

```swift
let builder = PostBuilder(content: "今天去海边了")
_ = builder.addImage("image_001")
_ = builder.setLocation("深圳")
let post = builder.build()
```

链式调用只是一种写法，任何返回 self 的方法都能链式调用。建造者模式的核心是分步收集信息、最后生成结果，与是否链式调用无关。

当创建过程有多个步骤、多个子对象或复杂组装规则时，建造者集中处理组装，调用方按步骤提供信息即可。可选步骤允许形成不同配置，不要求一次传入所有数据。

对于示例里仅有几个字段的 Post，直接初始化同样清楚：

```swift
let post = Post(
    content: "今天去海边了",
    imageIDs: ["image_001", "image_002"],
    location: "深圳"
)
```

Swift 的参数标签和默认参数已经可以解决不少简单构造问题，不必为了模式额外增加 Builder。

## 耦合与解耦

耦合是代码单元之间的依赖关系及其紧密程度。代码单元可以是函数、类或模块。如果 A 需要调用 B 的方法才能完成工作，A 就依赖 B。A 使用 B 定义的数据结构、与 B 共享状态，或者要求 B 先执行，也都是依赖；这些情况下 A 不一定调用 B 的方法。

当 A 需要了解 B 的具体类型、内部结构或处理步骤时，B 修改实现更容易迫使 A 一起修改。解耦是调整这些依赖，减少不必要的依赖并限制依赖范围，从而缩小变化的影响。

解耦可以减少修改范围、方便独立测试，并支持组件复用和实现替换。例如，业务逻辑能够使用测试对象时，验证业务规则就不必依赖真实文件或网络。调用方改为依赖接口后，接口的方法和行为约定就是调用方的依赖；约定变化时，调用方仍可能需要修改。

解耦只去掉不必要的依赖，必要的依赖仍然保留，也不必给每个类都加协议。先确认现有依赖是否妨碍修改、测试或复用，再采用满足当前需求的直接方案。

### 实例一：通过协议隔离具体存储实现

假设 ProfileModel 需要读取用户名。如果 ProfileModel 自行创建 FileNameStore 并调用 readName()，ProfileModel 就同时决定了需要什么能力和使用哪种存储实现。更换存储方式，或者测试时不想访问真实文件，都要修改 ProfileModel 中创建 FileNameStore 的代码。

改进方式是用 NameStore 协议声明读取能力，再由外部传入遵循 NameStore 的对象。下面的 FixedNameStore 返回固定用户名，用来代替真实的文件存储；示例省略了文件读取和错误处理。

```swift
protocol NameStore {
    func readName() -> String
}

final class ProfileModel {
    private let store: any NameStore

    init(store: any NameStore) {
        self.store = store
    }

    func loadName() -> String {
        store.readName()
    }
}

struct FixedNameStore: NameStore {
    let name: String

    func readName() -> String {
        name
    }
}

let profile = ProfileModel(store: FixedNameStore(name: "小明"))
print(profile.loadName()) // 小明
```

ProfileModel 只依赖 NameStore 的能力约定，不依赖 FixedNameStore 或 FileNameStore 的具体类型。正常运行时可以传入符合约定的文件存储或数据库存储；测试时可以传入 FixedNameStore。具体存储对象由外部代码创建和选择，因此创建具体存储的代码集中在组装 ProfileModel 的位置。

### 实例二：通过回调隔离通用组件与具体页面

如果工具栏直接持有某个具体页面控制器，并调用该控制器的跳转方法，工具栏就依赖这个页面。可以让工具栏只报告按钮点击事件，由使用工具栏的页面决定后续行为。

```swift
final class ToolbarActions {
    var onEditTapped: (() -> Void)?

    func editTapped() {
        onEditTapped?()
    }
}
```

ToolbarActions 只演示事件回调，省略了界面代码。页面给 onEditTapped 赋值；用户点击编辑按钮时，按钮调用 editTapped()；editTapped() 再执行页面提供的回调。执行顺序是：用户点击按钮 → ToolbarActions 报告点击事件 → 页面决定后续行为。

这样做去掉了 ToolbarActions 对具体页面和跳转逻辑的依赖。ToolbarActions 和页面仍需共同遵守回调的参数和事件含义。页面传入的闭包如果捕获了其他对象，页面需要管理这些对象的生命周期。

### 实例三：个人页登录模块通过通知告知其他页面

个人页的登录模块确认登录成功后，首页、消息页等页面可能需要刷新用户相关内容。如果登录模块直接持有这些页面并逐个调用刷新方法，登录模块就依赖具体页面；增加一个需要刷新的页面时，登录模块也可能需要修改。

NotificationCenter（通知中心）是负责分发通知的对象：发送方发布通知后，NotificationCenter 根据通知名称和发送者等条件，把通知转交给已注册的接收者。登录模块可以通过 NotificationCenter 发送登录成功通知。其他页面分别注册接收同一个通知，收到通知后执行各自的刷新逻辑。登录模块只负责发布事件，不需要知道有哪些页面正在监听。

下面的示例用打印代替页面刷新，省略了登录请求、状态存储和界面代码。个人页的登录流程在确认登录成功并保存登录状态后，调用 loginDidSucceed()。

```swift
import Foundation

extension Notification.Name {
    static let userDidLogin = Notification.Name("userDidLogin")
}

final class ProfileLoginModule {
    // 由真实登录流程在确认成功并保存登录状态后调用。
    func loginDidSucceed() {
        NotificationCenter.default.post(
            name: .userDidLogin,
            object: nil
        )
    }
}

// 首页先注册，保存返回的观察者凭据。
let homeObserver = NotificationCenter.default.addObserver(
    forName: .userDidLogin,
    object: nil,
    queue: .main
) { _ in
    print("首页收到登录成功通知，刷新用户相关内容")
}

// 消息页独立注册，不需要经过个人页。
let messageObserver = NotificationCenter.default.addObserver(
    forName: .userDidLogin,
    object: nil,
    queue: .main
) { _ in
    print("消息页收到登录成功通知，刷新消息相关内容")
}

// 假设此时真实登录已成功，登录状态也已保存。
let loginModule = ProfileLoginModule()
loginModule.loginDidSucceed()
```

执行顺序是：其他页面先注册接收通知 → 个人页登录模块确认登录成功并保存状态 → 发布 userDidLogin → 通知中心分发通知 → 已注册的页面各自读取当前状态并刷新。queue: .main 让两个回调都在主队列执行；示例代码不依赖首页与消息页两个回调的先后顺序。

addObserver 返回一个观察者对象，用来标识这次注册。页面需要保存这个返回值，即示例中的 homeObserver 和 messageObserver。页面不再需要监听时，用保存的返回值调用 removeObserver。下面两条语句分别在首页、消息页结束监听时执行，不是紧跟在注册语句之后执行：

```swift
// 首页结束监听时执行。
NotificationCenter.default.removeObserver(homeObserver)

// 消息页结束监听时执行。
NotificationCenter.default.removeObserver(messageObserver)
```

这样做去掉了登录模块对首页、消息页等具体页面的直接依赖。新增一个接收页面时，只需让新页面注册通知，登录模块不用增加对新页面的调用。发送方和接收方仍然共同依赖同一个通知中心、通知名称和事件含义，这些约定可以放在独立的公共事件定义中。

通知只表示“登录成功”这一次事件，登录状态本身仍需另外保存。通知发出之后才注册的页面收不到这次通知，因此页面在需要展示时应读取当前登录状态。真实页面的回调如果捕获页面自身，还要管理捕获关系和取消监听的时机，避免页面被不必要地长期持有。

## 依赖反转与依赖注入

依赖反转是一条设计原则，用于让高层业务逻辑与底层具体实现都依赖抽象，并由具体实现满足业务所需要的能力。高层业务逻辑决定完成什么业务；底层具体实现提供文件、数据库或网络等技术能力；抽象只声明需要哪些能力和行为约定，具体怎么实现由底层决定。Swift 可以使用协议表达这种抽象。

依赖反转包含两项要求：高层模块和底层具体实现都依赖抽象，高层模块不直接引用底层具体实现；抽象不依赖实现细节，具体实现依赖并遵守抽象。抽象按业务需要来定义，底层的内部结构留在具体实现中。

### 源码依赖方向与运行时调用顺序

沿用用户名存储示例，假设 FileNameStore 是具体文件存储类型。依赖调整前后可以表示为：

```text
原来：ProfileModel → FileNameStore
调整后：ProfileModel → NameStore ← FileNameStore
```

箭头 A → B 表示 A 的源码需要引用 B 的声明。调整后，ProfileModel 使用 NameStore，FileNameStore 遵守 NameStore。ProfileModel 不再引用 FileNameStore；FileNameStore 反过来依赖业务所要求的 NameStore 协议。被反转的是源码依赖方向。

运行时调用顺序没有变化：ProfileModel 调用 readName() → 实际传入的存储对象执行读取 → 返回用户名。调用仍由 ProfileModel 发起。

当代码分成独立模块时，NameStore 可以放在业务模块或独立接口模块中。业务模块通过 NameStore 使用能力，具体存储模块依赖 NameStore 的声明。如果 NameStore 放在具体存储模块里，业务模块为了使用 NameStore 就必须依赖整个存储模块，模块之间的依赖方向就没有反转。

只要 NameStore 的行为约定保持稳定，业务代码就可以换用另一种符合约定的实现，无须了解文件或数据库的内部细节。依赖反转减少了具体实现变化对业务代码的影响；业务代码仍然依赖 NameStore 协议。

### 依赖反转与依赖注入的区别

依赖注入是由外部向对象提供所需依赖的方式。示例中的 init(store:) 属于构造器注入：ProfileModel 接收外部创建的存储对象，而不是自行创建存储对象。

依赖反转规定源码应该依赖什么，关注业务与实现之间的依赖方向；依赖注入规定所需对象由谁提供，关注对象的组装方式。示例同时采用二者：ProfileModel 依赖 NameStore 抽象，并通过初始化方法接收具体对象。

如果初始化参数的类型写成具体类型 FileNameStore，从外部传入对象仍属于依赖注入；但 ProfileModel 的源码仍然引用 FileNameStore，依赖方向没有反转。实现依赖反转，参数类型需要是 NameStore 这样的抽象。
