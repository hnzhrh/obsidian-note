---
title: MySQL-数字类型
tags:
  - fleet-note
  - middleware/database/mysql/datatype
date: 2024-12-03
time: 11:49
aliases:
---
# 非必要勿刻意使用 unsigned
MySQL 非必要不需要刻意使用 unsigned 类型，在做数据分析时，如果两个 unsigned 数值相减出现了负数，则会报错：
```sql
BIGINT UNSIGNED value is out of range in
```

为了避免这个错误，需要对数据库参数 sql_mode 设置为 NO_UNSIGNED_SUBTRACTION，允许相减的结果为 signed.

# 自增主键
使用 BIGINT 做主键，而不是 INT，在 MySQL 8.0 以前，自增主键不会持久化，可能会出现回溯现象。
当插入数据达到最大 INT 上限后，会报重复错误，MySQL 不会将其重置为 1.

# Reference