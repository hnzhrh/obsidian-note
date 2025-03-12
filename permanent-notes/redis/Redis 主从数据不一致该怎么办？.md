---
title: Redis 主从数据不一致该怎么办？
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-11
time: 08:08
aliases: 
done: false
---
# 1 为什么会出现主从数据不一致？

* 主从库同步异步，有一定的传输延时
* 从库可能正在执行其他复杂度高的命令，导致命令的执行被延后
# 2 如何应对主从数据不一致？
## 2.1 硬件方面保证主从的网络连接状态良好
同机房部署
## 2.2 监控同步状态
使用 `INFO` 命令监控 `master_repl_offset` 和 `slave_repl_offset`，如果差距过大可以拒绝客户端读取从服务器
## 2.3 忽略
如果业务能接受，最好忽略
## 2.4 针对过期数据冗余存储过期时间
在 Value 中也存储一个过期时间，客户端二次判断过期时间进行兜底，但也存在时钟漂移问题。
## 2.5 确保从节点只读
避免因为客户端在从节点写入导致的数据不一致。
## 2.6 [`min-replicas-to-write 3` 和 `min-replicas-max-lag 10`](Redis%20主从同步.md#3.13%20`min-replicas-to-write%203`%20和%20`min-replicas-max-lag%2010`) 配置
可提升一致性，降低可用性
## 2.7 优化复制缓冲区和超时配置
* [3.7 `repl-backlog-size 1mb`](Redis%20主从同步.md#3.7%20`repl-backlog-size%201mb`)，缓冲区保存最近的主节点写操作，从节点短时间断连后可通过**增量同步**恢复，避免全量同步
* [3.5 `repl-timeout 60`](Redis%20主从同步.md#3.5%20`repl-timeout%2060`)，防止因短暂网络波动误判从节点超时
## 2.8 配置 [ `replica-serve-stale-data yes`](Redis%20主从同步.md#3.1%20`replica-serve-stale-data%20yes`)
当检测到 Master 已经掉线或者 RDB 同步正在进行中，配置为 `yes` 则 Replica 拒绝服务，保证了一定的数据一致性


# 3 Reference
* [32 \| Redis主从同步与故障切换，有哪些坑？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/303247)