---
title: RTA Redis 数据存储容量优化
tags:
  - permanent-note
  - middleware/redis/optimization
  - middleware/redis/persistence
date: 2025-03-03
time: 17:19
aliases:
---
本文核心优化方式根据 [RTA Redis 数据结构](RTA%20DDD%20Design.md#5.4.1%20Device%20cache%20data%20structure) 进行讨论，写入 100 W 设备号，固定 5 个营销计划, 每个设备号属于 10 个定向人群包，属于10 个排除人群包，频控窗口为 30 天。

# Key 优化

# Value 优化

## 人群包长 ID 变短 ID

## 数据压缩

在只使用压缩之前，100 W 设备好占用内存 `2.3 G`，直接使用 Protobuf 压缩后下降到 `921.11 M`。

# 数据结构

使用 `Listpack` 编码代替 `Hash table` 编码使其内存更加紧凑，更加省内存，该场景下使用紧凑内存可使其下降到 `696.04M`

# Reference
* [云音乐RTA投放与承接系统建设实践](云音乐RTA投放与承接系统建设实践.md)
* [使用info命令查看Redis 实例状态](使用info命令查看Redis%20实例状态.md)
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)
* [Redis listpack 编码](Redis%20listpack%20编码.md