# Android 数据存储：私有目录、键值存储与数据库

[返回笔记目录](../README.md)

本文说明 Android App 的私有文件目录 `filesDir` 与 `cacheDir`、键值存储 SharedPreferences 与 DataStore，以及数据库库 Room，并与 iOS 的存储方式对照。

## 存储方式与 iOS 对照

| 需求 | Android | iOS 中相近的方式 |
| --- | --- | --- |
| 长期保存的文件 | `context.filesDir` | Documents、Library/Application Support |
| 可以重新生成的缓存文件 | `context.cacheDir` | Library/Caches |
| 少量键值，例如“是否开启深色模式” | SharedPreferences、DataStore | UserDefaults |
| 结构化数据 | Room（封装 SQLite） | Core Data、SwiftData、SQLite |

iOS 各目录的规则见[iOS 文件存储](ios-file-storage.md)，iOS 的持久化方式见[iOS 数据持久化](ios-data-persistence.md)。

## filesDir 与 cacheDir

内部存储是系统为每个 App 单独划出的存储区域，只有这个 App 能读写，读写时不需要申请权限。`filesDir` 和 `cacheDir` 都位于内部存储中。以包名为 `com.example.notes` 的 App 为例：

| 属性 | 返回的目录 | 用途 |
| --- | --- | --- |
| `context.filesDir` | `/data/user/0/com.example.notes/files` | 必须长期保留的文件，例如用户的草稿 |
| `context.cacheDir` | `/data/user/0/com.example.notes/cache` | 丢失后可以重新下载或生成的文件，例如下载的封面图 |

两个属性都返回指向目录的 `File` 对象，目录不存在时系统自动创建。代码通过属性获取目录，不在代码中写死完整路径。

### 目录中的文件何时被删除

| 删除时机 | `filesDir` | `cacheDir` |
| --- | --- | --- |
| 用户卸载 App | 删除 | 删除 |
| 用户在系统设置的应用信息页点击“清除数据”（部分系统版本叫“清除存储空间”） | 删除 | 删除 |
| 用户在应用信息页点击“清除缓存” | 保留 | 删除 |
| 设备存储空间不足 | 保留 | 系统可能自动删除 |

`filesDir` 中的文件，除非 App 自己删除，会一直保留到用户卸载 App 或清除数据。`cacheDir` 中的文件随时可能被删除，App 读取缓存前要检查文件是否存在，不存在时重新下载或生成。

`cacheDir` 由系统和用户回收，App 自己也要控制缓存大小，例如定期删除旧的缓存图片。缓存占用很大时，用户会在应用信息页看到很大的缓存数字。

Android 6.0（API level 23）起，系统的自动备份默认包含 `filesDir` 中的文件，不包含 `cacheDir` 中的文件。这条规则与 iOS 的 Documents 参与备份、Caches 不参与备份一致。

### 用法示例

```kotlin
// 草稿丢失后用户无法找回，保存到 filesDir
val draftFile = File(context.filesDir, "draft.txt")
draftFile.writeText("今天的笔记")

// 封面图丢失后可以重新下载，保存到 cacheDir
val coverFile = File(context.cacheDir, "cover_42.jpg")
if (!coverFile.exists()) {
    // 缓存已被删除，重新下载封面图并写入 coverFile
}
```

## SharedPreferences 与 DataStore

SharedPreferences 是 Android 系统提供的键值存储，把键值对保存为 App 私有目录中的 XML 文件，作用接近 iOS 的 UserDefaults。

```kotlin
val prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE)
prefs.edit().putBoolean("dark_mode", true).apply()   // apply() 立即修改内存中的值，再在后台写入磁盘
val darkMode = prefs.getBoolean("dark_mode", false)  // 从未保存过时返回默认值 false
```

`commit()` 是 `apply()` 的同步版本：`commit()` 在调用线程上写完磁盘才返回，并返回是否写入成功。在主线程调用 `commit()` 会阻塞主线程。

DataStore 是 Jetpack 提供的键值存储，用来替代 SharedPreferences。DataStore 通过协程和 Flow 异步读写数据，读写都不在主线程执行磁盘操作。

## Room

Room 是 Jetpack 提供的数据库库，在 SQLite 之上用注解声明表结构和查询：

- `@Entity` 标注的类对应一张表，类的属性对应表的列。
- `@Dao` 标注的接口声明增删改查方法，方法上的 `@Query` 写 SQL 语句。
- `@Database` 标注的类声明数据库包含哪些表。

Room 在编译时检查 `@Query` 中的 SQL 语句，表名或列名写错时直接编译报错。数据库的表设计规则见[数据库范式](database-normal-forms.md)，SQLite 的建表和事务见[iOS 数据持久化](ios-data-persistence.md)。
