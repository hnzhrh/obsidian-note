---
title: RTA 架构设计
tags:
  - permanent-note
  - project/rta
date: 2025-02-26
time: 21:37
aliases:
---
# 架构图

![RTA架构.png](https://images.hnzhrh.com/note/RTA%E6%9E%B6%E6%9E%84.png)

# 配置清单

基于 15 W QPS 和 RT<60 ms

| 资源        | 配置     | 数量                                        |
| --------- | ------ | ----------------------------------------- |
| SLB       | 800 M  | N                                         |
| RTA 应用服务器 | 4C 8G  | 48                                        |
| Nginx     | 4C 8G  | 6                                         |
| Redis 集群  | 4C 32G | 6 台，每台部署 4 个容器节点，每个节点 7 G，共 24 个主节点，不需要副本 |
| Nacos     | 4C 8G  | 5 台                                       |




# Reference
* [Loading \| onemodel](https://www.onemodel.app/d/0AXIcFYlHHjP7Z9ByvsbG)