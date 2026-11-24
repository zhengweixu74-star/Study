# Android 工程结构、依赖库与安装包

[返回 Android 分类目录](README.md)

本文说明 Android 工程的目录组成、系统版本与最低支持版本、AndroidX 与 Jetpack 依赖库、构建变体，以及安装包的内容和发布格式。

## 工程目录与 iOS 对照

模块（module）是一组可以单独编译的源码和资源，作用接近 Xcode 的 target。新建的 Android 工程通常只有一个 `app` 模块，`app` 模块最终被打包成安装包：

```text
MyNotes/
├── settings.gradle.kts          // 声明工程包含哪些模块
├── build.gradle.kts             // 整个工程的构建配置
└── app/                         // app 模块
    ├── build.gradle.kts         // app 模块的配置：依赖库、最低支持的系统版本
    └── src/main/
        ├── AndroidManifest.xml  // 清单文件
        ├── java/com/example/notes/
        │   └── MainActivity.kt  // 源码；目录名是 java，Kotlin 文件也放在这里
        └── res/                 // 资源：布局、文字、图片等
```

Gradle 是 Android 的构建工具。Gradle 读取 `build.gradle.kts` 中的配置，下载依赖库、编译源码，最后打出安装包。构建脚本的写法见[Gradle 构建脚本与 .kts 文件](../engineering/gradle-build-scripts.md)，`res/` 中资源的组织和引用见[Android 资源与 R 类](resources-and-r-class.md)。

| Android | iOS |
| --- | --- |
| Android Studio | Xcode |
| Gradle + `build.gradle.kts` | Xcode 构建系统 + Podfile／SPM |
| 模块（module） | target |
| `AndroidManifest.xml` | `Info.plist` |
| `res/` | Assets.xcassets、Localizable.strings、Storyboard |
| APK | .ipa |

清单文件 `AndroidManifest.xml` 向系统声明应用名称、图标、组件和权限。系统安装 App 时解析清单，启动组件时使用解析结果，见[AndroidManifest.xml 为什么要声明 Activity](activity-fragment-and-navigation.md#androidmanifestxml-为什么要声明-activity)。

## API level 与 minSdk

API level 是每个 Android 系统版本对应的整数编号，例如 Android 14 的 API level 是 34。

`minSdk` 是 `build.gradle.kts` 中的配置，表示 App 支持的最低 API level。`minSdk = 24` 时，App 只能安装在 API level 24 及以上的设备上。

Android 设备的系统版本分布很广，一个 App 往往要同时支持多个大版本的系统。代码调用高版本才有的 API 时，先判断当前系统版本：

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    // 只在 API level 33 及以上执行
}
```

`Build.VERSION.SDK_INT` 是当前设备系统的 API level。iOS 中相近的写法是 `if #available(iOS 17, *)`。

## AndroidX、Jetpack 与兼容库

AndroidX 是 Google 发布的一组 Android 扩展库，包名以 `androidx` 开头。AndroidX 库随 App 一起打包进安装包，所以同一份库代码可以运行在不同版本的系统上。

兼容库是 AndroidX 中让新功能在旧系统上也能使用的库。例如 `AppCompatActivity` 是 `androidx.appcompat` 库提供的 Activity 子类：App 继承 `AppCompatActivity` 后，新版本系统的界面特性在旧版本系统上也能表现一致。

Jetpack 是 Google 对 AndroidX 中各类库的统称。常用的 Jetpack 库：

| 库 | 作用 |
| --- | --- |
| Lifecycle | 让普通对象监听 Activity 和 Fragment 的生命周期变化 |
| ViewModel | 在配置变化引起的 Activity 重建期间保存页面状态，见[Android 页面生命周期与状态保存](activity-lifecycle-and-state.md) |
| LiveData | 感知生命周期的可观察数据，见[LiveData 与 StateFlow](stateflow-and-lifecycle-collection.md#livedata-与-stateflow) |
| Room | 在 SQLite 上封装的数据库库，见[Android 数据存储](../network-and-storage/android-data-storage.md) |
| DataStore | 键值存储和类型化数据存储，用来替代 SharedPreferences |
| WorkManager | 执行可以延后、但必须完成的后台任务，见[Android 应用组件、Context 与权限](app-components-and-context.md) |
| Hilt | 依赖注入库 |

依赖注入是对象不自己创建所依赖的对象、改由外部创建后传入的做法，见[设计模式与 Swift 实践](../engineering/design-patterns.md)。Android 工程常用的依赖注入库有 Dagger、Hilt 和 Koin。

## 构建变体：buildType 与 productFlavor

构建变体（build variant）是一次构建产出的具体版本，由 buildType 和 productFlavor 组合而成：

- buildType 是构建方式，通常有 `debug` 和 `release` 两种。`debug` 包可以调试，`release` 包用于发布。buildType 的作用接近 Xcode 的 Configuration。
- productFlavor 是同一份代码打出的不同产品版本，例如国内版和海外版。每个 flavor 可以有自己的源码目录、资源和配置。

工程定义了 `cn` 和 `global` 两个 flavor 时，构建变体有 `cnDebug`、`cnRelease`、`globalDebug`、`globalRelease` 四种。构建 `cnDebug` 变体的 Gradle 任务是 `assembleCnDebug`。

## 安装包：APK、AAB 与 R8

APK 是 Android 的安装包文件，本质是一个 zip 压缩包：

| 内容 | 作用 |
| --- | --- |
| `classes.dex`（代码多时还有 `classes2.dex` 等） | App 代码编译出的 DEX 字节码 |
| `resources.arsc` | 资源表，记录资源 ID 与资源值、资源文件的对应关系 |
| `res/` | 布局、图片等资源文件 |
| `AndroidManifest.xml` | 编译后的清单文件 |
| `lib/` | C/C++ 代码编译出的 `.so` 动态库，按 CPU 架构分目录存放 |
| `assets/` | 按原始文件保存、不生成资源 ID 的文件 |
| `META-INF/` | 签名信息 |

DEX 的执行方式见[Android 系统分层与应用启动](system-architecture-and-app-launch.md)。

AAB（Android App Bundle）是上传到 Google Play 的发布格式。开发者上传 AAB，Google Play 再按每台设备的 CPU 架构、屏幕密度和语言生成只含所需文件的 APK，所以用户下载的安装包更小。

R8 是 Android 构建工具链中压缩和混淆代码的工具，旧工具名是 ProGuard。`release` 构建启用 R8 后，R8 执行三件事：

- 删除没有被调用的类和方法。
- 把类名、方法名改成短名称，这一步叫混淆。混淆让安装包更小，也让反编译后的代码难以阅读。
- 优化代码，例如内联短方法。

反射是程序运行时按名称查找并调用类和方法的机制；JNI 是 Java／Kotlin 代码与 C/C++ 代码互相调用的接口。通过反射或 JNI 按名称访问的类，R8 无法从代码调用关系中看出来。开发者要在 ProGuard 规则文件中声明保留这些类，否则 R8 会删除或改名这些类，运行时找不到对应的类。
