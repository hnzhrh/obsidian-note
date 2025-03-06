---
title: Redis 内存碎片问题
tags:
  - permanent-note
  - middleware/redis/eviction
date: 2025-03-06
time: 15:04
aliases:
---
# 1 内存碎片从何而来？

## 1.1 内存分配器的分配策略

Redis 使用 `jemalloc` 等内存分配器管理内存，分配器按固定大小（如 8B、16B、32B...）分配内存块。当数据大小与分配块不匹配时，会产生**内部碎片**（如申请 5B 数据，实际分配 8B）。

## 1.2 数据修改与删除

- **键值更新**：修改数据导致新旧值大小不同，旧内存释放后可能无法被新数据完全复用。
- **键过期/删除**：释放的内存块可能因不连续而无法被新数据利用，形成**外部碎片**。

## 1.3 持久化与复制

RDB/AOF 重写或主从复制时，Redis 会创建子进程。子进程与父进程共享内存，可能触发 Copy-on-Write 机制，导致内存页分裂，加剧碎片。

# 2 如何判断是否有内存碎片？

使用命令 `info memory` 获取状态：

- **`mem_fragmentation_ratio`**：碎片率（计算公式：`used_memory_rss / used_memory`）。
    - **等于1**：理想状态，无碎片。    
    - **大于1**：存在碎片（如 1.5 表示额外 50% 内存被浪费）。        
    - **小于1**：内存被交换到磁盘（需警惕性能问题）。        
- **`used_memory_rss`**：操作系统视角的 Redis 进程占用物理内存。
- **`used_memory`**：Redis 实际存储数据使用的内存。

`mem_fragmentation_ratio` 在 1～1.5 之间一般认为是正常的，内存碎片是无法避免的。

# 3 内存碎片如何清理

## 3.1 自动清理（Redis 4.0+）

启用 `activedefrag` 配置项，Redis 自动在后台整理碎片，相关配置参考 [2 内存碎片管理](Redis%20配置.md#2%20内存碎片管理)

* `active-defrag-ignore-bytes 100mb`：表示内存碎片的字节数达到 100MB 时，开始清理。
* `active-defrag-threshold-lower 10`：表示内存碎片空间占操作系统分配给 Redis 的总空间比例达到 10% 时，开始清理。

内存碎片的清理会占用 CPU，因此 Redis 提供了两个配置来控制内存碎片清理对 Redis 正常请求处理的影响：
* `active-defrag-cycle-min 1`： 表示自动清理过程所用 CPU 时间的比例不低于 25%，保证清理能正常开展
* `active-defrag-cycle-max 25`：表示自动清理过程所用 CPU 时间的比例不高于 75%，一旦超过，就停止清理，从而避免在清理时，大量的内存拷贝阻塞 Redis，导致响应延迟升高。

## 3.2 手动清理

调用命令：
* `redis-cli memory purge`
* `CONFIG SET activedefrag yes`

## 3.3 重启 Redis

过于暴力，且重启恢复数据阶段不能对外提供服务

## 3.4 配置优化

- **使用适当内存分配器**：如 `jemalloc`（默认）对碎片优化较好。
- **控制键过期/删除频率**：避免集中删除大量键，分散操作减少碎片。
- **禁用透明大页（THP）**：防止操作系统内存管理干扰，可参考[操作系统透明大页（THP）对 Redis 有什么影响？](操作系统透明大页（THP）对%20Redis%20有什么影响？.md)

# 4 Reference
* [20 \| 删除数据后，为什么内存占用率还是很高？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/289140)
* [使用info命令查看Redis 实例状态](使用info命令查看Redis%20实例状态.md)
* [2 内存碎片管理](Redis%20配置.md#2%20内存碎片管理)