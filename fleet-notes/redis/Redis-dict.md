---
title: Redis-dict
tags:
  - fleet-note
date: 2024-11-26
time: 19:01
aliases:
---


`dict` 数据结构是 Redis 核心结构之一，为 Redis 提供了 `O(1)` 的 `kv` 访问。

```c
// server.h
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    kvstore *keys;              /* The keyspace for this DB */
    kvstore *expires;           /* Timeout of keys with a timeout set */
    ebuckets hexpires;          /* Hash expiration DS. Single TTL per hash (of next min field to expire) */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *blocking_keys_unblock_on_nokey;   /* Keys with clients waiting for
                                             * data, and should be unblocked if key is deleted (XREADEDGROUP).
                                             * This is a subset of blocking_keys*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

`dict` 的定义：
```c
// dict.h
struct dict {
    dictType *type;

    dictEntry **ht_table[2];
    unsigned long ht_used[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    unsigned pauserehash : 15; /* If >0 rehashing is paused */

    unsigned useStoredKeyApi : 1; /* See comment of storedHashFunction above */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
    int16_t pauseAutoResize;  /* If >0 automatic resizing is disallowed (<0 indicates coding error) */
    void *metadata[];
};

// dict.c

/* -------------------------- types ----------------------------------------- */
struct dictEntry {
    void *key;
    union {
        void *val; // 节省内存技巧，如果存储的是整数和浮点数，就不需要额外一个指针了
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
};
```


`dict` 扩容时，会创建大于当前大小的 2 的最小幂次方，再通过 rehash 机制减少数据拷贝带来的性能损耗。

执行扩容：
![image.png](https://images.hnzhrh.com/note/20241126200841.png)
执行 rehash
![image.png](https://images.hnzhrh.com/note/20241126201227.png)
Redis dict rehash 是渐进式的，发生在：
* 增删改查时顺带迁移一个 rehashidx 位置的数据
* 定时任务调度
# References
* [美团针对Redis Rehash机制的探索和实践 - 美团技术团队](https://tech.meituan.com/2018/07/27/redis-rehash-practice-optimization.html)


