---
title: Redis 主从同步
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-10
time: 10:16
aliases: 
done: false
---
# 1 [[Redis 主从同步流程和版本演变]]
# 2 [[Redis 主从复制配置]]
# 3 监控
## 3.1 Master 和 Replica 的同步进度监控

* `master_repl_offset`，Master节点 offset
* `slave_repl_offset`，Replica 节点 offset
* `slave_read_repl_offset`，Replica 节点从网络读取 offset

根据这三个值可以计算延迟：
* Replica 网络延迟：`master_repl_offset - slave_read_repl_offset`。    
- Repica 处理延迟：`slave_read_repl_offset - slave_repl_offset`。    
- **总延迟**：`master_repl_offset - slave_repl_offset`。
# 4 [[Redis Master 和 Replica 如何处理过期数据？]]
# 5 [[Redis 主从同步全量同步策略有哪些？各自有什么优劣势？]]
# 6 [[Redis 主从同步拓扑结构有哪些？]]
# 7 [[Redis 主从数据不一致该怎么办？]]
# 8 [[Redis 如何避免主从复制风暴？]]
# 9 [[Redis 有哪些缓冲区？各自有什么用？]]
# 10 [[Redis 的 run id 和 replication id]]

相关配置
# 11 Reference
* [Redis replication \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
* [06 \| 数据同步：主从库如何实现数据一致？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/272852)
* [32 \| Redis主从同步与故障切换，有哪些坑？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/303247)
* [33 \| 脑裂：一次奇怪的数据丢失-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/303568)
* [INFO \| Docs](https://redis.io/docs/latest/commands/info/)
* [redis主从复制策略的原理：主从节点间如何同步数据？\_redis主从复制时主节点新数据-CSDN博客](https://blog.csdn.net/virusos/article/details/130915836)
* [Redis进阶篇05 — 复制技术（二）Redis主从复制 – Rocky Linux](https://www.rockylinux.cn/notes/redis-advanced-chapter-05-replication-technology-ii.html)
* [Redis进阶篇06 — 复制技术（三）有关Redis复制的配置参数 – Rocky Linux](https://www.rockylinux.cn/notes/redis-advanced-chapter-06-replication-technology-3.html)
* [Redis进阶篇07 — 复制技术（四）replica 与 sub-replica – Rocky Linux](https://www.rockylinux.cn/notes/redis-advanced-chapter-07-copy-technology-4-copy-and-sub.html)
* [Redis进阶篇09 — 集群 – Rocky Linux](https://www.rockylinux.cn/notes/redis-advanced-edition-09-cluster.html)
