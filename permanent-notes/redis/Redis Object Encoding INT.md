---
title: Redis Object Encoding INT
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 15:39
aliases:
---
Redis 如果发现存入的值是整数，则会使用 `INT` 编码，可能会使用到 [Redis共享对象](Redis共享对象)。使用整数时，直接将 redisObject 的 ptr 指针赋值为整数值即可。

源码可参考方法 `createStringObjectFromLongLongWithOptions`

# Reference