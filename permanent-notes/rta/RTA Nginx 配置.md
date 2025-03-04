---
title: RTA Nginx 配置
tags:
  - permanent-note
  - project/rta
  - nginx
date: 2025-02-28
time: 17:55
aliases:
---
RTA Nginx 配置使用 [Nginx 性能优化配置](Nginx%20性能优化配置.md)。

# 配置前后端长连接

通过配置前后端长连接避免大量的 `5XX` 错误和过多 `TIME_WAIT` 状态的连接。

# Reference
* [nginx反向代理时保持长连接-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1832932)