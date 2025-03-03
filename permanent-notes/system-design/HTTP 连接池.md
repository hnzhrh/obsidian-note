---
title: HTTP 连接池
tags:
  - permanent-note
  - system-design
date: 2025-02-28
time: 18:53
aliases:
---

# 配置项介绍

Spring Boot 3.0+ Tomcat 有关性能优化的默认配置：

```yaml
server:  
  tomcat:  
    max-connections: 8192
    threads:
      max: 200
      min-spare: 10
    accept-count: 100
    keep-alive-timeout: # 默认与connection-timeout 一致
    max-keep-alive-requests: 100 
    connection-timeout: 60s 
```


| 配置项                       | 默认值                    | 文档注释                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | 解释                                                                                                                                                                                                                                                                             |
| ------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `max-connections`         | 8192                   | The maximum number of connections that the server will accept and process at any given time. When this number has been reached, the server will accept, but not process, one further connection. This additional connection be blocked until the number of connections being processed falls below **maxConnections** at which point the server will start accepting and processing new connections again. Note that once the limit has been reached, the operating system may still accept connections based on the `acceptCount` setting. The default value is `8192`.<br><br>For NIO/NIO2 only, setting the value to -1, will disable the maxConnections feature and connections will not be counted. | 服务器在任意时刻能够接受并处理的最大并发连接数。一旦达到此数值上限，服务器将继续接受新的连接请求，但不会立即对其进行处理。这些额外的连接将被暂时挂起，直至当前处理的连接数降至**maxConnections**以下，届时服务器将恢复对新连接的正常接受与处理。需要注意的是，即便达到最大连接数限制，操作系统仍可能依据 `acceptCount` 参数的配置继续接受连接请求。该参数的默认值为 `8192`。<br><br>特别地，对于NIO/NIO2协议，若将此参数设置为-1，则会禁用最大连接数限制功能，此时系统不会对连接数进行统计与限制。 |
| `accept-count`            | 100                    | The maximum length of the operating system provided queue for incoming connection requests when `maxConnections` has been reached. The operating system may ignore this setting and use a different size for the queue. When this queue is full, the operating system may actively refuse additional connections or those connections may time out. The default value is 100.                                                                                                                                                                                                                                                                                                                            | 当服务器的连接数达到`maxConnections`上限时，操作系统会为待处理的连接请求维护一个队列，此参数用于指定该队列的最大长度。需要注意的是，操作系统可能会忽略此配置并自行调整队列的实际大小。若队列已满，操作系统可能会主动拒绝后续的连接请求，或者这些连接可能会因超时而被丢弃。该参数的默认值为100。                                                                                                                      |
| `threads.max`             | 200                    | The maximum number of request processing threads to be created by this **Connector**, which therefore determines the maximum number of simultaneous requests that can be handled. If not specified, this attribute is set to 200. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool. Note that if an executor is configured any value set for this attribute will be recorded correctly but it will be reported (e.g. via JMX) as `-1` to make clear that it is not used.                                                                                                           | 此**Connector**所创建的最大请求处理线程数，用于限制其能够同时处理的并发请求数量。若未显式配置，该参数的默认值为 200。若该 Connector 已关联一个外部线程池（executor），则此参数将失效，因为 Connector 会优先使用 executor 执行任务，而非依赖内部线程池。此外，若配置了 executor，尽管为该参数设置的值会被系统记录，但在监控数据中（例如通过 JMX）会显示为 `-1`，以明确表明该参数未实际生效。                                            |
| `threads.min-spare`       | 10                     | The minimum number of threads always kept running. This includes both active and idle threads. If not specified, the default of `10` is used. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool. Note that if an executor is configured any value set for this attribute will be recorded correctly but it will be reported (e.g. via JMX) as `-1` to make clear that it is not used.                                                                                                                                                                                               | 始终保持运行的最小线程数，包括活跃线程和空闲线程。若未显式配置，默认值为 `10`。如果该连接器已关联一个外部线程池（executor），则此参数将失效，因为连接器会优先使用 executor 来执行任务，而非依赖内部线程池。此外，若配置了 executor，尽管为该参数设置的值会被系统记录，但在监控数据中（例如通过 JMX）会显示为 `-1`，以明确表明该参数未实际生效。                                                                                   |
| `keep-alive-timeout`      | 同 `connection-timeout` | Time to wait for another HTTP request before the connection is closed. When not set the connectionTimeout is used. When set to -1 there will be no timeout.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | 长连接超时时间                                                                                                                                                                                                                                                                        |
| `max-keep-alive-requests` | 100                    | Maximum number of HTTP requests that can be pipelined before the connection is closed. When set to 0 or 1, keep-alive and pipelining are disabled. When set to -1, an unlimited number of pipelined or keep-alive requests are allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | 一个长连接最大处理请求数，在高并发系统中，需要调高或者直接设置为 -1 不限制                                                                                                                                                                                                                                        |
| `connection-timeout`      | 60 s                   | Amount of time the connector will wait, after accepting a connection, for the request URI line to be presented.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | 连接超时时间                                                                                                                                                                                                                                                                         |

# Tomcat 配置优化

![HTTP 连接池调优.png](https://images.hnzhrh.com/note/HTTP%20%E8%BF%9E%E6%8E%A5%E6%B1%A0%E8%B0%83%E4%BC%98.png)

# 相关操作系统配置优化

# 文件句柄数

手动查看占用：`lsof -p pid`

```shell
# 检查当前限制  
ulimit -n  
# 临时修改（需 root）  
ulimit -n 65535  
# 永久修改（/etc/security/limits.conf）  
* soft nofile 65535  
* hard nofile 65535
```

## TCP 参数优化

```shell
# 快速回收TIME_WAIT连接
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_tw_recycle=1
```

# 线程配置理论

## Little’s Law

> 系统中的请求数 = 请求的到达速率 × 每个请求处理时间

线程数配置经验公式：`Threads num = QPS × RT × 冗余系数（1.2 ~ 1.5）`

## 利用CPU 时间和 IO 时间计算

> 线程池大小 =（线程 I/O 阻塞时间 + 线程 CPU 时间) / 线程 CPU 时间

当线程因 I/O 阻塞时，其他线程可继续使用 CPU。公式通过时间比例估算需要多少线程才能**完全利用 CPU**，避免空闲。

假设某任务单次执行：
- **CPU 时间**：2 秒
- **I/O 阻塞时间**：8 秒  

则`线程池大小 = (2 + 8) / 2 = 5`。  

## 实际场景

1. 根据公式预估线程池大小
2. 进行压测，根据压测结果、指标监控、系统资源进行调整
3. 达到预期的效果


# 其他
## Tomcat 开启 MXBean

```yaml
# 开启Tomcat MXBean
server.tomcat.mbeanregistry.enabled = true
```

# Reference
* [Apache Tomcat 10 Configuration Reference (10.1.36) - The HTTP Connector](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)
* # [面试官：Spring Boot 最大连接数和最大并发数是多少？问倒一大片！](https://segmentfault.com/a/1190000044292458)
* [【高并发优化手段】基于Springboot项目(二)-阿里云开发者社区](https://developer.aliyun.com/article/1351156)
* [docker容器高性能之文件句柄](https://segmentfault.com/a/1190000041105289)
* [38 \| Tomcat拒绝连接原因分析及网络优化-深入拆解Tomcat & Jetty -极客时间](https://time.geekbang.org/column/article/115471)
* [36 \| Tomcat I/O和线程池的并发调优-深入拆解Tomcat & Jetty -极客时间](https://time.geekbang.org/column/article/112271)
* [网络连接状态](网络连接状态.md)
* [Tomcat HTTP 配置 AI 回答](Tomcat%20HTTP%20配置%20AI%20回答.md)
* [Spring Boot HTTP 性能优化配置方案](Spring%20Boot%20HTTP%20性能优化配置方案.md)