---
title:
  Tomcat HTTP 配置
tags:
  - literature-note
date:
  2025-02-28
time:
  18:59
aliases:
---
---

### **Tomcat 性能调优参数详解**

Tomcat 的性能调优参数主要集中在 `server.xml` 的 `Connector` 配置中。以下是关键参数及其作用原理：

#### **1. `maxThreads`**
- **作用**：定义 Tomcat 能同时处理请求的最大线程数。每个请求由一个线程处理，线程数直接影响并发能力。
- **默认值**：200。
- **调优依据**：
  - **CPU 核心数**：线程数不宜超过 CPU 核心数的 2~4 倍（例如 16 核服务器建议 `200~500`）。
  - **内存限制**：每个线程占用约 1 MB 栈内存，500 线程约需 500 MB，需确保 JVM 堆外内存充足。
  - **响应时间（RT）**：根据 **Little’s Law**，`并发数 = QPS × RT`。若 QPS=10000，RT=1 s，理论并发数需支持 10000。但线程数过高会导致上下文切换开销，需折中。

#### **2. `acceptCount`**
- **作用**：当所有线程繁忙时，新请求进入等待队列的最大长度。队列满后，新连接会被拒绝（返回 503）。
- **默认值**：100。
- **调优依据**：
  - 队列长度需平衡延迟和吞吐量：队列过长会增加平均响应时间，过短会丢弃请求。
  - 建议设为 `maxThreads` 的 50%~100%（如 `maxThreads=500`，则 `acceptCount=250~500`）。

#### **3. `maxConnections`**
- **作用**：Tomcat 同时保持的 TCP 连接数上限（包括活跃和空闲连接）。
- **默认值**：NIO 模式为 10000，BIO 模式同 `maxThreads`。
- **调优依据**：
  - 需大于 `maxThreads + acceptCount`，防止连接被过早拒绝。
  - 高并发场景建议设为 `10000`，但需监控系统文件描述符限制（`ulimit -n`）。

#### **4. `connectionTimeout`**
- **作用**：客户端建立连接后，等待请求数据的超时时间（毫秒）。
- **默认值**：20000（20 秒）。
- **调优依据**：
  - 防止慢客户端占用连接资源，建议缩短为 `5000~10000`（5~10 秒）。
  - 若业务涉及大文件上传，需单独调整 `connectionUploadTimeout`。

#### **5. 压缩与 Keep-Alive**
- **压缩配置**：
  ```xml
  compression="on" 
  compressionMinSize="1024" 
  compressableMimeType="text/html,text/css,application/json"
  ```
  - **依据**：减少文本类数据体积（节省 50%+ 带宽），但图片/视频无需压缩。
- **Keep-Alive**：
  ```xml
  keepAliveTimeout="30000" 
  maxKeepAliveRequests="100"
  ```
  - **依据**：复用 TCP 连接减少握手开销，但需避免长连接耗尽线程。

---

### **QPS=10000、RT=1 秒的配置方案**

#### **1. 核心参数计算**
- **并发需求**：根据 Little’s Law，`并发数 = QPS × RT = 10000 × 1 = 10000`。
- **线程池配置**：
  - **`maxThreads`**：直接设为 10000 会导致线程切换灾难。需根据服务器硬件调整：
    - **CPU 核心数**：假设 16 核，建议 `maxThreads=500`（经验值）。
    - **内存限制**：500 线程约需 500 MB 栈内存（JVM 参数 `-Xss1m`）。
  - **`acceptCount`**：设为 `500`（与 `maxThreads` 相同），队列总容量为 `500 + 500 = 1000`，但需结合实际情况。
  - **`maxConnections`**：设为 `10000`（默认值），确保能容纳突发流量。

- **矛盾点**：理论并发需求（10000）远大于 `maxThreads`（500），需通过 **异步处理** 或 **横向扩展** 解决：
  - **异步 Servlet**：将阻塞操作（如数据库查询）异步化，释放线程。
  - **集群部署**：通过多节点分担负载（如 20 个节点，每个节点 QPS=500）。

#### **2. 完整配置示例**
```xml
<Connector 
    port="80" 
    protocol="org.apache.coyote.http11.Http11Nio2Protocol"
    connectionTimeout="5000"
    redirectPort="8443"
    maxThreads="500" 
    acceptCount="500" 
    maxConnections="10000"
    keepAliveTimeout="15000"
    maxKeepAliveRequests="100"
    compression="on"
    compressionMinSize="1024"
    compressableMimeType="text/html,text/css,application/json"
    server="Unknown"  <!-- 隐藏Tomcat版本 -->
    enableLookups="false"  <!-- 禁用DNS反向查询 -->
    URIEncoding="UTF-8"
/>
```

#### **3. 配置理由**
- **协议选择**：`Http11Nio2Protocol`（NIO 2 异步模式），比 NIO 更高效。
- **线程模型**：`maxThreads=500` 平衡 CPU 核心数和内存限制，避免上下文切换开销。
- **队列容量**：`acceptCount=500` 提供缓冲，防止瞬间流量导致拒绝连接。
- **超时控制**：`connectionTimeout=5000` 避免慢连接占用资源。
- **Keep-Alive**：`keepAliveTimeout=15000` 平衡连接复用与资源释放。
- **压缩**：针对文本内容启用压缩，减少带宽消耗。

#### **4. 辅助优化措施**
- **JVM 调优**：
  ```bash
  -Xms4g -Xmx4g  # 堆内存设为 4GB（根据物理内存调整）
  -XX:+UseG1GC    # 启用 G1 垃圾回收器，减少停顿
  -Xss256k        # 减少线程栈大小（需测试稳定性）
  ```
- **操作系统调优**：
  - 文件描述符限制：`ulimit -n 100000`
  - TCP 参数优化（`/etc/sysctl.conf`）：
    ```conf
    net.core.somaxconn=65535
    net.ipv4.tcp_max_syn_backlog=65535
    net.ipv4.tcp_tw_reuse=1
    ```
- **监控与压测**：
  - 使用 **JMeter** 或 **Gatling** 模拟 10000 QPS，观察线程池状态（`busy` / `idle`）。
  - 监控 GC 日志和 `Tomcat 线程池` 指标（如 Prometheus + Grafana）。

#### **5. 横向扩展方案**
若单机无法满足需求，需通过集群解决：
- **负载均衡**：使用 Nginx 或 HAProxy 分发请求到多个 Tomcat 节点。
- **会话共享**：通过 Redis 存储 Session，实现无状态集群。
- **静态资源分离**：将图片/CSS/JS 托管到 CDN 或 Nginx，减轻 Tomcat 压力。

---

### **总结**
- **单机极限**：Tomcat 单节点在 500 线程下，实际可支撑约 500 QPS（RT=1 s）。要满足 10000 QPS，需至少 20 个节点（或通过异步化提升单机吞吐量）。
- **核心原则**：避免线程阻塞、减少锁竞争、优化 I/O 效率。
- **最终建议**：先通过 **异步处理** 和 **JVM 调优** 提升单机性能，再通过 **水平扩展** 实现万级 QPS。