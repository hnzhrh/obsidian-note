---
title: Redis Object Encoding QUICKLIST
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 14:03
aliases:
---

# QUICKLIST 结构

Redis 3.2 版本引入QUICKLIST，实际上就是一个双向链表，链表中的每一个节点都是一个 ZIPLIST，如此当发生连锁更新时，只会影响到一个 ZIPLIST 节点。

Redis 5.0 版本引入 LISTPACK，直到 7.0 版本，QUICKLIST 的节点都替换成了 LISTPACK。

# QUICKLIST 优势

1. **内存高效**
    - 每个 `QUICKLIST` 节点是一个 `ziplist` 或者 `listpack`，存储多个元素，减少指针的内存开销（例如存储 1000 个小元素时，内存占用显著降低）。
    - 通过配置 `list-max-ziplist-size` 或者`list-max-listpack-size`，可以控制每个节点的大小，优化内存与性能的平衡。    
2. **高性能操作**
    - **两端操作高效**：`LPUSH` / `RPOP` 等操作直接作用于头尾的 `ziplist` 或者 `listpack` 节点，时间复杂度接近 O(1)。  
    - **中间操作优化**：通过限制单个节点的大小，减少插入/删除元素时的数据移动量，避免全量拷贝。
3. **减少内存碎片**
    - `QUICKLIST` 的节点是较大的连续内存块（`ziplist` 或者 `listpack`），相比 `linkedlist` 频繁分配小内存节点，更利于内存管理。
4. **灵活性与可配置性**
    - 支持压缩中间节点（通过 `list-compress-depth` 配置），进一步节省内存，适合读多写少的场景。

# 相关源码

```c
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: 0 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmarks are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all listpacks */
    unsigned long len;          /* number of quicklistNodes */
    signed int fill : QL_FILL_BITS;       /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;


/* Node, quicklist, and Iterator are the only data structures used currently. */

/* quicklistNode is a 32 byte struct describing a listpack for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max lp bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, PLAIN=1 (a single item as char array), PACKED=2 (listpack with multiple items).
 * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * dont_compress: 1 bit, boolean, used for preventing compression of entry.
 * extra: 9 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *entry;
    size_t sz;             /* entry size in bytes */
    unsigned int count : 16;     /* count of items in listpack */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* PLAIN==1 or PACKED==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int dont_compress : 1; /* prevent compression of entry that will be used later */
    unsigned int extra : 9; /* more bits to steal for future usage */
} quicklistNode;
```

# Reference
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)
* [Redis数据结构——快速列表(quicklist) - 随心所于 - 博客园](https://www.cnblogs.com/hunternet/p/12624691.html)