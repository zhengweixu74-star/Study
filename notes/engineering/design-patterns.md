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
- [依赖反转与依赖注入](#依赖反转与依赖注入)

## 设计模式的分类与职责

设计模式是针对某类反复出现的软件设计问题，总结出的通用组织思路，描述代码分成哪些角色、各自负责什么以及如何协作。

例如，同一项任务有多种可替换算法，可以把算法分别封装，通过统一接口使用。这种组织方式叫策略模式；Swift 中可以用协议和具体类型实现，也可以在简单场景下用闭包表达。模式不等于某个关键字，也不是只有一种固定代码模板。

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

- 闭包是语言机制，可以用来实现回调或策略；闭包本身不是一个独立的 GoF 设计模式名称。
- MVC、MVVM 通常讨论的是架构或表现层组织方式，不属于上述 23 种。
- iOS 的 Delegate 是委托机制，中文也常叫“代理”，但不等同于 GoF 的 Proxy。

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

CommentList 是使用策略的示例对象，不是完整的 UIKit 列表视图。它只调用排序接口；换一个策略就能改变排序行为。增加其他排序算法时，可以增加对应实现。选择策略的地方仍然可以使用 switch，模式并不要求消灭所有条件分支。

简单算法也可以通过比较闭包传入：

```swift
var sortRule: (Comment, Comment) -> Bool = {
    $0.publishedAt > $1.publishedAt
}
let latestComments = comments.sorted(by: sortRule)

sortRule = { $0.likeCount > $1.likeCount }
let popularComments = comments.sorted(by: sortRule)
```

sorted(by:) 负责排序，闭包决定两个元素谁在前面。规则只有一两行时可以直接使用闭包；算法复杂、需要配置或依赖时，再考虑独立策略类型。策略的主要价值是隔离算法变化、统一调用和方便替换，不在于增加类型数量。

## 独立验证与页面验证

独立验证仍要执行代码，区别是入口和依赖：

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

好处是页面未完成也可验证，入口短、便于自动重复，容易构造空数组、单元素、相同时间等边界。算法失败时集中检查输入、逻辑和预期；算法正确但页面错误，再检查绑定和刷新。

页面里构造模拟数据同样可以绕过服务端；放在 viewDidLoad 中要执行到该入口，若页面方法本来无 UI 依赖，也可以直接调用。独立测试不等于不执行代码，也不保证所有 iOS 测试都不启动宿主 App。可测试性来自逻辑与 UI 分离、依赖与入口可控，普通函数也能做到；策略模式另行解决多算法的统一组织与替换。算法验证不能替代页面／集成验证。本节排序示例未实际创建或运行；测试运行器、Xcode 配置和已创建的练习另见 [Xcode 工程与自动化测试](xcode-and-testing.md)。

## 工厂方法、简单工厂与抽象要求

工厂方法模式由父类定义创建方法，子类决定具体创建什么对象。父类的通用业务流程可以通过这个方法取得对象并继续工作。

假设已经有 ImagePostDetailViewController 和 VideoPostDetailViewController，两者继承 UIViewController，并支持 init(postID:)。下面省略这些页面的实现，仅演示入口与创建方法的关系：

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

执行顺序：调用父类提供的 open → open 调用 makeDetailPage → 按实际子类执行图文入口的重写实现 → 返回图文详情页 → open 完成导航跳转。

换成 VideoPostDetailEntry，复用相同 open 流程，但获得视频详情页。这里的页面类、初始化接口及 entry 结构用于演示工厂方法的职责分工。

**价值与简单工厂：**

如果 open 直接创建 ImagePostDetailViewController，它就和该页面的创建方式绑定。增加视频页、播放器等创建依赖时，创建分支可能不断进入通用流程。

工厂方法将可变化的创建步骤交给子类，主要获得：

- 不同页面复用通用打开流程。
- 视频页创建方式变化时，主要修改视频入口。
- 新增页面类型时，可新增入口子类，并在上层接入选择；open 的通用流程可以不变。

上层仍需选择入口类型。模式隔离创建细节，不保证所有调用方永远不用修改，也不意味着完全没有分支。

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

这种通常叫简单工厂。此处 Post 是假设含有 type 和 id 的示意模型，与后面建造者小节的同名模型分开阅读。

| 写法 | 谁决定具体类型 |
|-|-|
| 简单工厂 | 一个工厂内部通过 switch 等逻辑决定 |
| 工厂方法模式 | 子类分别重写创建方法，各自决定 |

名称叫 make、create 或“工厂方法”，不自动意味着采用了 GoF 工厂方法模式。只有少量页面且构造简单时，直接初始化或简单工厂可以足够，不必为模式增加继承层级。

**Swift 的抽象方法边界：**

抽象方法表示父类规定一个方法要求，但不提供具体实现，由具体子类补齐。有些语言可以使用 abstract 声明这种要求。

Swift 没有直接声明抽象方法的 abstract 关键字。上例中的 fatalError 只是模拟“子类必须重写”的约定：编译器不会因为子类没重写就拒绝编译；执行到父类的这个实现时才会发生运行时错误。

Swift 协议可以在编译期表达能力要求：

```swift
protocol PostDetailCreating {
    func makeDetailPage(postID: String) -> UIViewController
}
```

遵循者必须满足该要求；协议要求与抽象父类是不同机制。不能把 fatalError 当成 Swift 内置的抽象方法能力，也不能把这类示意占位当作已经安全完成的真实项目实现。

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

这套实现是统一共享对象的常见写法；private init 限制外部构造，仍需类型自身遵守不另造其他实例的设计。

边界：static let 的一次初始化保证，不等于内部 Set 等可变状态自动并发安全；并发访问仍需明确隔离或同步。单例也不自动持久化，示例的内存记录不会保留到进程结束之后。不同账号或页面确实需要独立状态时，不应仅因为类名包含 Manager 就设成单例。共享可变状态还可能让测试互相影响。

**解决的问题与替代方式：**

需要共享的状态如果被各处分别创建，就会分散。单例固定对象身份、共享内部状态，并提供统一入口。

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

登录页使用对象 A，首页使用对象 B；修改 A 不会同步到 B，因此可能出现“登录页已成功，首页仍显示请登录”。这个例子说明不同对象各自保存状态时，状态不会自动同步。

问题来自“应该共享的状态建了两份”，不代表不用单例就一定出错。只创建一个普通对象并传给两个页面也能解决：

```swift
let sharedState = LoginState()
let loginPageState = sharedState
let homePageState = sharedState

loginPageState.isLoggedIn = true
print(homePageState.isLoggedIn)
// true
```

单例是获取共享对象的一种方式，不是共享状态的唯一方式。它更适合确实需要全局生命周期的服务；有限范围内共享对象，也可以由上层创建一次后传入。

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

PostBuilder 保存构建期间的信息并提供步骤；Post 是最终结果。不同需求可以选择不同步骤，正文先提供，图片或位置稍后补充，最后调用 build 组装。

本例展示单个建造者的简化形式，不包含 Director 或多个建造者生成不同表示的变体。

**链式调用与适用条件：**

addImage 和 setLocation 返回 self，也就是原来的建造者，所以可以继续调用它的方法：

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

链式调用是一种写法；建造者的核心是分步构建与结果生成，不能仅凭方法返回 self 就认定一定使用了建造者模式。

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

耦合是代码单元之间的依赖关系及其紧密程度。代码单元可以是函数、类或模块。如果 A 需要调用 B 的方法才能完成工作，A 就依赖 B。依赖也可以体现在数据结构、共享状态和执行顺序上，不要求双方互相调用。

当 A 需要了解 B 的具体类型、内部结构或处理步骤时，B 修改实现更容易迫使 A 一起修改。解耦是调整这些依赖，减少不必要的依赖并限制依赖范围，从而缩小变化的影响。

解耦可以减少修改范围、方便独立测试，并支持组件复用和实现替换。例如，业务逻辑能够使用测试对象时，验证业务规则就不必依赖真实文件或网络。接口的方法和行为约定仍然是依赖；约定变化时，调用方仍可能需要修改。

解耦不要求消除所有依赖，也不要求给每个类增加协议。应先确认现有依赖是否妨碍修改、测试或复用，再采用满足当前需求的直接方案。

### 实例一：通过协议隔离具体存储实现

假设 ProfileModel 需要读取用户名。如果 ProfileModel 自行创建 FileNameStore 并调用 readName()，业务对象就同时决定了所需能力和具体存储方式。更换文件存储，或者在测试中避免访问文件时，都要处理这个具体依赖。

可以用 NameStore 声明读取能力，再由外部传入满足协议的对象。下面的独立示例使用固定值实现说明替换关系，假定用户名已存在，不展开真实文件读取与错误处理。

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

ProfileModel 只依赖 NameStore 的能力约定，不依赖 FixedNameStore 或 FileNameStore 的具体类型。正常运行时可以传入符合约定的文件存储或数据库存储；测试时可以传入固定值实现。具体对象仍由外部创建并选择，相关创建依赖集中在组装位置。

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

ToolbarActions 是独立的事件回调示例，不包含界面实现。页面设置 onEditTapped；按钮点击时调用 editTapped()；editTapped() 再执行页面提供的回调。执行顺序是：用户点击按钮 → 组件报告点击事件 → 页面决定后续行为。

减少的是通用组件对具体页面和跳转逻辑的依赖。双方仍然需要遵守回调的参数和事件含义；闭包捕获对象时的生命周期也需要由调用方管理。

### 实例三：个人页登录模块通过通知告知其他页面

个人页的登录模块确认登录成功后，首页、消息页等页面可能需要刷新用户相关内容。如果登录模块直接持有这些页面并逐个调用刷新方法，登录模块就依赖具体页面；增加一个需要刷新的页面时，登录模块也可能需要修改。

可以通过 NotificationCenter 发送登录成功通知。通知中心是根据通知名称和发送者等条件，把通知分发给已注册接收者的对象。其他页面分别注册接收同一个通知，收到通知后执行各自的刷新逻辑。登录模块只负责发布事件，不需要知道有哪些页面正在监听。

下面是独立的通信示例，用打印代替实际页面刷新。个人页的登录流程在确认成功并保存登录状态后调用 loginDidSucceed()。示例不包含登录请求、状态存储和界面实现。

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

执行顺序是：其他页面先注册接收通知 → 个人页登录模块确认登录成功并保存状态 → 发布 userDidLogin → 通知中心分发通知 → 已注册的页面各自读取当前状态并刷新。本例使用 queue: .main 指定回调在主队列执行，不依赖首页与消息页两个回调之间的先后顺序。

addObserver 返回的对象用于标识这次注册，页面应保存该对象。在对应页面不再需要监听时，分别执行以下清理；这些语句表示页面结束监听时的操作，不应紧接注册语句执行。

```swift
// 首页结束监听时执行。
NotificationCenter.default.removeObserver(homeObserver)

// 消息页结束监听时执行。
NotificationCenter.default.removeObserver(messageObserver)
```

减少的是登录模块对首页、消息页等具体页面的直接依赖。新增一个接收页面时，只需让新页面注册通知；发送方无须增加对新页面的调用。双方仍然依赖同一个通知中心、通知名称和事件含义，这些约定可以放在独立的公共事件定义中。

通知表示一次事件，不替代登录状态的保存。通知发出后才注册的页面不会补收到这次历史通知，应在需要展示时读取当前登录状态。实际页面回调如果捕获自身，还应管理捕获关系和取消监听的时机，避免不必要的长期持有。

## 依赖反转与依赖注入

依赖反转是一条设计原则，用于让高层业务逻辑与底层具体实现都依赖抽象，并由具体实现满足业务所需要的能力。高层业务逻辑决定完成什么业务；底层具体实现提供文件、数据库或网络等技术能力；抽象声明能力和行为约定，不规定具体实现方式。Swift 可以使用协议表达这种抽象。

依赖反转包含两项要求：高层模块不直接依赖底层具体实现，两者都依赖抽象；抽象不依赖实现细节，具体实现依赖并遵守抽象。抽象应表达业务需要，而不是把底层内部结构原样暴露给业务。

### 源码依赖方向与运行时调用顺序

沿用用户名存储示例，假设 FileNameStore 是具体文件存储类型。依赖调整前后可以表示为：

```text
原来：ProfileModel → FileNameStore
调整后：ProfileModel → NameStore ← FileNameStore
```

箭头表示代码需要引用目标类型或协议的声明。ProfileModel 使用 NameStore，FileNameStore 遵守 NameStore。业务代码不再直接引用具体文件存储类型，而具体文件存储实现需要遵守业务所要求的协议。反转的是源码依赖方向。

运行时调用顺序仍然是：ProfileModel 调用 readName() → 实际传入的存储对象执行读取 → 返回用户名。依赖反转不会把这个过程改成存储对象主动调用业务对象。

当代码分成独立模块时，NameStore 可以放在业务模块或独立接口模块中。业务模块通过协议使用能力，具体存储模块依赖协议声明。如果协议仍放在具体存储模块里，业务为了使用协议而必须依赖整个存储模块，那么模块级依赖方向仍未反转。

只要协议的行为约定保持稳定，业务代码就可以使用另一种符合约定的实现，无须了解文件或数据库的内部细节。该原则减少具体实现对业务的影响，但不消除对协议的依赖。

### 依赖反转与依赖注入的区别

依赖注入是由外部向对象提供所需依赖的方式。示例中的 init(store:) 属于构造器注入：ProfileModel 接收外部创建的存储对象，而不是自行创建存储对象。

依赖反转规定源码应该依赖什么，关注业务与实现之间的依赖方向；依赖注入规定所需对象由谁提供，关注对象的组装方式。示例同时采用二者：ProfileModel 依赖 NameStore 抽象，并通过初始化方法接收具体对象。

仅把初始化参数写成具体类型 FileNameStore 并从外部传入，也属于依赖注入，但 ProfileModel 仍然依赖 FileNameStore，因此不能仅凭从外部传入对象就认定实现了依赖反转。
