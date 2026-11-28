# LRU 缓存：哈希表与双向链表

[返回数据结构目录](README.md) · [返回笔记目录](../README.md)

## 本文目录

- [LRU 的淘汰规则与操作要求](#lru-的淘汰规则与操作要求)
- [哈希表与双向链表的分工](#哈希表与双向链表的分工)
- [双向链表、节点中的键与哨兵节点](#双向链表节点中的键与哨兵节点)
- [Swift 实现](#swift-实现)
- [执行示例：容量为 2](#执行示例容量为-2)

## LRU 的淘汰规则与操作要求

LRU（Least Recently Used，最近最少使用）缓存是一种容量有限的键值缓存。缓存满了还要放入新数据时，LRU 缓存淘汰最久没有被访问的那条数据。读取和写入都算一次访问。

LRU 缓存提供两个操作，通常要求两个操作的时间都是 O(1)：

- `get(key)`：返回 `key` 对应的值，并把这条数据记为最近访问；`key` 不存在时返回空。
- `put(key, value)`：写入或更新 `key` 的值，并把这条数据记为最近访问；数据条数超过容量时，淘汰最久没有被访问的数据。

## 哈希表与双向链表的分工

LRU 缓存用哈希表和双向链表两个结构配合实现。哈希表负责按键找到数据，双向链表负责记录访问顺序：

| 结构 | 负责的事 | 为什么能做到 O(1) |
| --- | --- | --- |
| 哈希表 `[Key: Node]` | 按 `key` 找到对应的链表节点 | 用键的哈希值直接定位桶，不需要遍历，期望时间为 O(1)，成立条件见[HashMap 的结构与实现原理](hash-map.md#时间复杂度与实现边界) |
| 双向链表 | 按访问时间排列节点：头部是最久没有访问的节点，尾部是最近访问的节点 | 拿到节点后，只修改前后几个指针，就能完成摘除、移到尾部、淘汰头部 |

只用其中一个结构，总有一个操作退化为 O(n)：

- 只用哈希表：哈希表不记录访问顺序，找出最久没有访问的数据要遍历全部条目。
- 只用链表：按 `key` 查找要从头部逐个比较。

`get` 和 `put` 都同时用到两个结构。两个结构按“查找”和“维护顺序”分工，`get` 虽然是读取操作，也要把节点移到链表尾部。

## 双向链表、节点中的键与哨兵节点

**双向链表让摘除节点成为 O(1)。** 把一个节点从链表中间摘下，要让它的前一个节点直接指向它的后一个节点，所以必须先拿到前一个节点。

- 单向链表的节点只保存 `next`。找前一个节点要从头遍历，耗时 O(n)。
- 双向链表的节点同时保存 `prev` 和 `next`。摘除节点只需修改前后两个节点的指针，耗时 O(1)。

**节点中保存键。** 淘汰头部节点时，还要从哈希表中删除这条数据。节点保存了 `key`，程序才能用 `key` 删除哈希表中的条目。

**哨兵节点。** 哨兵节点是不存数据、只用于占位的节点。链表两端各放一个哨兵节点 `head` 和 `tail`，真实节点都位于两者之间。插入和摘除时，每个真实节点的前后一定有节点，程序不需要判断“链表是否为空”“节点是否为第一个”。

## Swift 实现

`Node` 是类，节点之间互相引用。`prev` 声明为 `weak`，相邻节点之间就只有 `next` 方向的强引用，不会形成引用环，见[weak 与 unowned 的使用条件](../language/swift-memory-and-references.md#weak-与-unowned-的使用条件)。

```swift
final class LRUCache {
    private final class Node {
        let key: Int
        var value: Int
        weak var prev: Node?
        var next: Node?
        init(key: Int, value: Int) { self.key = key; self.value = value }
    }

    private let capacity: Int
    private var map: [Int: Node] = [:]
    private let head = Node(key: 0, value: 0)   // 哨兵：head.next 是最久没有访问的节点
    private let tail = Node(key: 0, value: 0)   // 哨兵：tail.prev 是最近访问的节点

    init(_ capacity: Int) {
        self.capacity = capacity
        head.next = tail
        tail.prev = head
    }

    func get(_ key: Int) -> Int? {
        guard let node = map[key] else { return nil }   // 哈希表定位节点
        remove(node)                                     // 链表调整顺序
        appendToTail(node)
        return node.value
    }

    func put(_ key: Int, _ value: Int) {
        if let node = map[key] {                         // 键已存在：更新值并移到尾部
            node.value = value
            remove(node)
            appendToTail(node)
            return
        }
        let node = Node(key: key, value: value)
        map[key] = node
        appendToTail(node)
        if map.count > capacity, let oldest = head.next {
            remove(oldest)                               // 淘汰头部节点
            map[oldest.key] = nil                        // 用节点中保存的键删除哈希表条目
        }
    }

    private func remove(_ node: Node) {
        node.prev?.next = node.next
        node.next?.prev = node.prev
    }

    private func appendToTail(_ node: Node) {
        node.prev = tail.prev
        node.next = tail
        tail.prev?.next = node
        tail.prev = node
    }
}
```

## 执行示例：容量为 2

创建 `LRUCache(2)` 后依次执行以下操作，链表一列只列出 `head` 与 `tail` 之间的真实节点：

| 操作 | 链表（头 → 尾） | 结果 |
| --- | --- | --- |
| `put(1, 10)` | 1 | 写入键 1 |
| `put(2, 20)` | 1 → 2 | 写入键 2 |
| `get(1)` | 2 → 1 | 返回 10，节点 1 移到尾部 |
| `put(3, 30)` | 1 → 3 | 条数为 3，超过容量，淘汰头部的节点 2 |
| `get(2)` | 1 → 3 | 返回 nil |
| `put(1, 11)` | 3 → 1 | 键 1 已存在，值更新为 11，节点 1 移到尾部 |
