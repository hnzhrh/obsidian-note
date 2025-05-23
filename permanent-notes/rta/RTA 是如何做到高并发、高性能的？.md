---
title: RTA 是如何做到高并发、高性能的？
tags:
  - permanent-note
  - project/rta
date: 2025-02-28
time: 15:06
aliases:
---
# 1 网关层面

RTA 选择了云厂商的负载均衡器，路由到 RTA Nginx 集群。
[配置清单](RTA%20架构设计.md#配置清单)
[RTA Nginx 配置](RTA%20Nginx%20配置)
# 2 横向扩展和纵向扩展

# 3 限流和降级熔断

# 4 缓存

# 5 池化技术
[HTTP 连接池](HTTP%20连接池.md)
# 6 常规优化
## 6.1 日志

配置开关，关闭日志输出

## 6.2 数据预热

Redis 数据预热接口，主要目的是为了避免 ASK 和 MOV 请求。

## 6.3 数据压缩

# 7 其他优化的未来展望
* 使用原生高并发的语言，比如 Golang
* 不使用 Tomcat，使用 Undertow，实际测试下来 Undertow 表现略微好一些
* 使用 WebFlux + Lettuce 响应式编程
* 直接结合 Nginx 旁路缓存调用 Lua 脚本

# 8 Reference
* [System Design Index](System%20Design%20Index.md)
* [别再误人子弟了-tomcat、undertow、jetty性能对比-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1644339)
* [实践-tomcat-vs-undertow-高并发压测\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1fK421a7rH/?share_source=copy_web&vd_source=3eb28f54d17403f9d05aaa09bef421a4)