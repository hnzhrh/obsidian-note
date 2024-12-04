---
title: Redis-CPU绑核
tags:
  - fleet-note
  - middleware/redis/performance
date: 2024-11-30
time: 16:06
aliases:
---
CPU 一般有 L 1，2，3 层缓存，其中物理核心拥有各自的 L 1，L 2 缓存，不同的物理核心共享 L 3 缓存。

主流的 CPU 处理器，物理核会运行两个逻辑核心（超线程），同一个物理核的逻辑核共享 L 1、2 缓存。

> 在多 CPU 架构上，应用程序可以在不同的处理器上运行。在刚才的图中，Redis 可以先在 Socket  1 上运行一段时间，然后再被调度到 Socket  2 上运行。但是，有个地方需要你注意一下：如果应用程序先在一个 Socket 上运行，并且把数据保存到了内存，然后被调度到另一个 Socket 上运行，此时，应用程序再进行内存访问时，就需要访问之前 Socket 上连接的内存，这种访问属于远端内存访问。和访问 Socket 直接连接的内存相比，远端内存访问会增加应用程序的延迟。在多 CPU 架构下，一个应用程序访问所在 Socket 的本地内存和访问远端内存的延迟并不一致，所以，我们也把这个架构称为非统一内存访问架构（Non-Uniform Memory Access，NUMA 架构）。



# References
* [17 | 为什么CPU结构也会影响Redis的性能？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/286082)