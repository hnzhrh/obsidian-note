---
title: MySQL-修改字符集
tags:
  - fleet-note
  - middleware/database/mysql/datatype
date: 2024-12-04
time: 12:24
aliases:
---
### 正确修改字符集

当然，相信不少业务在设计时没有考虑到字符集对于业务数据存储的影响，所以后期需要进行字符集转换，但很多同学会发现执行如下操作后，依然无法插入 emoji 这类 UTF8MB4 字符：

```sql
ALTER TABLE emoji_test CHARSET utf8mb4;
```

其实，上述修改只是将表的字符集修改为 UTF8MB4，下次新增列时，若不显式地指定字符集，新列的字符集会变更为 UTF8MB4，**但对于已经存在的列，其默认字符集并不做修改**，你可以通过命令 SHOW CREATE TABLE 确认：

```sql
mysql> SHOW CREATE TABLE emoji_test\G

*************************** 1. row ***************************

       Table: emoji_test

Create Table: CREATE TABLE `emoji_test` (

  `a` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,

  PRIMARY KEY (`a`)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

1 row in set (0.00 sec)
```

可以看到，列 a 的字符集依然是 UTF8，而不是 UTF8MB4。因此，正确修改列字符集的命令应该使用 ALTER TABLE … CONVERT TO…这样才能将之前的列 a 字符集从 UTF8 修改为 UTF8MB4：

```sql
mysql> ALTER TABLE emoji_test CONVERT TO CHARSET utf8mb4;

Query OK, 0 rows affected (0.94 sec)

Records: 0  Duplicates: 0  Warnings: 0

mysql> SHOW CREATE TABLE emoji_test\G

*************************** 1. row ***************************

       Table: emoji_test

Create Table: CREATE TABLE `emoji_test` (

  `a` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,

  PRIMARY KEY (`a`)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

1 row in set (0.00 sec)
```

讲到这儿，我们已经学完了字符串相关的基础知识，接下来就一起进行 MySQL 字符串表结构的设计实战，希望你能在设计中真正用好字符串类型。


# Reference