---
title: ebay-pipeline-shcedule 资料
tags:
  - fleet-note
  - project/ebay/pipeline-schedule
date: 2025-02-16
time: 21:22
aliases:
---
# ChatGPT

## 去中心化定时任务

是的，**仅使用 MySQL** 也是可以实现去中心化的任务调度系统的，尽管 MySQL 并不是一个专门的分布式系统管理工具，它依然可以通过适当的设计来实现任务调度和分配，确保任务不会被重复执行，并且在多个工作节点之间进行有效的分配和协调。

### 设计思路

在使用 **MySQL** 实现去中心化的任务调度时，主要依靠以下几个关键机制：

1. **任务状态管理**：每个任务的状态会存储在 MySQL 数据库中，通过任务的状态字段来确保任务的执行顺序和唯一性。
2. **乐观锁**：使用乐观锁或排他性操作（如 `UPDATE` 语句结合版本号或时间戳）来确保任务不会被多个工作节点同时执行。
3. **任务队列**：通过设计合适的数据库表格来模拟任务队列，工作节点从任务表中读取任务，并在完成任务后更新任务的状态。
4. **任务重试机制**：任务执行失败后，需要标记任务的状态，并可能将任务重新入队，进行重试。
5. **心跳机制**：为了保证工作节点的健康，可以通过定时更新数据库中的心跳字段，来确定工作节点是否存活。

### 1. **任务表设计**

设计一个任务表来存储任务的相关信息，包括任务状态、任务的分配信息等。假设我们使用如下的表结构：

```sql
CREATE TABLE tasks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(255) NOT NULL,
    status ENUM('PENDING', 'IN_PROGRESS', 'COMPLETED', 'FAILED') DEFAULT 'PENDING',
    assigned_worker_id BIGINT DEFAULT NULL,  -- 记录任务被分配给哪个工作节点
    assigned_at TIMESTAMP NULL,              -- 任务被分配的时间
    started_at TIMESTAMP NULL,               -- 任务开始执行时间
    completed_at TIMESTAMP NULL,             -- 任务完成时间
    retries INT DEFAULT 0,                   -- 任务的重试次数
    cron_expression VARCHAR(255),            -- 定时任务的 Cron 表达式（可选）
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

- **`status`**：记录任务的状态，防止重复执行。
- **`assigned_worker_id`**：记录当前任务被分配给哪个工作节点。
- **`retries`**：记录任务的重试次数。
- **`created_at` 和 `updated_at`**：记录任务的创建和更新时间，用于调度时参考。

### 2. **任务获取与分配**

任务的分配可以通过 **乐观锁** 或 **基于时间戳的排他性** 更新来确保任务的唯一性。工作节点在获取任务时，会通过 `UPDATE` 语句来“锁定”任务，并检查任务的状态是否为“待分配”。

**示例：工作节点获取任务**

```java
public boolean acquireTask(Long workerId) {
    String sql = "UPDATE tasks SET status = 'IN_PROGRESS', assigned_worker_id = ?, assigned_at = NOW() " +
                 "WHERE status = 'PENDING' AND assigned_worker_id IS NULL LIMIT 1";
    int rowsUpdated = jdbcTemplate.update(sql, workerId);

    return rowsUpdated > 0;  // 如果返回大于0，表示成功获取任务
}
```

在这个操作中，工作节点通过 `UPDATE` 语句将 `status` 从 `PENDING` 更新为 `IN_PROGRESS`，并且 `assigned_worker_id` 被设置为当前工作节点的 ID。这个过程是 **原子性** 的，通过 MySQL 的 **行级锁**（在执行 `UPDATE` 时），确保只有一个工作节点能够获取到该任务。

### 3. **任务执行**

工作节点在执行任务时，会根据任务的状态更新数据库中的状态。

**示例：工作节点执行任务**

```java
public void executeTask(Long taskId) {
    // 先获取任务
    Task task = taskRepository.findById(taskId);
    if (task == null || !task.getStatus().equals(TaskStatus.IN_PROGRESS)) {
        return;
    }

    try {
        // 执行任务的具体逻辑
        System.out.println("Executing task: " + task.getTaskName());

        // 执行成功，更新任务状态为已完成
        task.setStatus(TaskStatus.COMPLETED);
        task.setCompletedAt(LocalDateTime.now());
    } catch (Exception e) {
        // 执行失败，更新任务状态为失败，并记录错误信息
        task.setStatus(TaskStatus.FAILED);
        task.setErrorMessage(e.getMessage());
        task.setRetries(task.getRetries() + 1);
    } finally {
        taskRepository.save(task);  // 保存任务状态
    }
}
```

### 4. **任务重试机制**

如果任务执行失败，通常需要重试。工作节点可以根据任务的失败次数（`retries` 字段）决定是否将任务重新入队。

**示例：任务失败重试**

```java
public void retryFailedTask(Long taskId) {
    Task task = taskRepository.findById(taskId);
    if (task == null || task.getStatus().equals(TaskStatus.COMPLETED)) {
        return;
    }

    if (task.getRetries() < MAX_RETRIES) {
        task.setStatus(TaskStatus.PENDING);  // 重新设置为待执行状态
        task.setAssignedWorkerId(null);      // 取消任务分配
        task.setAssignedAt(null);
        task.setRetries(task.getRetries() + 1);
        taskRepository.save(task);  // 保存重试次数
    }
}
```

任务失败后，重试次数会增加，如果重试次数超过最大次数（`MAX_RETRIES`），任务将标记为 `FAILED`，否则任务重新变为 `PENDING` 状态，等待下一个工作节点来执行。

### 5. **心跳机制与工作节点监控**

为了防止工作节点在执行任务时崩溃或失联，任务表可以设计一个 **心跳字段**。工作节点在处理任务时会定时更新自己的心跳状态。如果某个工作节点长时间没有更新心跳，可以将该节点分配的任务重新标记为待执行状态。

**示例：心跳更新**

```java
public void updateHeartbeat(Long workerId) {
    String sql = "UPDATE workers SET last_heartbeat = NOW() WHERE worker_id = ?";
    jdbcTemplate.update(sql, workerId);
}
```

### 6. **任务完成与清理**

工作节点执行完任务后，要及时更新任务的状态。如果任务执行成功，标记为 `COMPLETED`，如果失败则标记为 `FAILED`。定期清理已完成或失败的任务，以便数据库不被过多的无用数据填充。

### 7. **去中心化的调度与分配**

在这个设计中，每个工作节点都有能力自主获取任务并执行任务。因此，整个系统是去中心化的，没有一个专门的调度器来管理任务。每个节点在 **任务表** 中轮询待执行的任务，并通过 **乐观锁机制** 来确保任务不被重复执行。

### 8. **优势与限制**

#### 优势：

- **去中心化**：没有单一的调度器，每个工作节点都能独立获取任务并执行。
- **高可扩展性**：随着工作节点数量的增加，任务执行可以水平扩展，不会形成性能瓶颈。
- **简单可靠**：利用 MySQL 本身的事务性和一致性来管理任务状态，设计相对简单且可靠。

#### 限制：

- **MySQL性能瓶颈**：虽然 MySQL 可以很好地处理任务的状态管理，但在高并发的情况下，MySQL 的性能可能成为瓶颈，特别是对于任务查询和更新操作。
- **乐观锁冲突**：在高并发的环境下，乐观锁可能会频繁失败，导致任务被重试或无法成功分配。
- **任务排队延迟**：如果任务队列设计不好（例如，队列是全表扫描的形式），可能会导致任务分配和执行的延迟。

### 总结：

通过合理设计数据库结构、任务状态、乐观锁和重试机制，完全可以在 **MySQL** 中实现一个去中心化的任务调度系统。它适用于负载不太高、任务分配较为简单的场景。不过，对于高并发和高可靠性要求的场景，可能需要考虑使用 **消息队列** 或 **分布式协调工具**（如 Redis、ZooKeeper）来进一步优化任务调度和管理。


# Reference
* [SpringBoot 遇上状态机:简化复杂业务逻辑的利器 \| 小豆丁技术栈](http://www.mydlq.club/prepare/172/)
* [Weixin Official Accounts Platform](https://mp.weixin.qq.com/s?__biz=MzkwMTM4NTg5Ng==&mid=2247489361&idx=1&sn=3bb76b38af9e6bda271af286e054e27c&chksm=c0b4cc96f7c34580cace9e6679582a0812116b8d35ec79fa42f3c85f2cc49be4703ce5e29d79&scene=21#wechat_redirect)
* [GitHub - Leme34/stateMachine: 参考：高德打车通用可编排订单状态机引擎设计](https://github.com/Leme34/stateMachine)
* [GitHub - Childe-Chen/statemachine: 状态机学习](https://github.com/Childe-Chen/statemachine)
* [Tiny Job 一个轻量级定时任务调度系统Tiny Job 轻量级定时任务调度系统 主要技术选型 1、后端： Spri - 掘金](https://juejin.cn/post/7260526125847986213)
* [基于数据库版本的分布式定时任务调度中心构建一个统一的调度系统，用于触发定时任务的调度。2.2.（可使用开源项目框架，但目 - 掘金](https://juejin.cn/post/7152791076431429645)
* [Java 定时任务详解 \| JavaGuide](https://javaguide.cn/system-design/schedule-task.html)
