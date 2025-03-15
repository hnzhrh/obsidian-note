---
title: MySQL MVCC 机制
tags:
  - permanent-note
  - middleware/database/mysql/transaction
date: 2025-03-15
time: 12:48
aliases: 
done: false
---
# 1 什么是 MVCC？

MVCC（Multi-version Concurrency Control ），多版本并发控制，为了提高 MySQL InnoDB 的并发性能。

数据库并发场景有三种：
* 读-读
* 读-写，有线程安全问题，根据事务隔离级别可能造成脏读、不可重复读和幻读
	* MVCC
	* 锁
* 写-写，有线程安全问题，可能存在更新丢失
	* 悲观锁
	* 乐观锁

MVCC 就是为了实现快照读-写冲突不加锁的一种机制。
# 2 MVCC 的核心机制

通过 Read View 判断当前事务对记录的可见性，不可见时需要在 `undo log` 多版本链中找对应版本的数据。

## 2.1 `undo log` 多版本存储

MySQL 为聚簇索引记录维护了几个隐藏列：
- `DB_TRX_ID`， 6 字节，每次每一个事务修改记录时记录事务 ID
- `DB_ROLL_PTR`，7 字节，版本 `undo log` 链指针
- `DB_ROW_ID`，可选

![image.png](https://images.hnzhrh.com/note/20250315154703814.png)


每次跟新记录时，都会链接一条 `undo log` 到 `DB_ROLL_PTR` 的链尾，每个版本的 `undo log` 还包含了生成该版本时的事务 ID，链首为最新版本的数据。
## 2.2 Read View 判断可见性

Read View 记录了一个[事务启动](https://blog.csdn.net/jkzyx123/article/details/131206646)（或第一次查询时）数据库的“快照”信息，用于判断其他事务的修改是否对当前事务可见。通过 Read View，MySQL 可以避免脏读、不可重复读等问题，同时减少锁的使用，提高并发性能。

在创建Read View 时需要维护以下字段（[mysql-server/storage/innobase/read/read0read.cc at 14ba93991ba6a58a39d2e675b276af56f1079174 · mysql/mysql-server · GitHub](https://github.com/mysql/mysql-server/blob/14ba93991ba6a58a39d2e675b276af56f1079174/storage/innobase/read/read0read.cc#L315)）：
* `m_ids`: 活跃事务 ID 列表（活跃事务指的是启动了但没有 `COMMIT` 的事务）
* `min_trx_id`: `m_ids` 最小值
* `max_trx_id`: 数据库生成的下一个事务 ID（当前全局最大事务 DI + 1）
* `creator_trx_id`: 创建该 Read View 的事务 ID

Read View 是可见性算法，当事务去访问记录的时候，会判断记录的事务 ID 和 Read View 中的字段，判断流程如下：
* 等于 `creator_trx_id`，可见
* 小于 `min_trx_id`，该记录是在创建 Read View 前的事务内变更的，可见
* 大于 `max_trx_id`，该记录是在创建 Read View 后启动的事务内变更的，不可见
* 在两者之间
	* 在 `m_ids` 中，说明变更该记录的事务还没提交，不可见
	* 不在 `m_ids` 中，说明变更该记录的事务已经提交，可见

当通过 Read View 判断了当前记录为不可见时，会根据记录的 `DB_ROLL_PTR` 指针指向的版本链上查找事务 ID小于当前 Read View `min_trx_id` 的第一条历史版本记录数据。

# 3 RC 和 RR 快照读的实现

RC 读已提交在每次读取数据时都会生成一个新的 Read View
RR 可重复读在事务启动时生成一个新的 Read View，整个事务期间保持使用这个 Read View


# 4 Reference
* [事务隔离级别是怎么实现的？ \| 小林coding](https://xiaolincoding.com/mysql/transaction/mvcc.html)
* [MySQL - MySQL InnoDB的MVCC实现机制 \| Java 全栈知识体系](https://pdai.tech/md/db/sql-mysql/sql-mysql-mvcc.html)
* [MySQL 是怎样运行的：从根儿上理解 MySQL - 小孩子4919 - 掘金小册](https://juejin.cn/book/6844733769996304392/section/6844733770071801870)
* [MySQL :: MySQL 8.4 Reference Manual :: 17.3 InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
* [MySQL 事务的启动方式和 Autocommit 配置\_mysql开启事务几种方式-CSDN博客](https://blog.csdn.net/jkzyx123/article/details/131206646)