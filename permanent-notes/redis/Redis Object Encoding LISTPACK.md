---
title: Redis Object Encoding LISTPACK
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 14:04
aliases:
---
# LISTPACK 结构

* `lpbytes` 整个结构的占用的字节大小，包括头部和尾部
* `lplen` 整个 LISTPACK 的元素数量，2 个字节，`O(1)`  下最大 65535 个元素，如果超过了这个大小，则需要遍历获取元素数量了
* `lpend` 表示尾部的 `LP_EOF`，值为 `0xFF`

![image.png](https://images.hnzhrh.com/note/20250304150726097.png)


# 相关源码

```c
/* Each entry in the listpack is either a string or an integer. */
typedef struct {
    /* When string is used, it is provided with the length (slen). */
    unsigned char *sval;
    uint32_t slen;
    /* When integer is used, 'sval' is NULL, and lval holds the value. */
    long long lval;
} listpackEntry;


#define LP_HDR_SIZE 6       /* 32 bit total len + 16 bit number of elements. */


/* Create a new, empty listpack.
 * On success the new listpack is returned, otherwise an error is returned.
 * Pre-allocate at least `capacity` bytes of memory,
 * over-allocated memory can be shrunk by `lpShrinkToFit`.
 * */
unsigned char *lpNew(size_t capacity) {
    unsigned char *lp = lp_malloc(capacity > LP_HDR_SIZE+1 ? capacity : LP_HDR_SIZE+1);
    if (lp == NULL) return NULL;
    lpSetTotalBytes(lp,LP_HDR_SIZE+1);
    lpSetNumElements(lp,0);
    lp[LP_HDR_SIZE] = LP_EOF;
    return lp;
}
```

# Reference
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)
* [Site Unreachable](https://zhuanlan.zhihu.com/p/669544722)