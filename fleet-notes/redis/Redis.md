---
title: Redis
tags:
  - fleet-note
  - middleware/redis
date: 2024-11-26
time: 16:53
aliases:
---

# 高性能（High Performance）

## 内存层

### 数据结构
#### Dict
[[Redis-dict]]

## 处理层
[Redis-CPU绑核](Redis-CPU绑核.md)

## IO 模型
## 存储层
### AOF
[Redis-aof](Redis-aof.md)
### RDB 
[Redis-rdb](Redis-rdb.md)

# 高可靠（High Reliability）
## 主从复制
[Redis-主从复制](Redis-主从复制.md)
## 哨兵机制
[Redis-哨兵](Redis-哨兵.md)

# 高可扩展性（High Scalability）
## 分片

# References
* [图解Redis介绍 | 小林coding](https://xiaolincoding.com/redis/)
* [Redis 核心技术与实战](https://time.geekbang.org/column/intro/100056701?tab=catalog)
* [Redis源码剖析与实战\_Redis\_Redis源码\_数据结构\_主从复制\_缓存\_集群\_分布式数据库\_键值数据库\_事件驱动框架-极客时间](https://time.geekbang.org/column/intro/100084301?tab=catalog)
* [16张图带你吃透Redis架构演进大家好，我是 Kaito。 这篇文章我想和你聊一聊 Redis 的架构演化之路。 .. - 掘金](https://juejin.cn/post/6925284711296155655) (大佬文章，需要好好读一读)
* [Segmented, Paged and Virtual Memory - YouTube](https://youtu.be/p9yZNLeOj4s?si=CVjlC-nsjzwapib6)