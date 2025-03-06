---
title: Redis Object Encoding LINKEDLIST
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 15:20
aliases:
---
`LINKEDLIST` 为数据结构学习中最常见的双端链表来实现：

```c

/* Node, List, and Iterator are the only data structures used currently. */

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
# Reference