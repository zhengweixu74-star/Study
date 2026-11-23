# iOS 文件存储

[返回笔记目录](../README.md)

## Documents、Application Support、Caches 与 tmp

iOS 为不同用途的数据提供不同目录。选择目录时，要判断三点：数据是否必须长期保留、能否重新生成、是否由用户管理。

| 目录 | 用途 | 备份与保留规则 |
| --- | --- | --- |
| Documents | 用户创建或管理的文档 | 默认参与备份，适合需要长期保留的用户文件 |
| Library/Application Support | 应用内部需要长期保留的数据，例如内部配置和数据库文件 | 默认参与备份；适合不需要由用户直接管理的应用数据 |
| Library/Caches | 可以重新下载或重新生成的缓存文件 | 不参与备份，系统可能清理；文件被清理后，应用要能重新获取数据或继续运行 |
| tmp | 短期中间文件，例如一次导出操作使用的临时数据 | 不参与备份，系统可能清理；系统清理的时机不确定，退出应用时不一定清空 |

缓存是为了减少重复下载或计算而保留的数据，缓存丢失后可以重新生成。重要数据只有一份时，要放在 Documents 或 Application Support，不能放在 Caches 或 tmp。临时文件用完后，应用要主动删除，因为系统清理的时机不确定。

Documents 和 Application Support 默认参与备份，但其中可以重新下载或生成的大文件应排除备份。应用通过 URLResourceValues.isExcludedFromBackup 设置单个文件是否排除备份。

## 文件访问与存储方式

应用通过 FileManager 或 URL 提供的目录 API 获取路径，代码中不写死完整路径，因为应用沙箱的实际路径可能变化。FileManager 用于管理目录和文件，Data 等类型用于读写文件内容。

选择文件目录和选择持久化方式是两个独立的决定。文件目录决定文件保存在哪里，以及系统如何备份或清理文件；持久化方式决定数据用什么格式和接口读写。UserDefaults、文件存储、Keychain、SQLite、Core Data 和 SwiftData 的机制及适用场景见[iOS 数据持久化](ios-data-persistence.md)。
