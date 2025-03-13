---
title: Redis 的 run id 和 replication id
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-11
time: 22:17
aliases: 
done: false
---

# `run id`

 通过 `info server` 可以查看到，该 `ID` 是标识唯一 Redis server 的随机值。一般被 `sentinel` 和 `cluster` 使用，重新启动都会重新生成。

# `replication id`

Replication id 是标识 Redis 实例数据集历史的唯一 ID，
* Master 冷启动，比如首次启动、清空数据后重新启动，会生成一个新的 replicaion id
* Replica 被提升为 Master 后会重新生成一个 replication id ，因为老的 Master 可能因为网络缘故依然存活，可能出现冲突

`replication id` 和 `second replication id` 提供了 Redis 主从同步 failover 后避免全量同步的能力，详情参考 [1.8.3 Redis 4.0 版本 `PSYNC2`](Redis%20主从同步流程和版本演变.md#1.8.3%20Redis%204.0%20版本%20`PSYNC2`)

# Reference
* [Redis replication \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/#replication-id-explained)