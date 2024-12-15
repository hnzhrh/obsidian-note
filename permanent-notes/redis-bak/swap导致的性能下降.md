---
title: swap导致的性能下降
tags:
  - permanent-note
  - middleware/redis/ops
date: 2024-12-06
time: 10:42
aliases:
---
Redis 会在以下情况下出现 swap 现象：

1. **内存不足**：当 Redis 实例使用的内存量超过了服务器的物理内存时，操作系统可能会将部分不活跃的数据从内存中换出到磁盘上的交换空间（swap）。这通常发生在 Redis 配置的 `maxmemory` 设置过大或者没有设置，导致 Redis 占用了过多的内存，超出了物理内存的限制。

2. **其他进程占用大量内存**：即使 Redis 的内存使用量在合理范围内，如果同一台服务器上的其他进程占用了大量的内存，也可能迫使操作系统使用 swap 来管理有限的物理内存资源。

3. **配置不当**：如果 Redis 或者系统级别的配置不适当，例如设置了过大的日志缓存、连接数过多等，也会间接导致内存压力增加，进而触发 swap。

要检查 Redis 是否发生了 swap，可以按照如下步骤操作：

1. **获取 Redis 进程 ID**：
   使用 `redis-cli info server | grep process_id` 命令来获取 Redis 的进程 ID (PID)。这一步是了解哪个进程是我们要监控的目标。

2. **查看 /proc 文件系统下的 smaps 文件**：
   进入 `/proc/<PID>` 目录，然后执行 `cat smaps | egrep '^(Swap|Size)'`。这个命令会显示每个内存映射区域的大小和对应的 swap 数量。如果看到非零的 Swap 值，那么就意味着该区域的部分或全部内容已经被换到了 swap 分区。

3. **使用 free 命令**：
   执行 `free -m` 可以查看整个系统的内存使用情况，包括已使用的 swap 大小。如果有显著的 swap 使用量，这可能意味着系统整体上正在经历内存压力。

4. **检查 Redis 日志**：
   查看 Redis 日志文件，寻找任何与内存相关的问题报告或警告信息。某些版本的 Redis 在检测到高 swap 使用率时可能会记录相应的消息。

5. **监控工具**：
   使用专业的监控工具如 Prometheus + Grafana, Datadog, New Relic 等，这些工具可以帮助实时跟踪 Redis 和系统的性能指标，包括内存和 swap 的使用情况。

6. **调整 maxmemory 和策略**：
   确保正确设置了 `maxmemory` 参数，并选择了合适的淘汰策略 (`maxmemory-policy`)。这有助于防止 Redis 内存无节制增长，从而避免不必要的 swap 活动。

7. **优化应用和数据结构**：
   评估应用程序的行为和 Redis 中存储的数据结构，尝试减少不必要的大对象存储，优化键值对的生命周期管理，以及适时清理不再需要的数据。
# Reference