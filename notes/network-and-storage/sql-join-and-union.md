# SQL 多表查询：JOIN 与 UNION

[返回笔记目录](../README.md)

## 目录

- [示例数据](#示例数据)
- [JOIN 与内连接](#join-与内连接)
- [表别名与列别名](#表别名与列别名)
- [外连接：LEFT JOIN 与 RIGHT JOIN](#外连接left-join-与-right-join)
- [UNION 与 UNION ALL](#union-与-union-all)
- [全外连接的替代写法](#全外连接的替代写法)
- [ON 条件与 WHERE 条件](#on-条件与-where-条件)
- [SQL 子句的逻辑执行顺序](#sql-子句的逻辑执行顺序)
- [JOIN 的执行方式与驱动表](#join-的执行方式与驱动表)

本文以 MySQL 为例，介绍两种合并数据的方式：

- JOIN 按关联条件把两张表的行左右拼接成一行，结果的列数增加。
- UNION 把多条 SELECT 的结果行上下拼接，结果的列数不变，行数增加。

## 示例数据

`user` 表：

| id | name |
| --- | --- |
| 1 | 张三 |
| 2 | 李四 |
| 3 | 王五 |

`orders` 表。`orders.user_id` 记录订单属于哪个用户：

| id | user_id | amount |
| --- | --- | --- |
| 101 | 1 | 50 |
| 102 | 1 | 80 |
| 103 | 2 | 30 |
| 104 | 4 | 60 |

王五（id = 3）没有订单。订单 104 的 `user_id` 是 4，`user` 表中没有 id = 4 的用户。下文用这两行说明不同连接方式对“配不上的行”的处理。

## JOIN 与内连接

```sql
SELECT user.id, user.name, orders.id AS order_id, orders.amount
FROM user JOIN orders ON user.id = orders.user_id;
```

| 片段 | 含义 |
| --- | --- |
| `user`、`orders` | 参与连接的两张表 |
| `JOIN` | 连接两张表。单独写 `JOIN` 等同于 `INNER JOIN` |
| `ON ...` | 配对条件。MySQL 把满足条件的两行配成一对 |
| `user.id` | `表名.列名` 的写法，表示 `user` 表的 `id` 列。两张表都有 `id` 列，不写表名时 MySQL 无法判断指的是哪一列，会报错 |
| `orders.id AS order_id` | `AS` 给结果列起别名，避免结果中出现两个都叫 `id` 的列 |

内连接（INNER JOIN）只保留满足 `ON` 条件、两边都配对成功的行组合。左右两张表中配不上的行都不会出现在结果中。

```text
张三（id=1）→ 在 orders 中找 user_id = 1 的行 → 找到 101、102 → 产生 2 行
李四（id=2）→ 在 orders 中找 user_id = 2 的行 → 找到 103     → 产生 1 行
王五（id=3）→ 在 orders 中找 user_id = 3 的行 → 没有找到     → 不产生结果
```

| id | name | order_id | amount |
| --- | --- | --- | --- |
| 1 | 张三 | 101 | 50 |
| 1 | 张三 | 102 | 80 |
| 2 | 李四 | 103 | 30 |

- 一个用户有几张订单，结果中就有几行。张三有 2 张订单，张三的 `id` 和 `name` 出现了 2 次。
- 王五没有订单，不出现。订单 104 找不到 id = 4 的用户，也不出现。

## 表别名与列别名

给表起别名时，`AS` 关键字可以省略。下面两种写法完全等价，`u` 是 `user` 的别名，`o` 是 `orders` 的别名：

```sql
FROM user AS u JOIN orders AS o ON u.id = o.user_id
FROM user u    JOIN orders o    ON u.id = o.user_id
```

给列起别名时，`AS` 同样可以省略：`o.id AS order_id` 可以写成 `o.id order_id`。常见的写法是表别名省略 `AS`，列别名保留 `AS`，便于区分列名和别名。

表起了别名后，同一条 SQL 中要用别名引用这张表的列。写成 `FROM user u` 后，再在 `SELECT` 中写 `user.id`，MySQL 会报错 `Unknown column 'user.id'`。

## 外连接：LEFT JOIN 与 RIGHT JOIN

写在 `JOIN` 左边的表叫左表，写在右边的表叫右表。在 `FROM user u JOIN orders o` 中，`user` 是左表，`orders` 是右表。

外连接除了保留配对成功的行，还保留某一边配不上的行，并把另一边的列填成 `NULL`。

LEFT JOIN（左连接）保留左表的全部行：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id;
```

| id | name | order_id | amount |
| --- | --- | --- | --- |
| 1 | 张三 | 101 | 50 |
| 1 | 张三 | 102 | 80 |
| 2 | 李四 | 103 | 30 |
| 3 | 王五 | NULL | NULL |

王五被保留，订单列为 `NULL`。订单 104 属于右表，配不上时不保留。

RIGHT JOIN（右连接）保留右表的全部行：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u RIGHT JOIN orders o ON u.id = o.user_id;
```

| id | name | order_id | amount |
| --- | --- | --- | --- |
| 1 | 张三 | 101 | 50 |
| 1 | 张三 | 102 | 80 |
| 2 | 李四 | 103 | 30 |
| NULL | NULL | 104 | 60 |

订单 104 被保留，用户列为 `NULL`。王五属于左表，配不上时不保留。

`A RIGHT JOIN B` 与 `B LEFT JOIN A` 的结果相同，只是两张表交换了位置。上面的 RIGHT JOIN 可以改写为：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM orders o LEFT JOIN user u ON u.id = o.user_id;
```

所以多数 SQL 统一使用 LEFT JOIN，把需要全部保留的表写在左边。

## UNION 与 UNION ALL

UNION 是合并查询结果的运算符：MySQL 分别执行两条 SELECT，把第二条的结果行接在第一条的结果行后面，并去掉完全相同的重复行。UNION ALL 同样合并结果，但不去重。

```sql
SELECT id, user_id, amount FROM orders WHERE user_id = 1    -- 得到 101、102
UNION
SELECT id, user_id, amount FROM orders WHERE amount > 40;   -- 得到 101、102、104
```

```text
MySQL 执行第一条 SELECT → (101,1,50)、(102,1,80)
→ MySQL 执行第二条 SELECT → (101,1,50)、(102,1,80)、(104,4,60)
→ MySQL 把两组结果接在一起，共 5 行
→ UNION 去掉完全相同的行：(101,1,50) 和 (102,1,80) 各出现两次，各保留一次
→ 最终结果：101、102、104
```

把 `UNION` 换成 `UNION ALL`，MySQL 不去重，最终结果为 5 行：101、102、101、102、104。

语法规则：

- 每条 SELECT 查询的列数必须相同，对应位置的列类型要兼容。
- 结果集的列名取自第一条 SELECT。
- 写在最后的 `ORDER BY` 和 `LIMIT` 作用于合并后的整个结果。只想对其中一条 SELECT 排序或限制行数时，给这条 SELECT 加括号，把 `ORDER BY` 和 `LIMIT` 写在括号内。

| | UNION | UNION ALL |
| --- | --- | --- |
| 是否去重 | 去重 | 不去重 |
| 开销 | 较大 | 较小 |
| 适用场景 | 两部分结果可能重叠，且需要去重 | 两部分结果不会重叠，或者需要保留重复行 |

UNION 去重时，MySQL 把两部分结果写入临时表，再在临时表上去掉重复行。临时表是 MySQL 执行查询时在内存或磁盘上建立的表，用来存放中间结果，查询结束后自动删除。UNION ALL 不需要这一步。

没有 `ORDER BY` 时，UNION 和 UNION ALL 结果的行顺序都不保证。

### UNION 与 OR 条件

UNION 判断重复的标准是整行相同，即 SELECT 查询的所有列的值都相同。上例的每一行都包含主键 `id`，不同订单的行一定不同，所以 UNION 只去掉“同一张订单被两个条件都查到”的情况，结果与下面的 OR 查询相同：

```sql
SELECT id, user_id, amount FROM orders WHERE user_id = 1 OR amount > 40;
```

SELECT 不包含主键时，两者的结果可能不同：

```sql
-- UNION：两条 SELECT 分别得到 (1)、(1) 和 (1)、(1)、(4)，去重后只剩 2 行：1、4
SELECT user_id FROM orders WHERE user_id = 1
UNION
SELECT user_id FROM orders WHERE amount > 40;

-- OR：订单 101、102、104 满足条件，得到 3 行：1、1、4
SELECT user_id FROM orders WHERE user_id = 1 OR amount > 40;
```

把 OR 拆成 UNION 常用于让每条 SELECT 各自使用一个索引。这种改写只在 OR 两边的列都有索引时才能避免全表扫描；任意一边没有索引，那一条 SELECT 仍要扫描全表，见[索引失效的常见场景](database-index-and-btree.md#匹配的记录在索引中不连续)。

## 全外连接的替代写法

全外连接（FULL OUTER JOIN）同时保留左右两表配不上的行。MySQL 不支持 `FULL OUTER JOIN`，可以把 LEFT JOIN 和 RIGHT JOIN 的结果用 UNION 合并：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id
UNION
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u RIGHT JOIN orders o ON u.id = o.user_id;
```

两条 SELECT 都包含张三和李四的 3 行配对结果，UNION 去重后只保留一份。最终结果有 5 行：

| id | name | order_id | amount |
| --- | --- | --- | --- |
| 1 | 张三 | 101 | 50 |
| 1 | 张三 | 102 | 80 |
| 2 | 李四 | 103 | 30 |
| 3 | 王五 | NULL | NULL |
| NULL | NULL | 104 | 60 |

四种连接方式的对比：

| 连接方式 | 配对成功的行 | 左表配不上的行（王五） | 右表配不上的行（订单 104） |
| --- | --- | --- | --- |
| INNER JOIN | 保留 | 不保留 | 不保留 |
| LEFT JOIN | 保留 | 保留，右表列为 NULL | 不保留 |
| RIGHT JOIN | 保留 | 不保留 | 保留，左表列为 NULL |
| FULL OUTER JOIN（MySQL 不支持） | 保留 | 保留 | 保留 |

## ON 条件与 WHERE 条件

在外连接中，条件写在 `ON` 里还是写在 `WHERE` 里，结果不同：

- `ON` 中的条件只决定右表的哪些行能和左表的行配对，不删除左表的行。
- `WHERE` 中的条件在连接完成后过滤整行。

### 在 ON 中写右表的条件

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 40;
```

这条 SQL 的结果等价于先从 `orders` 中筛出 `amount > 40` 的订单，再做 LEFT JOIN：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN (SELECT * FROM orders WHERE amount > 40) o ON u.id = o.user_id;
```

MySQL 执行时不会先单独生成一张筛选后的 `orders` 表，而是在每个用户查找订单时检查 `amount > 40`。以 `user` 为驱动表（见[JOIN 的执行方式与驱动表](#join-的执行方式与驱动表)）、`orders.user_id` 上有索引为例：

```text
张三（id=1）→ 用索引找到 user_id = 1 的订单 101、102
           → 检查 amount > 40：50 成立，80 成立 → 输出 2 行
李四（id=2）→ 找到订单 103 → 检查 amount > 40：30 不成立，103 不算配上
           → 李四没有配上任何订单 → 输出 (2, 李四, NULL, NULL)
王五（id=3）→ 没有找到订单 → 输出 (3, 王五, NULL, NULL)
```

| id | name | order_id | amount |
| --- | --- | --- | --- |
| 1 | 张三 | 101 | 50 |
| 1 | 张三 | 102 | 80 |
| 2 | 李四 | NULL | NULL |
| 3 | 王五 | NULL | NULL |

对 LEFT JOIN 来说，“找到了订单但不满足 ON 条件”与“没有订单”的效果相同，都按配不上处理。`orders` 上建立联合索引 `(user_id, amount)` 时，MySQL 可以直接在索引中定位 `user_id = 1 AND amount > 40` 的记录，不必先查出 101、102 再逐行检查。

### 在 WHERE 中写右表的条件

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 40;
```

LEFT JOIN 先产生 4 行：张三 101、张三 102、李四 103、王五 NULL。`WHERE o.amount > 40` 再过滤这 4 行：李四的 30 不满足条件；王五的 `NULL > 40` 结果为 `NULL`，不为真。最终只剩张三的 2 行。

### 在 ON 中写左表的条件

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id AND u.name = '张三';
```

这条 SQL 不会只返回张三。李四的订单 103 满足 `u.id = o.user_id`，但李四不满足 `u.name = '张三'`，ON 的两个条件不能同时成立，所以订单 103 不算配上。李四配对失败，仍以订单列为 `NULL` 的形式出现在结果中。

| 写法 | 张三 | 李四 | 王五 |
| --- | --- | --- | --- |
| `ON u.id = o.user_id` | 101、102 两行 | `103, 30` | `NULL, NULL` |
| `ON u.id = o.user_id AND u.name = '张三'` | 101、102 两行 | `NULL, NULL` | `NULL, NULL` |
| `ON u.id = o.user_id` 加 `WHERE u.name = '张三'` | 101、102 两行 | 不出现 | 不出现 |

只想查询张三时，要把 `u.name = '张三'` 写在 `WHERE` 中。

## SQL 子句的逻辑执行顺序

逻辑执行顺序规定一条 SQL 的结果应该按什么顺序算出来，与子句的书写顺序不同：

```text
FROM 和 JOIN ... ON：按 ON 条件配对，外连接还会补回对应一侧未匹配的行，另一侧的列填 NULL
→ WHERE：筛选单行
→ GROUP BY：按指定值分组
→ HAVING：筛选分组
→ SELECT：计算每条结果要返回的值
→ ORDER BY：排列结果行
→ LIMIT：限制返回的行数
```

ON 条件与 WHERE 条件的区别来自这个顺序：ON 在配对阶段生效，配不上的左表行随后被 LEFT JOIN 补回；WHERE 在补回之后过滤，被过滤的行不再出现。

逻辑执行顺序不等于实际执行顺序。只要结果不变，优化器可以调整实际的执行步骤。例如 `WHERE u.name = '张三'` 只涉及左表，MySQL 在读取 `user` 时就过滤这个条件，不会等连接完成后再过滤。

### 从用户记录到年龄统计结果

`SELECT *` 返回查询涉及的所有可见列。例如，`SELECT * FROM user WHERE name LIKE '张%' AND age = 20` 返回符合条件的用户记录。把 `SELECT *` 改成 `SELECT name, age` 后，每条符合条件的记录仍对应一条结果，结果只包含 `name` 和 `age` 两列。

按年龄统计时，查询可以为每个保留的年龄组计算一条结果。

```sql
SELECT age, COUNT(*) AS total
FROM user
WHERE name LIKE '张%'
GROUP BY age
HAVING COUNT(*) >= 2
ORDER BY total DESC
LIMIT 2;
```

假设 `user` 表里有 11 条记录，每条记录代表一个人。其中李一是 20 岁，其余 10 条记录的姓名以“张”开头。

| 子句 | 这一步之后的数据 |
| --- | --- |
| `FROM user` | 11 条用户记录 |
| `WHERE name LIKE '张%'` | 去掉李一，留下 10 条用户记录 |
| `GROUP BY age` | 10 条记录分成 4 组。20、21、22、23 岁各有 4、2、1、3 条记录 |
| `HAVING COUNT(*) >= 2` | `COUNT(*)` 统计每组行数。22 岁组只有 1 条记录，整组被去掉 |
| `SELECT age, COUNT(*) AS total` | 每组生成一条统计结果。结果包含 `(20, 4)`、`(21, 2)`、`(23, 3)` |
| `ORDER BY total DESC` | `total` 是每组的行数。按 4、3、2 的顺序排成 `(20, 4) → (23, 3) → (21, 2)` |
| `LIMIT 2` | 取前两条统计结果 `(20, 4)` 和 `(23, 3)` |

20 岁的 4 条用户记录都参与 `COUNT(*)`，最终只生成一条统计结果 `(20, 4)`。因此 `LIMIT 2` 取到的是两个年龄组的统计结果。

## JOIN 的执行方式与驱动表

### 嵌套循环连接

嵌套循环连接是 MySQL 执行 JOIN 的基本方式：外层循环逐行读取一张表，对每一行到另一张表中查找能配对的行。外层循环读取的表叫驱动表，内层被查找的表叫被驱动表。

以 `user` 为驱动表时：

```text
MySQL 读取 user 的一行，例如 id = 1
→ MySQL 用 1 到 orders 中查找 user_id = 1 的行
→ MySQL 把找到的每一行与 user 的这一行拼成结果行
→ MySQL 读取 user 的下一行，重复上面的步骤
```

查找被驱动表的这一步决定了 JOIN 的速度：

- `orders.user_id` 上有索引时，每次查找都在索引中定位，读取的页很少。
- `orders.user_id` 上没有索引时，每次查找都要扫描整张 `orders`。`user` 有 1 万行时，`orders` 要被扫描 1 万次。

所以被驱动表的关联列需要建立索引，本例中需要建立索引的是 `orders.user_id`。

MySQL 8.0.18 起支持哈希连接（hash join）：先把一张表读入内存，按关联列建立哈希表，再逐行读取另一张表，到哈希表中查找配对的行。关联列没有索引时，哈希连接避免了对被驱动表的反复扫描。

### 驱动表的选择

内连接只输出配对成功的行，从哪一张表开始遍历，结果都相同。所以 INNER JOIN 的驱动表由优化器选择，不一定是写在前面的表。优化器通常选择经过 WHERE 过滤后行数较少的表作为驱动表，这种做法常称为“小表驱动大表”。

用嵌套循环执行 LEFT JOIN 时，左表必须作为驱动表：

- `user` 作为驱动表时，MySQL 读取王五后到 `orders` 中查找 `user_id = 3`，没有找到，当场输出 `(3, 王五, NULL, NULL)`。
- `orders` 作为驱动表时，外层循环只遍历 4 张订单。没有订单的 `user_id` 是 3，王五从不会被访问，MySQL 无法从这个循环得知还需要输出王五。

同理，RIGHT JOIN 由右表驱动。需要全部保留的表必须由外层循环遍历。

MySQL 8.0.20 起，LEFT JOIN 也可以使用哈希连接：MySQL 用右表建立哈希表，再逐行读取左表到哈希表中查找。遍历的仍然是左表，左表配不上的行照样能当场输出。

### 外连接被改写为内连接

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 40;
```

LEFT JOIN 补回的行中，右表的列全部是 `NULL`，`o.amount > 40` 对这些行的结果为 `NULL`，不为真，所以这些行都会被 WHERE 过滤掉。最终结果与 INNER JOIN 完全相同。

右表的列全部为 `NULL` 时结果不可能为真的条件，叫拒绝 NULL 的条件。WHERE 中出现右表的拒绝 NULL 条件时，优化器把 LEFT JOIN 改写为 INNER JOIN，再自由选择驱动表。可以用两步查看：

1. 执行 `EXPLAIN` 加上这条 SQL。`id` 相同的多行输出中，排在第一行的表是驱动表。
2. 紧接着执行 `SHOW WARNINGS`。输出中显示优化器改写后的 SQL，原来的 `left join` 变成了 `join`。

条件写在 ON 中时（`ON u.id = o.user_id AND o.amount > 40`），配不上的左表行仍要保留，优化器不会把 LEFT JOIN 改写为 INNER JOIN，左表仍是驱动表。
