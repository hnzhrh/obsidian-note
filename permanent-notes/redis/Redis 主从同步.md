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
# 6 Redis 场景的拓扑结构有哪些？

| **拓扑类型** | **适用场景**      | **核心优势**  | **注意事项**     |
| -------- | ------------- | --------- | ------------ |
| 复制拓扑结构   | 简单备份、基础读写分离   | 配置简单、成本低  | 扩展性差，需配合哨兵容灾 |
| 星形拓扑结构   | 高并发读、多地域部署    | 高读吞吐、数据冗余 | 主节点同步压力大     |
| 树状拓扑结构   | 超大规模系统、分散同步压力 | 层级扩展、负载分散 | 延迟增加、维护复杂    |

## 6.1 复制拓扑结构
- **结构描述**：单个主节点（Master）对应一个从节点（Slave）。
- **特点**：
    - **简单易配置**：适合初次使用主从复制的场景。    
    - **数据冗余**：从节点作为主节点的完整副本，提供数据备份。    
    - **读写分离**：主节点处理写请求，从节点分担读请求。    
- **缺点**：
    - **扩展性有限**：单个从节点可能无法满足高并发读需求。    
    - **单点风险**：主节点或从节点宕机可能导致服务中断（需配合哨兵实现自动故障转移）。   
* **适用场景**：中小规模应用、数据备份、基础读写分离。
## 6.2 星形拓扑结构
- **结构描述**：单个主节点（Master）下挂多个从节点（Slaves）。
- **特点**：
    - **高读性能**：多个从节点分散读请求，显著提升读吞吐量。
    - **地理分布**：从节点可部署在不同区域，优化本地读访问。
    - **容灾备份**：多个副本提高数据安全性。
- **缺点**：
    - **主节点压力**：主节点需同步数据到所有从节点，可能成为带宽或性能瓶颈。
    - **故障转移复杂度**：主节点宕机时需协调多个从节点切换（依赖哨兵或集群方案）。
- **适用场景**：高并发读场景（如热点数据读取）、多地域部署。
## 6.3 树状拓扑结构
- **结构描述**：主节点（Master）连接一级从节点（Slave1），一级从节点作为二级主节点，再连接更多从节点（Slave2-1、Slave2-2…），形成树状层级。
- **特点**：
    - **减轻主节点负载**：主节点仅需同步数据到一级从节点，后续层级由次级节点分发。
    - **灵活扩展**：支持大规模从节点部署，层级间带宽压力分散。
- **缺点**：
    - **数据延迟增加**：层级越深，底层从节点数据同步延迟可能越高。
    - **维护复杂度**：需管理多级节点关系，故障排查难度提升。
- **适用场景**：超大规模读场景、需要分散主节点同步压力的环境。
# 7 [[Redis 主从数据不一致该怎么办？]]
# 8 [[Redis 如何避免主从复制风暴？]]
# 9 Redis 节点 `REPLICAOF` 自己会怎么样？
# 10 [[Redis 有哪些缓冲区？各自有什么用？]]
# 11 [[Redis 的 run id 和 replication id]]

相关配置
# 12 Reference
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
