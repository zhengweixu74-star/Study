# iOS 文件存储

[返回笔记目录](../README.md)

## Documents、Application Support、Caches 与 tmp

iOS 为不同用途的数据提供不同目录。选择目录时，需要区分数据是否必须长期保留、能否重新生成，以及是否由用户管理。

| 目录 | 用途 | 备份与保留规则 |
| --- | --- | --- |
| Documents | 用户创建或管理的文档 | 默认参与备份，适合需要长期保留的用户文件 |
| Library/Application Support | 应用内部需要长期保留的数据，例如内部配置和数据库文件 | 默认参与备份；适合不需要由用户直接管理的应用数据 |
| Library/Caches | 可以重新下载或重新生成的缓存文件 | 不参与备份，系统可能清理；应用必须能在文件缺失时重新获取数据或继续运行 |
| tmp | 短期中间文件，例如一次导出操作使用的临时数据 | 不参与备份，系统可能清理；不保证每次退出应用后立即清空 |

缓存是为了减少重复下载或计算而保留的可重建数据。不能把唯一一份重要数据放在 Caches 或 tmp。应用确认不再需要临时文件后，应主动删除，不应依赖系统立即清理。

Documents 和 Application Support 的默认备份行为不代表其中每个文件都应该备份。可以重新下载或生成的大文件应排除备份；应用可以通过 URLResourceValues.isExcludedFromBackup 设置文件是否排除备份。

## 文件访问与存储方式

应用应通过 FileManager 或 URL 提供的目录 API 获取路径，不应硬编码完整路径。FileManager 用于管理目录和文件，Data 等类型用于读写文件内容。

选择文件目录和选择持久化方式是两个问题。文件目录决定文件保存在哪里，以及系统如何备份或清理文件。UserDefaults、文件存储、Keychain、SQLite、Core Data 和 SwiftData 的机制及适用场景见[iOS 数据持久化](ios-data-persistence.md)。
