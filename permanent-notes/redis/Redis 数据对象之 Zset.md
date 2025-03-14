---
title: Redis 数据对象之 Zset
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-14
time: 16:57
aliases: 
done: false
---
# 数据结构

`zset` 由一个哈希表和一个跳表构成，哈希表主要是为了 `O（1）` 获取 `score`，跳表实现了有序的数据集。

```c
typedef struct zset {  
    dict *dict;  
    zskiplist *zsl;  
} zset;
```

# Reference