---
title: Actuator Tomcat
tags:
  - fleet-note
  - development-framework/spring-boot/actuator
date: 2025-03-03
time: 13:52
aliases:
---
# 指标

`/actuator/metrics` 暴露的指标：

```json
{
  "names": [
    "application.ready.time",
    "application.started.time",
    "disk.free",
    "disk.total",
    "executor.active",
    "executor.completed",
    "executor.pool.core",
    "executor.pool.max",
    "executor.pool.size",
    "executor.queue.remaining",
    "executor.queued",
    "http.server.requests.active",
    "jvm.buffer.count",
    "jvm.buffer.memory.used",
    "jvm.buffer.total.capacity",
    "jvm.classes.loaded",
    "jvm.classes.unloaded",
    "jvm.compilation.time",
    "jvm.gc.live.data.size",
    "jvm.gc.max.data.size",
    "jvm.gc.memory.allocated",
    "jvm.gc.memory.promoted",
    "jvm.gc.overhead",
    "jvm.info",
    "jvm.memory.committed",
    "jvm.memory.max",
    "jvm.memory.usage.after.gc",
    "jvm.memory.used",
    "jvm.threads.daemon",
    "jvm.threads.live",
    "jvm.threads.peak",
    "jvm.threads.started",
    "jvm.threads.states",
    "logback.events",
    "process.cpu.usage",
    "process.files.max",
    "process.files.open",
    "process.start.time",
    "process.uptime",
    "system.cpu.count",
    "system.cpu.usage",
    "system.load.average.1m",
    "tomcat.cache.access",
    "tomcat.cache.hit",
    "tomcat.connections.config.max",
    "tomcat.connections.current",
    "tomcat.connections.keepalive.current",
    "tomcat.global.error",
    "tomcat.global.received",
    "tomcat.global.request",
    "tomcat.global.request.max",
    "tomcat.global.sent",
    "tomcat.servlet.error",
    "tomcat.servlet.request",
    "tomcat.servlet.request.max",
    "tomcat.sessions.active.current",
    "tomcat.sessions.active.max",
    "tomcat.sessions.alive.max",
    "tomcat.sessions.created",
    "tomcat.sessions.expired",
    "tomcat.sessions.rejected",
    "tomcat.threads.busy",
    "tomcat.threads.config.max",
    "tomcat.threads.current"
  ]
}

```


# Tomcat 配置说明

以下是 Spring Boot Actuator 中 Tomcat 相关指标项的详细解释，按功能分类整理：

---

### **缓存相关**
- **`tomcat.cache.access`**  
  Tomcat 静态资源缓存的总访问次数。每当请求尝试访问缓存中的资源时，该值递增。

- **`tomcat.cache.hit`**  
  缓存命中的次数。表示请求直接从缓存获取资源的次数，命中率可通过 `hit / access` 计算。

---

### **连接相关**
- **`tomcat.connections.config.max`**  
  Tomcat 配置的最大连接数（如 `server.tomcat.max-connections`）。超过此值的连接会被拒绝。

- **`tomcat.connections.current`**  
  当前活跃的连接数（正在处理请求或保持活跃状态）。

- **`tomcat.connections.keepalive.current`**  
  当前处于 `keep-alive` 状态的连接数（等待复用而非立即关闭）。

---

### **全局请求与流量**
- **`tomcat.global.error`**  
  全局错误总数（所有请求中返回 4 xx/5 xx 状态码的次数）。

- **`tomcat.global.received`**  
  接收的总字节数（客户端上传数据的总量）。

- **`tomcat.global.sent`**  
  发送的总字节数（服务器响应数据的总量）。

- **`tomcat.global.request`**  
  处理的总请求数（所有 HTTP 请求的累计数量）。

- **`tomcat.global.request.max`**  
  单个请求处理时间的最大值（单位：毫秒）。

---

### **Servlet 处理**
- **`tomcat.servlet.error`**  
  Servlet 处理过程中抛出的异常或错误总数。

- **`tomcat.servlet.request`**  
  Servlet 处理的总请求数（与全局 `global.request` 一致，除非存在其他处理方式）。

- **`tomcat.servlet.request.max`**  
  Servlet 请求处理时间的最大值（单位：毫秒）。

---

### **会话（Session）管理**
- **`tomcat.sessions.active.current`**  
  当前活跃的会话数（未被过期的会话）。

- **`tomcat.sessions.active.max`**  
  应用启动以来，同时活跃会话数的峰值。

- **`tomcat.sessions.alive.max`**  
  单个会话存活的最长时间（单位：秒）。

- **`tomcat.sessions.created`**  
  自应用启动后创建的会话总数。

- **`tomcat.sessions.expired`**  
  因超时而过期的会话数。

- **`tomcat.sessions.rejected`**  
  因超过配置的最大会话数而被拒绝创建的会话数。

---

### **线程池状态**
- **`tomcat.threads.busy`**  
  当前正在处理请求的线程数（忙碌线程）。

- **`tomcat.threads.config.max`**  
  线程池配置的最大线程数（如 `server.tomcat.threads.max`）。

- **`tomcat.threads.current`**  
  当前线程总数（包括空闲线程，即 `busy + idle`）。

---

### **使用场景示例**
- **监控性能瓶颈**  
  通过 `tomcat.threads.busy` 和 `tomcat.threads.config.max` 判断线程池是否过载。  
  若 `busy` 长期接近 `config.max`，需考虑优化代码或增加线程数。

- **分析缓存效率**  
  计算 `tomcat.cache.hit / tomcat.cache.access` 得到缓存命中率，低命中率可能需调整缓存策略。

- **诊断连接泄漏**  
  观察 `tomcat.connections.keepalive.current` 是否持续高位，可能意味着连接未正确关闭。

- **会话管理优化**  
  结合 `tomcat.sessions.active.max` 和 `tomcat.sessions.rejected` 调整会话超时时间或最大限制。

---

以上指标可通过 Actuator 的 `/actuator/metrics` 端点获取，结合监控工具（如 Prometheus+Grafana）可实现实时可视化与告警。
# Reference