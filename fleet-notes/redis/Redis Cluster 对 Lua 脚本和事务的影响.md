---
title: Redis Cluster 对 Lua 脚本和事务的影响
tags:
  - fleet-note
  - middleware/redis/lua
  - middleware/redis/transaction
date: 2025-02-18
time: 18:21
aliases:
---
Lua 脚本和事务在 Cluster 集群中不能跨 Slot 执行，否则报错：`command keys must in same slot`。

解决办法：

1. 使用 Redis 的 `Hash tag` 机制，在 Key 中使用 `{}` 来强制作为 Redis Cluster 分配 slot 的 Hash key，例如 `user:{1001}:name` 和 `user:{1001}:age` 会被分配到同一个 slot 中
2. Redis 7.0 以后新增的 `allow-cross-slot-keys` 可以在同一个节点跨 slot 访问，但不能跨节点
3. 使用 Redis proxy 在客户端层面解决，比如 ：
	1. [GitHub - CodisLabs/codis: Proxy based Redis cluster solution supporting pipeline and scaling dynamically](https://github.com/CodisLabs/codis/)
	2. [GitHub - joyieldInc/predixy: A high performance and fully featured proxy for redis, support redis sentinel and redis cluster](https://github.com/joyieldInc/predixy)

# Reference

* [在redis-cluster下使用lua脚本的限制 - Sidney的小木屋](https://www.sidney.wiki/db/1141)
* [Benchmark · joyieldInc/predixy Wiki · GitHub](https://github.com/joyieldInc/predixy/wiki/Benchmark)