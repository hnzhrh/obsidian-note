---
title: Redis 数据对象之 List
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-04
time: 08:52
aliases:
  - Redis List Object
---


```plantuml

@startmindmap
!theme cerulean
* List
	* LISTPACK
	* QUICKLIST
	* ~~ZIPLIST~~
	* ~~LINKEDLIST~~
@endmindmap
```

Redis 最初使用 `linkedlist` 实现列表，3.2 引入了 `quicklist`，Redis 6.2 版本以后，正式废弃了 `linkedlist` 的实现。

`linkedlist` 为数据结构学习中最常见的双端链表来实现：

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

`quicklist` 是 Redis 从 3.2 版本开始引入的一种优化过的 List 数据类型的实现。它旨在结合 `linkedlist` 和 `ziplist`（7.0 以后为 `listpack`）的优点，提供更好的性能和内存效率。

QuickList 的实现方式：

1. **双向链表结构**：
   - `quicklist` 是一个双向链表，每个节点（`quicklistNode`）包含一个指向下一个节点和前一个节点的指针。
   
2. **节点内的紧凑存储**：
   - 每个 `quicklistNode` 内部使用 `listpack`（以前称为 `ziplist`）来存储多个连续的小型元素。`listpack` 是一种紧凑的内存表示形式，可以高效地存储小对象，并减少内存碎片。

3. **动态调整**：
   - 当 `quicklist` 中的某个节点（即 `listpack`）超出了一定大小或元素数量时，Redis 会自动将该节点拆分为两个新的节点，以保持每个节点内的元素数量在合理的范围内。这有助于维持操作的时间复杂度接近常数时间。

# 1 `ZIPLIST` 编码


# 2 LISTPACK 编码

Listpack 也叫紧凑列表，它的特点就是用一块连续的内存空间来紧凑地保存数据，同时为了节省内存空间，listpack 列表项使用了多种编码方式，来表示不同长度的数据，这些数据包括整数和字符串。

![image.png](https://images.hnzhrh.com/note/20241214190629.png)



# 3 Reference
* [Redis源码-List：Redis List存储原理、Redis List命令、 Redis List存储底层编码quicklist、Redis List应用场景\_redis的list源码-CSDN博客](https://blog.csdn.net/qq_41929714/article/details/126342953)
* [Redis Ziplist Data Structure. Redis is an in-memory storage server… \| by Shubhi Jain \| InterviewNoodle](https://interviewnoodle.com/redis-ziplist-data-structure-23c8e7e3266d)
* [listpack/listpack.md at master · antirez/listpack · GitHub](https://github.com/antirez/listpack/blob/master/listpack.md)
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)