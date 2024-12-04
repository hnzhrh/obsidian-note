---
title: MySQL-用户性别设计
tags:
  - fleet-note
  - middleware/database/mysql/datatype
date: 2024-12-04
time: 12:25
aliases:
---
* int 存储
* enum 存储
* 8.0 以后使用约束


设计表结构时，你会遇到一些固定选项值的字段。例如，性别字段（Sex），只有男或女；又或者状态字段（State），有效的值为运行、停止、重启等有限状态。

我观察后发现，大多数开发人员喜欢用 INT 的数字类型去存储性别字段，比如：

```sql
CREATE TABLE `User` (

  `id` bigint NOT NULL AUTO_INCREMENT,

  `sex` tinyint DEFAULT NULL,

  ......

  PRIMARY KEY (`id`)

) ENGINE=InnoDB；
```

其中，tinyint 列 sex 表示用户性别，但这样设计问题比较明显。

- **表达不清**：在具体存储时，0 表示女，还是 1 表示女呢？每个业务可能有不同的潜规则；
- **脏数据**：因为是 tinyint，因此除了 0 和 1，用户完全可以插入 2、3、4 这样的数值，最终表中存在无效数据的可能，后期再进行清理，代价就非常大了。

在 MySQL 8.0 版本之前，可以使用 ENUM 字符串枚举类型，只允许有限的定义值插入。如果将参数 SQL_MODE 设置为严格模式，插入非定义数据就会报错：

```sql
mysql> SHOW CREATE TABLE User\G

*************************** 1. row ***************************

       Table: User

Create Table: CREATE TABLE `User` (

  `id` bigint NOT NULL AUTO_INCREMENT,

  `sex` enum('M','F') COLLATE utf8mb4_general_ci DEFAULT NULL,

  PRIMARY KEY (`id`)

) ENGINE=InnoDB

1 row in set (0.00 sec)

mysql> SET sql_mode = 'STRICT_TRANS_TABLES';

Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> INSERT INTO User VALUES (NULL,'F');

Query OK, 1 row affected (0.08 sec)

mysql> INSERT INTO User VALUES (NULL,'A');

ERROR 1265 (01000): Data truncated for column 'sex' at row 1
```

由于类型 ENUM 并非 SQL 标准的数据类型，而是 MySQL 所独有的一种字符串类型。抛出的错误提示也并不直观，这样的实现总有一些遗憾，主要是因为MySQL 8.0 之前的版本并没有提供约束功能。自 MySQL 8.0.16 版本开始，数据库原生提供 CHECK 约束功能，可以方便地进行有限状态列类型的设计：

```sql
mysql> SHOW CREATE TABLE User\G

*************************** 1. row ***************************

       Table: User

Create Table: CREATE TABLE `User` (

  `id` bigint NOT NULL AUTO_INCREMENT,

  `sex` char(1) COLLATE utf8mb4_general_ci DEFAULT NULL,

  PRIMARY KEY (`id`),

  CONSTRAINT `user_chk_1` CHECK (((`sex` = _utf8mb4'M') or (`sex` = _utf8mb4'F')))

) ENGINE=InnoDB

1 row in set (0.00 sec)

mysql> INSERT INTO User VALUES (NULL,'M');

Query OK, 1 row affected (0.07 sec)

mysql> INSERT INTO User VALUES (NULL,'Z');

ERROR 3819 (HY000): Check constraint 'user_chk_1' is violated.
```

从这段代码中看到，第 8 行的约束定义 user_chk_1 表示列 sex 的取值范围，只能是 M 或者 F。同时，当 15 行插入非法数据 Z 时，你可以看到 MySQL 显式地抛出了违法约束的提示。

# Reference