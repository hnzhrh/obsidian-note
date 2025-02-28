---
title:
  Spring Boot Tomcat 配置
tags:
  - literature-note
date:
  2025-02-28
time:
  19:11
aliases:
---
以下是针对 Spring Boot 3. X 的 HTTP 性能优化配置方案，适配新版本特性并满足 **RT=1 s、QPS=4000** 的要求：

---

### **一、Spring Boot 3. X 核心变化**
1. **JDK 17+ 要求**：默认支持虚拟线程（需手动启用，Loom 项目）。  
2. **Jakarta EE 9+**：包名从 `javax.*` 改为 `jakarta.*`（影响部分配置）。  
3. **默认服务器优化**：Tomcat 10. X、Jetty 11. X 等嵌入式服务器性能提升。  
4. **GraalVM 原生镜像支持**（可选）：通过 Spring Native 减少启动时间和内存占用。

---

### **二、关键配置方案**
#### **1. Tomcat 线程池配置（同步阻塞模型）**
```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 1200          # 最大工作线程（公式：QPS * RT = 4000*1=4000，需留冗余）
      min-spare: 100     # 最小空闲线程（预热）
    accept-count: 1000   # 等待队列容量
    max-connections: 2000 # 最大连接数
```
**适配说明**：  
- Spring Boot 3. X 中 `server.tomcat.max-threads` 改为 `server.tomcat.threads.max`。  
- **仅适用于同步阻塞模型**（如 Spring MVC），若需更高性能需切换到异步模型（WebFlux）。

---

#### **2. 启用虚拟线程（JDK 21+ Loom 项目）**
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true  # 启用虚拟线程（需 JDK 21+）
```
```java
// 配置虚拟线程池
@Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
public AsyncTaskExecutor asyncTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```
**优势**：  
- **轻量级线程**：支持百万级线程，避免传统线程池的上下文切换开销。  
- **兼容性**：与 Spring MVC 和 WebFlux 兼容，无需重构代码。  

---

#### **3. 异步非阻塞模型（WebFlux + Netty）**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```
```yaml
# application.yml
server:
  netty:
    max-in-memory-size: 16MB  # 控制请求体内存缓冲
```
**理由**：  
- **事件驱动模型**：Netty 的 EventLoop 线程数通常为 CPU 核数 * 2，适合高并发场景。  
- **非阻塞 IO**：避免线程阻塞，轻松支持 4000 QPS。  

---

#### **4. HTTP/2 与压缩配置**
```yaml
server:
  http2:
    enabled: true
  compression:
    enabled: true
    mime-types: text/html,text/css,application/json,application/xml
    min-response-size: 1024  # 最小压缩响应大小
  ssl:
    enabled: true  # HTTP/2 强制要求 HTTPS
```
**注意**：  
- Spring Boot 3. X 默认使用 Tomcat 10. X，需确保 SSL 证书配置正确。  
- 推荐使用 **TLSv 1.3** 提升握手效率。

---

#### **5. 数据库连接池优化（HikariCP）**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db?useSSL=false&allowPublicKeyRetrieval=true
    hikari:
      maximum-pool-size: 200  # 连接池大小与业务线程数匹配
      connection-timeout: 2000
      idle-timeout: 30000
```
**关键点**：  
- 若使用 **响应式驱动**（如 R 2 DBC），需配合 WebFlux 实现全链路非阻塞。

---

### **三、JVM 与系统级调优**
#### **1. JVM 参数（JDK 17+）**
```bash
# 推荐使用 G1 或 ZGC
java -jar \
  -Xms4g -Xmx4g \
  -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \  # 或 -XX:+UseZGC
  -Dspring.threads.virtual.enabled=true \   # 启用虚拟线程
  app.jar
```
**GC 选择**：  
- **ZGC**：低延迟（暂停时间 <1 ms），适合高吞吐场景。  
- **G 1**：平衡吞吐和延迟，成熟稳定。

---

#### **2. 操作系统优化（Linux）**
```bash
# 调整文件描述符限制
ulimit -n 100000

# 网络参数优化
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
```

---

### **四、性能验证与横向扩展**
#### **1. 单机性能估算**
| 模型           | 理论 QPS               | 配置要求                  |
|----------------|------------------------|---------------------------|
| 同步阻塞       | ~1200 (max-threads/RT) | Tomcat 线程池 1200        |
| 异步非阻塞     | ~10000+                | WebFlux + 全链路非阻塞    |
| 虚拟线程       | ~4000+                 | JDK 21+，无阻塞操作       |

**结论**：  
- **单机实现 4000 QPS**：  
  - **方案 1**：WebFlux + 异步数据库驱动（如 R 2 DBC）。  
  - **方案 2**：Spring MVC + 虚拟线程（需确保无同步阻塞调用）。  
- **横向扩展**：通过 Kubernetes 或 Nginx 部署多实例。

---

#### **2. 压测工具命令示例**
```bash
# 使用 wrk 进行压测
wrk -t16 -c4000 -d30s --latency https://your-api-endpoint
```
**监控指标**：  
- 线程池使用率（Tomcat/Netty）  
- GC 暂停时间与频率  
- 数据库连接池等待时间  

---

### **五、Spring Boot 3. X 特有优化**
#### **1. 原生镜像编译（GraalVM）**
```bash
# 生成原生镜像（需 Spring Native 依赖）
mvn -Pnative native:compile
```
**优势**：  
- 启动时间 <100 ms，内存占用减少 50%+。  
- 适合云原生场景，但对反射/动态代理有限制。

---

#### **2. 响应式 SQL 访问（R 2 DBC）**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
```
```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:mysql://user:password@localhost:3306/db
```
**全链路非阻塞**：从 HTTP 到数据库实现零阻塞。

---

### **六、配置总结**
| 优化方向         | Spring Boot 3. X 配置要点                          |
|------------------|---------------------------------------------------|
| 线程模型         | 优先选择 WebFlux 或虚拟线程（JDK 21+）            |
| 服务器配置       | Tomcat `threads.max` 或 Netty 事件循环            |
| 数据库访问       | HikariCP 连接池或 R 2 DBC 响应式驱动                |
| 协议与压缩       | 强制启用 HTTP/2 + TLSv 1.3 + GZIP                  |
| JVM 调优         | ZGC/G 1 + 堆内存 4 GB+                             |
| 系统参数         | 文件描述符、TCP 缓冲区、端口复用                  |

---

### **注意事项**
1. **虚拟线程限制**：若代码中存在 `synchronized` 或阻塞 IO 调用，虚拟线程优势无法发挥。  
2. **全链路异步**：WebFlux 需配合非阻塞数据库驱动（如 R 2 DBC、MongoDB Reactive）。  
3. **监控与诊断**：集成 Micrometer + Prometheus 监控线程、GC、连接池状态。