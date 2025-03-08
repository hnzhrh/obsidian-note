---
title: 获取 Redis 进程号
tags:
  - permanent-note
  - middleware/redis/ops
date: 2025-03-06
time: 16:59
aliases:
---

通过 `pidfile` 参数指定进程文件。

```shell
redis-server --pidfile /var/run/redis.pid
```

# Reference