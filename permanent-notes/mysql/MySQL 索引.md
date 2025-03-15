---
title: MySQL 索引
tags:
  - permanent-note
  - middleware/database/mysql/index
date: 2025-03-13
time: 17:31
aliases: 
done: false
---
# 1 什么是索引？

# 2 索引的分类
## 2.1 按物理存储分类
* 聚簇索引 
* 非聚簇索引
## 2.2 按字段特性分类
* 主键索引（PRIMARY KEY）
* 唯一索引（UNIQUE）
* 普通索引（INDEX）
* 全文索引（FULLTEXT）
## 2.3 按字段个数分类
* 单列索引
* 联合索引
## 2.4 特殊索引分类
* 覆盖索引
* 降序索引
* 自适应哈希索引（AHI）
* 隐藏索引（MySQL 8.0+）
# 3 索引数据结构

# 4 索引创建与管理
## 4.1 创建语法进阶
- 多列索引顺序选择
- 前缀索引长度计算
- 函数索引（MySQL 8.0+）
## 4.2 索引维护策略
- OPTIMIZE TABLE 原理
- 在线 DDL 操作（ALGORITHM=INPLACE）
## 4.3 索引监控与诊断
- SHOW INDEX 关键指标解读
- INFORMATION_SCHEMA 统计表
- 执行计划 (EXPLAIN) 深度分析
# 5 索引优化策略
## 5.1 索引设计原则
## 5.2 索引失效场景
## 5.3 索引合并优化 （Index Merge）
## 5.4 索引下推 （ICP）
## 5.5 MRR 优化原理


# 6 Reference