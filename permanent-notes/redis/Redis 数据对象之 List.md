---
title: Redis 数据对象之 List
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-04
time: 08:52
aliases:
  - Redis List Object
---
# 1 实现版本变更

* Redis 3.2 版本之前用 [[Redis Object Encoding LINKEDLIST]] 和 [Redis Object Encoding ZIPLIST](Redis%20Object%20Encoding%20ZIPLIST.md)
* Redis 3.2 版本引入了 [Redis Object Encoding QUICKLIST](Redis%20Object%20Encoding%20QUICKLIST.md) 编码方式降低连锁更新的影响。
* Redis 5.0 版本引入了 [Redis Object Encoding LISTPACK](Redis%20Object%20Encoding%20LISTPACK.md) 编码方式解决了连锁更新。
* Redis 7.0 版本彻底废除了 ZIPLIST 编码方式。

`quicklist` 是 Redis 从 3.2 版本开始引入的一种优化过的 List 数据类型的实现。它旨在结合 `linkedlist` 和 `ziplist`（7.0 以后替换为 `listpack`）的优点，提供更好的性能和内存效率。

# 2 Reference
* [Redis源码-List：Redis List存储原理、Redis List命令、 Redis List存储底层编码quicklist、Redis List应用场景\_redis的list源码-CSDN博客](https://blog.csdn.net/qq_41929714/article/details/126342953)
* [Redis Ziplist Data Structure. Redis is an in-memory storage server… \| by Shubhi Jain \| InterviewNoodle](https://interviewnoodle.com/redis-ziplist-data-structure-23c8e7e3266d)
* [listpack/listpack.md at master · antirez/listpack · GitHub](https://github.com/antirez/listpack/blob/master/listpack.md)
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)