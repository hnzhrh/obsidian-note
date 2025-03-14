---
title: Redis Object Encoding INTSET
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-14
time: 17:26
aliases: 
done: false
---
如果 `Set` 中全部都是整数，则 Redis 会使用 `INTSET` 编码方式，`INTSET` 数据结构是有序的整数数组，查找使用二分查找。

`INTSET` 如果新插入的数据编码高于当前编码，则会进行升级，升级会重新分配内存，并遍历所有元素完成编码转换。

```c
// intset.h
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;

// intset.c
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

# Reference