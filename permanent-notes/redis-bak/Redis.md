---
title: Redis
tags:
  - permanent-note
  - middleware/redis
date: 2024-11-25
time: 16:57
aliases:
---
# 应用维度

> 如何应用该产品实现哪些功能？如何设计？为什么要这么做？

* [[Redis有哪些数据类型可以使用？]]

# 系统维度

> 某些核心功能是怎么运作的？怎么实现的？源码分析

* [[Redis有哪些数据结构？为了性能考虑Redis做了哪些优化？]]
* [[Redis内存满了怎么办？]]
* [[Redis配置和部署]]
* [Redis持久化机制](Redis持久化机制.md)
* [[Redis 如何保证系统高可靠？]]
* [Redis单机升级集群方案](Redis单机升级集群方案.md)

# 问题排查

## 方案

> 怎么去排查某一类问题？
* [使用info命令查看Redis 实例状态](使用info命令查看Redis%20实例状态.md)
* [swap导致的性能下降](swap导致的性能下降.md)
## 案例

> 积攒的问题排查案例


# Reference

* [这篇Redis内存模型也太细了吧上一篇文章中，我们介绍了Redis的5种基本对象类型（字符串、哈希、列表、集合、有序集合 - 掘金](https://juejin.cn/post/6885710269406789645)