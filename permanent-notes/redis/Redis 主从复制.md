---
title: Redis 主从复制
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-06
time: 17:25
aliases:
---
# 1 主从复制概述

## 1.1 主从复制的核心价值

## 1.2 典型应用场景

## 1.3 Redis 主从复制特性演进

### 1.3.1 Redis 2.8 前的 `SYNC` 机制

### 1.3.2 `PSYNC` 协议演进

### 1.3.3 Redis 4.0 的优化改造

# 2 架构设计解析
## 2.1 角色定义
### 2.1.1 主节点职责

### 2.1.2 从节点职责

## 2.2 同步类型

### 2.2.1 全量同步

### 2.2.2 部分同步

## 2.3 复制元数据

### 2.3.1 Replication ID 机制
### 2.3.2 Offset 偏移量系统
### 2.3.3 复制积压缓冲区 (Replication Backlog)

# 3 同步流程深度剖析
## 3.1 连接建立阶段

### 3.1.1 从节点配置解析

### 3.1.2 主从握手协议
### 3.1.3 身份验证流程
## 3.2 全量同步流程
### 3.2.1 RDB 文件生成过程
### 3.2.2 RDB 传输协议
### 3.2.3 从节点数据加载策略
### 3.2.4 复制缓冲区管理
## 3.3 增量同步流程
### 3.3.1 PSYNC 命令工作原理
### 3.3.2 命令传播协议
### 3.3.3 实时数据一致性保障
## 3.4 心跳检测机制
### 3.4.1 REPLCONF ACK 机制
### 3.4.2 网络延迟检测算法

# 4 配置与管理实践
## 4.1 主从配置详解
### 4.1.1 redis.conf 关键参数
### 4.1.2 运行时配置命令
## 4.2 状态监控
### 4.2.1 INFO replication 输出解析
### 4.2.2 ROLE 命令深度解读
### 4.2.3 关键监控指标
## 4.3 故障处理
### 4.3.1 网络中断恢复策略
### 4.3.2 主节点切换场景处理
### 4.3.3 数据一致性校验方法
# 5 高级主题
## 5.1 读写分离架构
### 5.1.1 流量分发策略
### 5.1.2 读写一致性控制
## 5.2 级联复制 (树状复制)
### 5.2.1 链式复制配置
### 5.2.2 延迟优化策略
## 5.3 主从切换机制
### 5.3.1 手动故障转移流程
### 5.3.2 与 Sentinel 的集成
## 5.4 磁盘复制模式
### 5.4.1 disk-backed 与 diskless 对比
### 5.4.2 无盘复制优化
# 6 性能优化与调优
## 6.1 同步性能瓶颈分析
## 6.2 关键参数调优
## 6.3 持久化策略选择
## 6.4 网络优化策略
# 7 异常场景处理
## 7.1 主从数据不一致检测
## 7.2 脑裂问题处理
## 7.3 复制风暴预防
## 7.4 大Key同步优化
## 7.5 版本兼容性问题
# 8 安全增强
## 8.1 TLS 加密传输
## 8.2 ACL 访问控制
## 8.3 敏感操作保护
# 9 内部实现解析
## 9.1 复制状态机实现
## 9.2 异步复制管道
## 9.3 内存管理策略
## 9.4 源码关键逻辑分析
# 10 最佳实践
## 10.1 容量规划建议
## 10.2 跨机房复制策略
## 10.3 监控报警方案
## 10.4 灾备演练流程
# 11 版本演进与未来方向
## 11.1 Redis 5.0 改进
## 11.2 Redis 6.0 多线程复制
## 11.3 Redis 7.0 新特性
## 11.4 社区发展方向





# 12 Redis 主从复制机制

Redis 提供了主从模式，采用读写分离的方式：
* 读操作：主从都接受
* 写操作：主库执行，从库同步



## 12.1 Redis 主从复制工作流程

| 阶段   | 主节点                                           | 从节点                                       | 描述                                                        |
| ---- | --------------------------------------------- | ----------------------------------------- | --------------------------------------------------------- |
| 配置阶段 |                                               | 配置主节点                                     | 比如使用 `replicaof` 命令                                       |
|      |                                               | 建立连接                                      |                                                           |
|      |                                               | 身份验证                                      |                                                           |
| 全量同步 |                                               | `PSYNC ? -1`                              | `PSYNC 主库Run ID offset`                                   |
|      | `+FULLRESYNC {Master Run id} {Master offset}` |                                           |                                                           |
|      |                                               | 保存主节点信息                                   |                                                           |
|      | 生成 RDB，传输 RDB                                 |                                           | 根据配置 `repl-diskless-sync yes` 是否开启无盘复制                    |
|      | 新的命令写入 `replication buffer`                   |                                           | 是 `client buffer`，可由配置 `client-output-buffer-limit` 设置最大值 |
|      |                                               | 清空 DB，加载 RDB                              |                                                           |
|      | 传输 RDB 完成后，发送 `replication buffer` 的命令        |                                           |                                                           |
|      |                                               | 接受命令，写入 DB                                |                                                           |
| 增量同步 |                                               | 断开连接                                      |                                                           |
|      | 写入新的命令到 `repl backlog buffer`                 |                                           | 环形缓冲区，主节点会记录自己的 offset，这个缓冲区的大小配置`repl-backlog-size 1mb`  |
|      |                                               | `PSYNC {Master run id} {Replica  offset}` | 从节点重新连接，发送从节点的 offset                                     |
|      | 比较 `master_repl_offset` 和 `slave_repl_offset` |                                           |                                                           |
|      | 如果在缓冲区内，发送区间命令                                |                                           |                                                           |
|      | 如果不在缓冲区内，开启全量同步                               |                                           |                                                           |
## 12.2 Redis 主从复制如何配置？

### 12.2.1 使用命令配置

执行命令：

```shell
replicaof masterip masterport
```

### 12.2.2 通过配置文件配置

```shell
# Master-Replica replication. Use replicaof to make a Redis instance a copy of
# another Redis server. A few things to understand ASAP about Redis replication.
#
#   +------------------+      +---------------+
#   |      Master      | ---> |    Replica    |
#   | (receive writes) |      |  (exact copy) |
#   +------------------+      +---------------+
#
# 1) Redis replication is asynchronous, but you can configure a master to
#    stop accepting writes if it appears to be not connected with at least
#    a given number of replicas.
# 2) Redis replicas are able to perform a partial resynchronization with the
#    master if the replication link is lost for a relatively small amount of
#    time. You may want to configure the replication backlog size (see the next
#    sections of this file) with a sensible value depending on your needs.
# 3) Replication is automatic and does not need user intervention. After a
#    network partition replicas automatically try to reconnect to masters
#    and resynchronize with them.
#
# replicaof <masterip> <masterport>
```

### 12.2.3 通过启动参数配置

启动参数 `--replicaof`




# 13 Reference
* [Redis replication \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
* [06 \| 数据同步：主从库如何实现数据一致？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/272852)
* [32 \| Redis主从同步与故障切换，有哪些坑？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/303247)
* [33 \| 脑裂：一次奇怪的数据丢失-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/303568)
* [INFO \| Docs](https://redis.io/docs/latest/commands/info/)