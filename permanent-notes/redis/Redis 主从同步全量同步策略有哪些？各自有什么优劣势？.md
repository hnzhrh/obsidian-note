---
title: Redis 主从同步全量同步策略有哪些？各自有什么优劣势？
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-11
time: 09:54
aliases: 
done: false
---
# 全量同步策略

1. **Disk-backed（基于磁盘）**
    - **流程**：主节点 fork 子进程将 RDB **写入磁盘**，再由父进程逐步传输给副本。    
    - **特点**： 
        - RDB 文件生成后，**后续新连接的副本可直接复用该文件**，无需重复生成。        
        - 传输过程基于磁盘文件，支持批量服务多个副本。        
2. **Diskless（无盘）**
    - **流程**：主节点 fork 子进程**直接将 RDB 写入副本的 Socket**，完全绕过磁盘。    
    - **特点**：    
        - 传输开始后，新副本需等待当前传输完成。        
        - 主节点会等待 `repl-diskless-sync-delay`（默认 5 秒），**聚合多个副本后再并行传输**。        
        - 适用于磁盘慢但网络快的场景。

# 基于磁盘和无盘对比

| **维度**        | **Disk-backed**              | **Diskless**             |
| ------------- | ---------------------------- | ------------------------ |
| **磁盘 I/O 影响** | ❌ 频繁写入 RDB 文件，可能成为磁盘瓶颈。      | ✅ 完全避免磁盘写入，适合慢磁盘环境。      |
| **网络要求**      | ✅ 对网络带宽要求较低，数据分批次传输。         | ❌ 需要高带宽，需快速传输整个 RDB 文件。  |
| **多副本处理能力**   | ✅ 单次生成 RDB 文件可服务多个副本，资源利用率高。 | ❌ 每个传输独立处理，并行能力依赖延迟配置。   |
| **容错性**       | ✅ RDB 持久化到磁盘，传输中断后可恢复。       | ❌ 传输中断需重新生成 RDB，增加主节点负担。 |
| **延迟敏感度**     | ❌ RDB 写入磁盘增加同步延迟。            | ✅ 数据直传副本，降低延迟（需网络配合）。    |
| **内存压力**      | ❌ Fork 子进程写磁盘可能触发内存拷贝（COW）。  | ✅ 直接写 Socket，内存压力相对较小。   |
# 配置

配置 [`repl-diskless-sync yes`](Redis%20主从同步.md#3.2%20`repl-diskless-sync%20yes`) 开启无盘复制，配置 `repl-diskless-sync-delay 5` 可以等待 5 s 中并发传输 RDB，如果在等待期间需要同步的副本达到了 [`repl-diskless-sync-max-replicas 0`](Redis%20主从同步.md#3.3%20`repl-diskless-sync-max-replicas%200`) 则直接开始传输。

配置等待期是为了避免重复生成 RDB，通过顺序写入多个 Socket，实现网络层并发传输。

为了实现高性能的传输，Redis 使用零拷贝技术（`sendfile` 或者`mmap + write`），通过 **共享内存页** 或 **写时复制（COW）** 技术，将同一份数据映射到多个 Socket 的发送缓冲区，**避免内存重复占用**。

# Reference
* [9.1 什么是零拷贝？ \| 小林coding](https://xiaolincoding.com/os/8_network_system/zero_copy.html#_9-1-%E4%BB%80%E4%B9%88%E6%98%AF%E9%9B%B6%E6%8B%B7%E8%B4%9D)