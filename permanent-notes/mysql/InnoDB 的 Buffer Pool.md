---
title: InnoDB 的 Buffer Pool
tags:
  - permanent-note
  - middleware/database/mysql
date: 2025-03-16
time: 09:19
aliases: 
done: false
---
# 1 概述

MySQL 的数据都是在磁盘中的，更新数据时，需要从磁盘 `load` 到内存中，修改后刷盘，MySQL 引入 `buffer pool` 机制，将数据缓存起来，如果正好有读取命令则可以直接从内存中读取了，提高了读写性能。

# 2 什么是 `buffer pool`

MySQL 启动时，根据配置 `innodb_buffer_pool_size` 申请一段连续内存空间，按照 16 KB 划分为缓存页，每一个缓存页都有一个对应的控制块，也就是元信息，记录该页的表空间、页号、地址等。

为了高效操作 `buffer pool` 使用链表管理控制块：
* `free` 链表，记录空闲页
* `flush` 链表，修改了缓存页后变成脏页，需要刷盘，脏页加入 `flush` 链表等待操作系统调度刷盘
* `LRU` 链表，增大缓存命中率
	* 冷热数据分区，配置 `innodb_old_blocks_pct` 决定冷数据的比例
	* 热数据区域性能优化，避免每次访问都把数据插到 `LRU` 头部，只有位于热数据区域后四分之一的数据才头插，减少性能损耗


# 3 脏页刷盘

定期刷盘：
* `BUF_FLUSH_LRU`： `LRU` 冷数据一部分刷盘
* `BUF_FLUSH_LIST`：`flush` 链表一部分刷盘

# 4 配置
## 4.1 配置多个 `buffer pool` 实例

可以配置多个 `buffer pool` 实例来提高性能
* 单个 `buffer pool` 过大
* 并发度高，使用多个 `buffer pool` 可降低冲突

设置 `buffer pool` 的实例个数：
```mysql
-- buffer pool size 小于 1G 时无效
innodb_buffer_pool_instances = 2
```

每个 `buffer pool` 的大小：
$$
innodb\_buffer\_pool\_size \div innodb\_buffer\_pool\_instances
$$
## 4.2 配置 `buffer pool` 大小

可以配置 `innodb_buffer_pool_size` 调整大小，如果是运行时调整，重新申请大内存可能会失败，MySQL 提供了 `innodb_buffer_pool_chunk_size` 配置，一个 `buffer pool` 包含多个 `chunk`，`chunk` 大小只能启动时指定，无法运行时修改。

注意点：**`innodb_buffer_pool_size` 必须是 `innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances` 的倍数（这主要是想保证每一个 `Buffer Pool` 实例中包含的 `chunk` 数量相同）**

# 5 查看 `buffer pool metrics`

```mysql
SHOW ENGINE INNODB STATUS
```

# 6 Reference
* [MySQL :: MySQL 8.4 Reference Manual :: 17.5.1 Buffer Pool](https://dev.mysql.com/doc/refman/8.4/en/innodb-buffer-pool.html)
* [MySQL 日志：undo log、redo log、binlog 有什么用？ \| 小林coding](https://xiaolincoding.com/mysql/log/how_update.html#%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81-buffer-pool)
* [MySQL 是怎样运行的：从根儿上理解 MySQL - 小孩子4919 - 掘金小册](https://juejin.cn/book/6844733769996304392/section/6844733770063429646)