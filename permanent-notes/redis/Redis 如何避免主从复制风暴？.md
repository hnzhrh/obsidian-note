---
title: Redis 如何避免主从复制风暴？
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-11
time: 10:31
aliases: 
done: false
---
**主从复制风暴**指Redis主节点因处理大量从节点的复制请求，导致资源耗尽（如CPU、内存、网络带宽）而引发性能骤降或服务中断的现象。

产生原因：
* 过多从节点同时复制，导致磁盘、网络过载
* 全量复制开销大
* 在高并发写入情况下，`Replication backlog buffer` 设置过小导致从节点部分同步失效，重新开启全量同步
* 生成 RDB 文件时，`Replication buffer` 缓冲区溢出导致主节点关闭了从节点的连接，导致同步反复失败

如何避免：
* 采用 [7.3 树状拓扑结构](Redis%20主从同步.md#7.3%20树状拓扑结构) 降低主服务器的压力
* 避免同时启动多个从节点同步
* 优化复制配置
	* 增大 `Replication backlog buffer` ，确保从节点掉线后能执行重新同步而不是全量同步
	* 合理设置 [3.5 `repl-timeout 60`](Redis%20主从同步.md#3.5%20`repl-timeout%2060`) 避免网络波动触发全量同步
	* 增大 `Replicationg buffer` ，防止缓冲区溢出
# Reference