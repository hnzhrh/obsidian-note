---
title: MySQL 的日志
tags:
  - permanent-note
  - middleware/database/mysql/log
date: 2025-03-13
time: 18:47
aliases: 
done: false
---

# 大纲

## 1. 日志系统概述
### 1.1 日志在数据库系统中的作用
### 1.2 MySQL 日志体系架构
### 1.3 日志类型分类标准

## 2. 错误日志 (Error Log)
### 2.1 核心功能与记录内容
#### 2.1.1 启动/关闭事件
#### 2.1.2 运行错误跟踪
#### 2.1.3 存储引擎异常
### 2.2 配置参数详解
#### 2.2.1 log_error
#### 2.2.2 log_error_verbosity
#### 2.2.3 log_error_suppression_list
### 2.3 日志轮转与维护策略
### 2.4 故障诊断案例分析

## 3. 通用查询日志 (General Query Log)
### 3.1 工作机理与应用场景
### 3.2 参数配置
#### 3.2.1 general_log
#### 3.2.2 general_log_file
#### 3.2.3 log_output
### 3.3 性能影响与优化建议
### 3.4 日志分析工具实践

## 4. 慢查询日志 (Slow Query Log)
### 4.1 阈值设定策略
#### 4.1.1 long_query_time
#### 4.1.2 log_queries_not_using_indexes
#### 4.1.3 log_slow_admin_statements
### 4.2 日志格式解析
#### 4.2.1 Query_time 分析
#### 4.2.2 Lock_time 解读
#### 4.2.3 Rows_examined 优化
### 4.3 性能分析工具
#### 4.3.1 mysqldumpslow
#### 4.3.2 pt-query-digest

## 5. 二进制日志 (Binary Log)
### 5.1 复制与恢复核心机制
### 5.2 日志格式对比
#### 5.2.1 STATEMENT
#### 5.2.2 ROW
#### 5.2.3 MIXED
### 5.3 高级配置参数
#### 5.3.1 binlog_format
#### 5.3.2 sync_binlog
#### 5.3.3 expire_logs_days
### 5.4 日志管理操作
#### 5.4.1 PURGE BINARY LOGS
#### 5.4.2 mysqlbinlog 工具
### 5.5 GTID 实现原理

## 6. 中继日志 (Relay Log)
### 6.1 主从复制中的角色
### 6.2 与 Binary Log 的差异
### 6.3 日志应用流程
#### 6.3.1 SQL 线程工作机制
#### 6.3.2 复制延迟分析

## 7. 重做日志 (Redo Log)
### 7.1 InnoDB 持久性保证
### 7.2 日志结构解析
#### 7.2.1 LSN (Log Sequence Number)
#### 7.2.2 Checkpoint 机制
### 7.3 配置优化
#### 7.3.1 innodb_log_file_size
#### 7.3.2 innodb_log_files_in_group
#### 7.3.3 innodb_flush_log_at_trx_commit

## 8. 撤销日志 (Undo Log)
### 8.1 事务回滚与 MVCC 实现
### 8.2 日志存储管理
#### 8.2.1 undo tablespace
#### 8.2.2 版本链管理
### 8.3 参数调优
#### 8.3.1 innodb_undo_directory
#### 8.3.2 innodb_undo_tablespaces

## 9. 日志系统监控与优化
### 9.1 性能指标监控
#### 9.1.1 日志写入吞吐量
#### 9.1.2 磁盘 I/O 分析
### 9.2 安全审计实践
### 9.3 云环境下的日志管理

## 10. 事务日志协同工作流程
### 10.1 事务提交完整过程
### 10.2 崩溃恢复机制
#### 10.2.1 Redo 前滚过程
#### 10.2.2 Undo 回滚过程

## 11. 日志系统最佳实践
### 11.1 生产环境配置建议
### 11.2 日志备份策略
### 11.3 多日志系统协同分析

## 附录：日志类型速查表
| 日志类型       | 存储引擎   | 主要功能                  | 关键参数                 |
|----------------|------------|---------------------------|--------------------------|
| Error Log      | Server     | 错误追踪                  | log_error                |
| General Query  | Server     | 查询审计                  | general_log              |
| Slow Query     | Server     | 性能优化                  | long_query_time          |
| Binary Log     | Server     | 复制/恢复                 | binlog_format            |
| Redo Log       | InnoDB     | 事务持久性                | innodb_log_file_size     |
| Undo Log       | InnoDB     | MVCC/回滚                 | innodb_undo_tablespaces  |
| Relay Log      | Server     | 从库复制                  | relay_log                |

## 参考资料
- MySQL 8.0 Official Documentation
- InnoDB Internals Manual
- Percona Performance Blog

# Reference