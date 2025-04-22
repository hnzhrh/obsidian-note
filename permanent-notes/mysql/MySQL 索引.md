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

# MySQL 索引失效

* 左模糊匹配或者中间模糊匹配
* 对索引列进行了计算、函数、类型转换
* 联合索引没有匹配最左匹配原则
* WHERE 子句中，OR 前是索引列，但是 OR 后不是索引列

# 6 Reference
* [索引失效有哪些？ \| 小林coding](https://xiaolincoding.com/mysql/index/index_lose.html#%E7%B4%A2%E5%BC%95%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84%E9%95%BF%E4%BB%80%E4%B9%88%E6%A0%B7)