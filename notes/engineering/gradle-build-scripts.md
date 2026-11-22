# Gradle 构建脚本与 .kts 文件

[返回笔记目录](../README.md)

Gradle 是根据构建脚本管理依赖、编译和打包的构建工具。构建脚本是描述构建配置和构建逻辑的代码文件。`.kts` 是 Kotlin 脚本文件的扩展名，`build.gradle.kts` 表示用 Kotlin 编写的 Gradle 构建脚本。

## .kt 与 .kts 的区别

`.kt` 是普通 Kotlin 源文件，`.kts` 是 Kotlin 脚本文件。脚本可以把可执行语句直接写在文件顶层，不需要声明 `main()`。

| 文件 | 常见用途 | 是否进入 App |
| --- | --- | --- |
| App 模块中的 `MainActivity.kt` | 实现 App 功能 | 参与编译和打包 |
| `build.gradle.kts` | 配置构建过程 | 不作为 App 代码打包 |

Gradle 构建时先编译 `build.gradle.kts`，再执行编译结果，得到依赖和最低系统版本等配置。脚本同样需要编译，只是 Gradle 自动完成编译。

## Groovy 与 Kotlin 两种写法

Groovy 是运行在 Java 虚拟机上的编程语言。Gradle 同时支持 Groovy 和 Kotlin 构建脚本。

| 文件名 | 脚本语言 |
| --- | --- |
| `build.gradle` | Groovy |
| `build.gradle.kts` | Kotlin |

下面两条声明都添加同一个依赖。

```groovy
dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.1'
}
```

```kotlin
dependencies {
    implementation("androidx.appcompat:appcompat:1.6.1")
}
```

两种脚本的用途相同，语法不同。Kotlin 脚本具有静态类型检查，通常也能获得更准确的 IDE 代码补全。不能只修改扩展名来完成语言转换，脚本内容也需要改成对应语言的语法。
