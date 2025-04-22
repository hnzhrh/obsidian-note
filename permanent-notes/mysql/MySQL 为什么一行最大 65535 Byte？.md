---
title: MySQL 为什么一行最大 65535 Byte？
tags:
  - permanent-note
  - mysql/qa
date: 2025-04-11
time: 13:02
aliases: 
done: false
---


MySQL 服务器层使用 2 字节存储行长度，理论最大 65,535 字节

一页 16 KB，如果一行数据大于了 16 KB 的一半，则会发生行分裂，通过指针指向下一个数据地址。

MySQL B+ 树节点最少要能存储两个数据，否则会退化成低效链表




# Reference
* [了解 MySQL的数据行、行溢出机制吗？ - 赐我白日梦 - 博客园](https://www.cnblogs.com/ZhuChangwu/p/14035330.html)
* [MySQL 一行记录是怎么存储的？ \| 小林coding](https://xiaolincoding.com/mysql/base/row_format.html)