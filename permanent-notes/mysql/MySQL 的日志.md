---
title: MySQL 的日志
tags:
  - permanent-note
  - middleware/database/mysql/log
date: 2025-03-13
time: 18:47
aliases: 
done: false
---

# 1 `undo log` 回滚日志

实现了事务的原子性，用于事务回滚和 [MySQL MVCC 机制](MySQL%20MVCC%20机制.md)。

`undo log` 是在事务执行前记录回滚信息的机制，用于 MySQL 出现异常情况退出、回滚等状态时的原子性保证。
* 插入，记录主键 ID，如果回滚则直接删除这条数据
* 删除，记录记录内容，回滚时重新插入
* 更新
	* 主键列更新，记录反向更新语句
	* 非主键列更新，先删除再插入

# 2 `redo log` 重做日志

[InnoDB 的 Buffer Pool](InnoDB%20的%20Buffer%20Pool.md) 基于内存，如果修改了数据，脏页还没有刷到磁盘就宕机了那数据就没有了，因此 MySQL 提供了 `redo log` 来保证**持久性**。

把数据在内存修改了之后写入 `redo log` 就认为成功，InnoDB 在合适的时候把 [InnoDB 的 Buffer Pool](InnoDB%20的%20Buffer%20Pool.md) 脏页刷入磁盘，这就是 WAL （Writing-Ahead Logging）技术。当系统宕机后，MySQL 通过 `redo log` 可以恢复没有刷入磁盘的脏页数据。

## 2.1 为什么写 `redo log` 不直接写数据到磁盘？

`redo log` 写入使用了追加操作，顺序写，写入数据需要先找到数据位置在写入，是随机写。顺序写性能大于随机写。

## 2.2 `redo log buffer`

`redo log buffer` 是 `redo log` 的缓存，也是用来平衡内存-磁盘速度的缓冲区，防止每次都进行 IO 交互。

`redo log buffer` 可以通过配置 `innodb_log_buffer_size` 指定。

`redo log buffer` 何时刷盘？
* MySQL 正常关闭
* `redo log buffer` 容量达到一半
* InnoDB 后台线程每隔 1 s 刷盘
* 每次事务提交时都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘，由配置 `innodb_flush_log_at_trx_commit` 决定，默认为 1
	* `innodb_flush_log_at_trx_commit = 0`：事务提交时，只写入 `buffer` 不会刷盘
	* `innodb_flush_log_at_trx_commit = 1`：事务提交时刷盘
	* `innodb_flush_log_at_trx_commit = 2` ：事务提交时，写入文件但不刷盘（何时刷盘由操作系统调度）
## 2.3 `redo log` 文件

MySQL 数据目录下（`SHOW VARIABLES LIKE 'datadir'`）下默认有两个名为 `ib_logfile0` 和 `ib_logfile1` 的文件，`log buffer` 中的日志默认情况下就是刷新到这两个磁盘文件中。
- `innodb_log_group_home_dir`：该参数指定了 `redo` 日志文件所在的目录，默认值就是当前的数据目录。
- `innodb_log_file_size`：该参数指定了每个 `redo` 日志文件的大小，在 `MySQL 5.7.21` 这个版本中的默认值为 `48MB`，
- `innodb_log_files_in_group`：该参数指定 `redo` 日志文件的个数，默认值为2，最大值为100。
`redo log` 是循环写，0 文件满了写 1,1 满了写 0。旧的 `redo log` 是可以清除的。如果并发高，导致文件被写满了，MySQL 就不会再执行更新操作了，需要对旧 `redo log` 擦除后才正常工作。

# 3 `bin log` 归档日志

`bin log` 是 MySQL server 层面的日志，记录了更新操作，事务提交后写入文件。

|      | `bin log`            | `redo log`    |
| ---- | -------------------- | ------------- |
| 用途   | Server 层生成，用于备份、主从同步 | 存储引擎生成，用于故障恢复 |
| 写入方式 | 追加写                  | 循环写，          |
| 文件格式 | 三种方式                 | 物理日志，某个数据页的修改 |

`bin log` 有三种格式：
* `STATEMENT`：记录 SQL，如果有函数操作，在从库执行可能导致数据不一致
* `RAW`：记录修改后的数据，修改数据过多可能很大，不如一条 SQL 节省空间
* `MIXED`：MySQL 分析 SQL，自动切换是使用 `STATEMENT` 还是 `RAW`。


# 4 References
* [MySQL 日志：undo log、redo log、binlog 有什么用？ \| 小林coding](https://xiaolincoding.com/mysql/log/how_update.html)
* [MySQL 是怎样运行的 undo log](https://juejin.cn/book/6844733769996304392/section/6844733770067607566)
* [MySQL 是怎样运行的 redo log](https://juejin.cn/book/6844733769996304392/section/6844733770063626253)