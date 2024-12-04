---
title: MySQL-自增主键回溯现象
tags:
  - fleet-note
  - middleware/database/mysql/datatype
date: 2024-12-03
time: 21:53
aliases:
---

```sql
mysql> SELECT * FROM t;

+---+

| a |

+---+

| 1 |

| 2 |

| 3 |

+---+

3 rows in set (0.01 sec)

mysql> DELETE FROM t WHERE a = 3;

Query OK, 1 row affected (0.02 sec)

mysql> SHOW CREATE TABLE t\G

*************************** 1. row ***************************

       Table: t

Create Table: CREATE TABLE `t` (

  `a` int NOT NULL AUTO_INCREMENT,

  PRIMARY KEY (`a`)

) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci

1 row in set (0.00 sec
```

在删除自增为 3 的这条记录后，下一个自增值依然为 4（AUTO_INCREMENT=4），这里并没有错误，自增并不会进行回溯。但若这时数据库发生重启，那数据库启动后，表 t 的自增起始值将再次变为 3，即自增值发生回溯。具体如下所示：

```sql
mysql> SHOW CREATE TABLE t\G

*************************** 1. row ***************************

       Table: t

Create Table: CREATE TABLE `t` (

  `a` int NOT NULL AUTO_INCREMENT,

  PRIMARY KEY (`a`)

) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci

1 row in set (0.00 s
```

若要彻底解决这个问题，有以下 2 种方法：

- 升级 MySQL 版本到 8.0 版本，每张表的自增值会持久化；
- 若无法升级数据库版本，则强烈不推荐在核心业务表中使用自增数据类型做主键。

其实，在海量互联网架构设计过程中，为了之后更好的分布式架构扩展性，不建议使用整型类型做主键，更为推荐的是**字符串类型**（这部分内容将在 05 节中详细介绍）。
# Reference
* [01 数字类型：避免自增踩坑](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/MySQL%e5%ae%9e%e6%88%98%e5%ae%9d%e5%85%b8/01%20%20%e6%95%b0%e5%ad%97%e7%b1%bb%e5%9e%8b%ef%bc%9a%e9%81%bf%e5%85%8d%e8%87%aa%e5%a2%9e%e8%b8%a9%e5%9d%91.md)