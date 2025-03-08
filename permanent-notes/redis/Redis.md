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
## 2.4 [[Redis 内存满了怎么办？]]
## 2.5 [[Redis 内存碎片问题]]
## 2.6 [[操作系统透明大页（THP）对 Redis 有什么影响？]]
## 2.7 [[Redis 主从复制]]
## 2.8 Redis Cluster
### 2.8.1 Redis Cluster 客户端查找 Key 的路由过程
# 3 运维和问题排查
## 3.1 方法
### 3.1.1 [使用info命令查看Redis 实例状态](使用info命令查看Redis%20实例状态.md)
### 3.1.2 [[获取 Redis 进程号]]
## 3.2 案例
# 4 Q&A
## 4.1 数据类型
### 4.1.1 `set` 一个已有的数据会怎么样？
### 4.1.2 Redis String 数据类型可以有多大？
### 4.1.3 为什么数据编码 `EMBSTR` 的阈值是 44？
### 4.1.4 为什么 Redis 要用 `SDS` 而不是 `char` 数组？
### 4.1.5 在 ZIPLIST 数据结构下，查询节点个数的时间复杂度是多少？
# 5 Reference
* [云数据库 Tair（兼容 Redis®）(Tair)-阿里云帮助中心](https://help.aliyun.com/zh/redis/?spm=a2c4g.11186623.0.0.3223490cwufiga)
* [Redis - The Real-time Data Platform](https://redis.io/)
* [Redis 配置](Redis%20配置.md)
* [Redis 最佳设计 Top 9 \| 张金铭的博客](https://www.zjmeow.com/archives/redis-best-design)