# iOS 文件存储

[返回笔记目录](../README.md)

## Documents、Caches 与 tmp

Documents 用于用户文档，默认参与备份。Caches 用于可重建缓存，tmp 用于短期中间文件，后两者默认不备份。Caches 和 tmp 的内容都可能丢失，tmp 不保证每次退出立即清空。
