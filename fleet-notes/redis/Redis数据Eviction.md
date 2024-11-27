---
title: Redis数据Eviction
tags:
  - fleet-note
  - middleware/redis/eviction
date: 2024-11-26
time: 09:31
aliases:
---

- `noeviction`: Keys are not evicted but the server will return an error when you try to execute commands that cache new data. If your database uses replication then this condition only applies to the primary database. Note that commands that only read existing data still work as normal.
- `allkeys-lru`: Evict the [least recently used](https://redis.io/docs/latest/develop/reference/eviction/#apx-lru) (LRU) keys.
- `allkeys-lfu`: Evict the [least frequently used](https://redis.io/docs/latest/develop/reference/eviction/#lfu-eviction) (LFU) keys.
- `allkeys-random`: Evict keys at random.
- `volatile-lru`: Evict the least recently used keys that have the `expire` field set to `true`.
- `volatile-lfu`: Evict the least frequently used keys that have the `expire` field set to `true`.
- `volatile-random`: Evict keys at random only if they have the `expire` field set to `true`.
- `volatile-ttl`: Evict keys with the `expire` field set to `true` that have the shortest remaining time-to-live (TTL) value.

需要重点了解 LRU 和 LFU
# References
* [Key eviction | Docs](https://redis.io/docs/latest/develop/reference/eviction/)