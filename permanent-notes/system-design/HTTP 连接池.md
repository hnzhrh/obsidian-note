---
title: HTTP 连接池
tags:
  - permanent-note
  - system-design
date: 2025-02-28
time: 18:53
aliases:
---

Spring Boot 3.0+ Tomcat 有关性能优化的默认配置：

```yaml
server:  
  tomcat:  
    max-connections: 8192
    threads:
      max: 200
      min-spare: 10
    accept-count: 100
    keep-alive-timeout: 
    max-keep-alive-requests: 100 
    connection-timeout:
```


# 配置项介绍

| 配置项                       | 默认值  | 文档注释                                                                                                                                                                                                                                    | 解释                            |
| ------------------------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| `max-connections`         | 8192 | Maximum number of connections that the server accepts and processes at any given time. Once the limit has been reached, the operating system may still accept connections based on the "acceptCount" property.                          | 服务器可接收的最大连接数，受限于操作系统的可打开最大描述符 |
| `threads.max`             | 200  | Maximum amount of worker threads.                                                                                                                                                                                                       | 配置的 Tomcat 最大工作线程             |
| `threads.min-spare`       | 10   | Minimum amount of worker threads.                                                                                                                                                                                                       | 配置的 Tomcat 最小工作线程，这个配置就       |
| `accept-count`            | 100  | Maximum queue length for incoming connection requests when all possible request processing threads are in use.                                                                                                                          |                               |
| `keep-alive-timeout`      | /    | Time to wait for another HTTP request before the connection is closed. When not set the connectionTimeout is used. When set to -1 there will be no timeout.                                                                             |                               |
| `max-keep-alive-requests` | 100  | Maximum number of HTTP requests that can be pipelined before the connection is closed. When set to 0 or 1, keep-alive and pipelining are disabled. When set to -1, an unlimited number of pipelined or keep-alive requests are allowed. |                               |
| `connection-timeout`      | /    | Amount of time the connector will wait, after accepting a connection, for the request URI line to be presented.                                                                                                                         |                               |

# 配置经验

```
threads.max = (预期QPS × 平均响应时间RT(秒)) × 冗余系数(1.2~1.5)
accept-count 应能容纳突发流量，但需监控 avgQueueTime（JMeter的Latency）
```

但是也不能配置过高，如果线程数量太多还有上下文切换的开销。
# 操作系统配置

* Linux 文件句柄配置
```shell
# 查看当前限制
ulimit -n
# 临时修改（需同步修改 /etc/security/limits.conf）
ulimit -n 65535
```
* TCP 参数优化
```shell
# 快速回收TIME_WAIT连接
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_tw_recycle=1
```

# 如何根据 Metrics 调整配置？

| 指标          | 异常表现                     | 调整方向                   |
| ----------- | ------------------------ | ---------------------- |
| **QPS/TPS** | 低于预期                     | 增加线程数或优化业务逻辑           |
| **响应时间**    | P99延迟陡增                  | 检查线程阻塞或数据库瓶颈           |
| **错误率**     | 连接拒绝（Connection refused） | 增大 `accept-count` 或线程池 |
| **线程池利用率**  | 持续100%                   | 增大 `threads.max`       |
| **等待队列堆积**  | 队列满导致拒绝                  | 增大 `accept-count`      |

# Reference

* [Tomcat HTTP 配置](Tomcat%20HTTP%20配置.md)
* [Spring Boot Tomcat 配置](Spring%20Boot%20Tomcat%20配置.md)
* [Tomcat NIO vs WebFlux](Tomcat%20NIO%20vs%20WebFlux.md)