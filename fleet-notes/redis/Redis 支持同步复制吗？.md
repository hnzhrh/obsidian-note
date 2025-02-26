---
title: Redis 支持同步复制吗？
tags:
  - fleet-note
  - middleware/redis/persistence
date: 2025-02-18
time: 17:33
aliases:
---

Redis 不完全支持主从的同步复制，但可以使用 `WAIT` 命令来最大程度的保证数据一致性。

```redis
WAIT numreplicas timeout
```

 这个命令会阻塞当前主服务器端，直到指定的 `numreplicas` 副本数目与主服务器数据一致，该命令不会阻塞从服务器的读，所以不可能是强一致性。
# Reference