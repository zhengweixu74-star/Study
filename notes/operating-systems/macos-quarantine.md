# macOS 隔离标记

[返回操作系统基础目录](README.md)

隔离标记是 macOS 给从网络下载的文件附加的扩展属性，名称为 `com.apple.quarantine`。扩展属性是文件正文之外的元数据。macOS 首次打开带有隔离标记的应用时，会触发安全检查。隔离标记只记录文件来自网络，应用是否安全由安全检查判断。

`xattr` 是 macOS 查看和修改扩展属性的命令。以下示例只处理指定的应用：

```bash
xattr -p com.apple.quarantine /Applications/Example.app
xattr -rd com.apple.quarantine /Applications/Example.app
```

第一条命令查看隔离标记；应用没有隔离标记时，`xattr -p` 报告找不到属性。第二条命令移除隔离标记：`-d` 表示删除指定属性，`-r` 表示递归处理应用包中的所有文件。移除隔离标记后，首次打开时的安全检查不再触发，所以只对来源可信的应用执行。
