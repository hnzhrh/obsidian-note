---
title: 使用info命令查看Redis 实例状态
tags:
  - permanent-note
  - middleware/redis/ops
date: 2024-12-06
time: 10:05
aliases:
---
# Memory

- `used_memory`: Redis 分配器分配的内存量（字节）。这是 Redis 实际使用的内存，包括所有数据和内部开销。
- `used_memory_human`: 以人类可读的格式表示 `used_memory`。
- `used_memory_rss`: 这个值显示了操作系统认为 Redis 进程占用了多少物理内存（驻留集大小），它通常会比 `used_memory` 大，因为这包含了未释放但可以回收的内存页、共享库等。
- `used_memory_rss_human`: 以人类可读的格式表示 `used_memory_rss`。
- `used_memory_peak`: Redis 内存使用量的历史峰值（字节）。
- `used_memory_peak_human`: 以人类可读的格式表示 `used_memory_peak`。
- `used_memory_peak_perc`: 当前内存使用量相对于历史峰值的比例。
- `used_memory_overhead`: Redis 中非数据结构占用的内存量。
- `used_memory_startup`: Redis 启动时的初始内存开销。
- `used_memory_dataset`: 数据集实际占用的内存量。
- `used_memory_dataset_perc`: 数据集内存相对于总内存使用量的比例。
- `allocator_allocated`, `allocator_active`, `allocator_resident`: 分别代表 jemalloc 分配器分配的内存、活跃的内存区域以及驻留在 RAM 中的内存量。
- `total_system_memory`: 系统总内存。
- `total_system_memory_human`: 以人类可读的格式表示 `total_system_memory`。
- `used_memory_lua`: Lua 引擎占用的内存。
- `used_memory_vm_eval`: Lua 虚拟机评估脚本所用的内存。
- `used_memory_scripts_eval`: 已缓存 Lua 脚本的内存使用量（这里是 0，意味着没有缓存的脚本）。
- `number_of_cached_scripts`: 缓存的 Lua 脚本数量。
- `number_of_functions`, `number_of_libraries`: 定义的函数和加载的库的数量（这里是 0，意味着没有定义或加载）。
- `used_memory_vm_functions`, `used_memory_vm_total`: Lua VM 相关内存使用。
- `used_memory_functions`, `used_memory_scripts`: 函数和脚本占用的额外内存。
- `maxmemory`, `maxmemory_human`: 设置的最大内存限制（这里为 0，表示无限制）。
- `maxmemory_policy`: 达到最大内存限制时采用的淘汰策略（这里为 noeviction，即不驱逐任何键）。
- `allocator_frag_ratio`, `allocator_frag_bytes`, `allocator_rss_ratio`, `allocator_rss_bytes`: 内存碎片化比率和字节数。
- `rss_overhead_ratio`, `rss_overhead_bytes`: RSS 的额外开销比率和字节数。
- `mem_fragmentation_ratio`: 内存碎片化比率，`used_memory_rss` 与 `used_memory` 的比率。
- `mem_fragmentation_bytes`: 内存碎片化的字节数。
- `mem_not_counted_for_evict`, `mem_replication_backlog`, `mem_total_replication_buffers`, `mem_clients_slaves`, `mem_clients_normal`, `mem_cluster_links`, `mem_aof_buffer`: 各种类型的内存使用情况，如客户端连接、复制缓冲区等。
- `mem_allocator`: 使用的内存分配器（这里是 jemalloc 版本 5.2.1）。
- `active_defrag_running`: 是否有活动的内存碎片整理在运行（0 表示没有）。
- `lazyfree_pending_objects`, `lazyfreed_objects`: 延迟释放的对象数。

# Reference
* [INFO \| Docs](https://redis.io/docs/latest/commands/info/)