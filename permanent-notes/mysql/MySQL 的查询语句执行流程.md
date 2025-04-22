---
title: MySQL 的查询语句执行流程
tags:
  - permanent-note
  - mysql/qa
date: 2025-04-11
time: 12:35
aliases: 
done: false
---

*  连接器：建立连接，校验身份
* 查询缓存：查询语句如果命中查询缓存则直接返回，否则继续，8.0 已经删除了 Query Cache
* 解析 SQL：词法、语法分析，构建语法树，如果有语法错误会抛出错误
* 执行 SQL
	* 预处理：检查表、字段是否存在，把 `select *` 扩展到所有列字段
	* 优化阶段：优化器基于查询成本选择执行计划
	* 执行阶段：根据执行计划执行 SQL，从存储引擎读取记录返回给 Server
![image.png](https://images.hnzhrh.com/note/20250411123904263.png)

# Reference
* [执行一条 select 语句，期间发生了什么？ \| 小林coding](https://xiaolincoding.com/mysql/base/how_select.html#%E6%89%A7%E8%A1%8C%E5%99%A8)