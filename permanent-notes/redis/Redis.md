---
title: Redis
tags:
  - permanent-note
  - middleware/redis
  - permanent-note/index
date: 2024-12-09
time: 16:04
aliases:
---
# 1 应用维度

> 如何实现哪些功能？如何设计？为什么要这么做？
## 1.1 [[Redis Big Keys 问题]]
# 2 系统维度

 > 某些核心功能是怎么运作的？怎么实现的？源码分析。可以从高性能、高并发、高可用三个方面去阐述。
## 2.1 [[Redis 有哪些数据类型？各自的实现是怎样的？]]
## 2.2 [[Redis 性能优化之绑定 CPU]]
## 2.3 [Redis 持久化机制](Redis%20持久化机制.md)
## 2.4 [Redis 延迟删除机制](Redis%20延迟删除机制.md)
## 2.5 [[Redis 过期数据处理机制]]
## 2.6 [[Redis 内存满了怎么办？]]
## 2.7 [[Redis 内存碎片问题]]
## 2.8 [[操作系统透明大页（THP）对 Redis 有什么影响？]]
## 2.9 [Redis 主从同步](Redis%20主从同步.md)
## 2.10 [[Redis 哨兵机制]]
## 2.11 Redis Cluster
### 2.11.1 Redis Cluster 客户端查找 Key 的路由过程

# 3 运维
## 3.1 [使用info命令查看Redis 实例状态](使用info命令查看Redis%20实例状态.md)
## 3.2 [[获取 Redis 进程号]]
## 3.3 [[使用 client list 命令查看客户端]]
## 3.4 [[异构数据导入 Redis]]
# 4 问题排查
## 4.1 [[Redis 突然变慢了怎么办？]]
# 5 Q&A
## 5.1 数据类型
### 5.1.1 `set` 一个已有的数据会怎么样？
### 5.1.2 Redis String 数据类型可以有多大？
### 5.1.3 为什么数据编码 `EMBSTR` 的阈值是 44？
### 5.1.4 为什么 Redis 要用 `SDS` 而不是 `char` 数组？
### 5.1.5 在 ZIPLIST 数据结构下，查询节点个数的时间复杂度是多少？
# 6 Reference
* [云数据库 Tair（兼容 Redis®）(Tair)-阿里云帮助中心](https://help.aliyun.com/zh/redis/?spm=a2c4g.11186623.0.0.3223490cwufiga)
* [Redis - The Real-time Data Platform](https://redis.io/)
* [Redis 配置](Redis%20配置.md)
* [Redis 最佳设计 Top 9 \| 张金铭的博客](https://www.zjmeow.com/archives/redis-best-design)
* [图解Redis介绍 | 小林coding](https://xiaolincoding.com/redis/)
* [Redis 核心技术与实战](https://time.geekbang.org/column/intro/100056701?tab=catalog)
* [Redis源码剖析与实战\_Redis\_Redis源码\_数据结构\_主从复制\_缓存\_集群\_分布式数据库\_键值数据库\_事件驱动框架-极客时间](https://time.geekbang.org/column/intro/100084301?tab=catalog)
* [16张图带你吃透Redis架构演进大家好，我是 Kaito。 这篇文章我想和你聊一聊 Redis 的架构演化之路。 .. - 掘金](https://juejin.cn/post/6925284711296155655) (大佬文章，需要好好读一读)
* [Segmented, Paged and Virtual Memory - YouTube](https://youtu.be/p9yZNLeOj4s?si=CVjlC-nsjzwapib6)
* [Redis为什么变慢了？常见延迟问题定位与分析Redis作为内存数据库，拥有非常高的性能，单个实例的QPS能够达到10W - 掘金](https://juejin.cn/post/6893396349648273422)
* [云数据库Tair（兼容Redis®）\_高性能内存数据库\_多线程并发处理\_数据库-阿里云](https://www.aliyun.com/product/tair?spm=a2c4g.11174283.0.0.69ab3629gWxzMZ)
* [云数据库 Tair（兼容 Redis®）(Tair)-阿里云帮助中心](https://help.aliyun.com/zh/redis/?spm=5176.29637306.J_AHgvE-XDhTWrtotIBlDQQ.9.1ba955b1Lp5oKb)