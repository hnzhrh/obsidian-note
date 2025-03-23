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

* 使用了左模糊或者左右模糊
* 索引使用了函数
* 索引表达式计算了
* 索引发生了隐式转换
* 联合索引最左匹配原则
* or 混油索引列和非索引列

# 6 Reference
* [索引失效有哪些？ \| 小林coding](https://xiaolincoding.com/mysql/index/index_lose.html#%E7%B4%A2%E5%BC%95%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84%E9%95%BF%E4%BB%80%E4%B9%88%E6%A0%B7)