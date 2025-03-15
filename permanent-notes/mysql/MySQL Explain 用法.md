---
title: MySQL Explain 用法
tags:
  - permanent-note
  - middleware/database/mysql/index
date: 2025-03-14
time: 19:16
aliases: 
done: false
---

# 1 **2. 输出字段解析**
`EXPLAIN` 输出包含以下字段：

## 1.1 id
- **作用**：标识查询中每个子查询或操作的执行顺序。
- **可能值**：
  - **相同数字**：按顺序执行。
  - **不同数字**：数字越大优先级越高（嵌套子查询）。
  - **NULL**：表示 UNION 操作的合并结果。

## 1.2 select_type
- **作用**：查询的类型。
- **常见值**：
  - **SIMPLE**：简单 SELECT（无子查询或 UNION）。
  - **PRIMARY**：最外层查询。
  - **SUBQUERY**：子查询中的第一个 SELECT。
  - **DERIVED**：派生表（FROM 子句中的子查询）。
  - **UNION**：UNION 中的第二个或后续查询。
  - **UNION RESULT**：UNION 的结果。
  - **DEPENDENT SUBQUERY**：依赖外层查询的子查询。
  - **MATERIALIZED**：物化子查询（如 `IN` 子查询被优化为临时表）。

## 1.3 table
- **作用**：当前操作涉及的表名。
- **可能值**：
  - 表名或别名。
  - `<derivedN>`：派生表（N 是 id）。
  - `<unionM,N>`：UNION 合并的结果。

## 1.4 partitions
- **作用**：查询涉及的分区（仅当表有分区时）。
- **可能值**：分区名或 `NULL`。

## 1.5 type
- **作用**：表的访问方式（性能关键字段，按效率从高到低排序）：
  - **system**：表只有一行（系统表）。
  - **const**：通过主键或唯一索引查找单行。
  - **eq_ref**：JOIN 时使用主键或唯一索引。
  - **ref**：使用非唯一索引查找。
  - **fulltext**：全文索引。
  - **ref_or_null**：类似 `ref`，但包含 NULL 值搜索。
  - **index_merge**：合并多个索引的结果。
  - **unique_subquery**：子查询使用唯一索引。
  - **index_subquery**：子查询使用非唯一索引。
  - **range**：索引范围扫描（如 `BETWEEN`、`IN`）。
  - **index**：全索引扫描（比全表扫描快）。
  - **ALL**：全表扫描（需优化）。

## 1.6 possible_keys
- **作用**：可能使用的索引。
- **可能值**：索引名或 `NULL`（无可用索引）。

## 1.7 key
- **作用**：实际使用的索引。
- **可能值**：索引名或 `NULL`（未使用索引）。

## 1.8 key_len
- **作用**：索引使用的字节数。
- **计算规则**：根据索引字段类型和长度计算（如 INT 为 4 字节）。

## 1.9 ref
- **作用**：显示索引如何被引用。
- **可能值**：
  - `const`：常量值（如 `WHERE id = 1`）。
  - 列名（如 `users.name`）。
  - `NULL`：无引用（如索引全扫描）。

## 1.10 rows
- **作用**：预估需要扫描的行数（越小越好）。

## 1.11 filtered
- **作用**：查询条件过滤后剩余行的百分比。
- **示例**：`filtered=10%` 表示仅保留 10% 的数据。

## 1.12 Extra
- **作用**：额外信息（优化关键字段）。
- **常见值**：
  - **Using index**：覆盖索引（无需回表）。
  - **Using where**：WHERE 条件过滤数据。
  - **Using temporary**：使用临时表（常见于 GROUP BY 或 ORDER BY）。
  - **Using filesort**：外部排序（需优化索引）。
  - **Using join buffer**：使用连接缓存（可调大 `join_buffer_size`）。
  - **Impossible WHERE**：WHERE 条件永远为假。
  - **Select tables optimized away**：优化器已提前计算结果（如 `MIN()` 使用索引）。

# 2 高级用法
- **`EXPLAIN ANALYZE`**（MySQL 8.0+）：实际执行查询并返回详细耗时。
- **`FORMAT=JSON`**：生成 JSON 格式的执行计划（包含成本估算）。


# 3 各种类型的示例

| id | select\_type | table | partitions | type | possible\_keys | key | key\_len | ref | rows | filtered | Extra |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | SIMPLE | A | null | index | null | PRIMARY | 4 | null | 21 | 100 | Using index |

# 4 Reference
* [MySQL :: MySQL 8.4 Reference Manual :: 10.8.2 EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.4/en/explain-output.html)
* [MySQL explain 应用详解(吐血整理🤩) - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000023565685)