---
title: Redis 数据对象之 String
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-04
time: 08:48
aliases:
  - Redis String Object
---
# 1 为什么不使用 char 数组？

为什么 Redis 不直接使用 `char*` 呢？

`char*` 的不足：
- 操作效率低：获取长度需遍历，`O(N)` 复杂度
- 二进制不安全：无法存储包含 `\0` 的数据

SDS 的优势：
- 操作效率高：获取长度无需遍历，`O(1)` 复杂度
- 二进制安全：因单独记录长度字段，所以可存储包含 `\0` 的数据
- 兼容 C 字符串函数，可直接使用字符串 API

Redis String 类型有三种编码方式：
* INT 
* EMBSTR
* RAW

# 2 Redis 数据对象 String 编码方式

## 2.1 [Redis Object Encoding INT](Redis%20Object%20Encoding%20INT.md)
## 2.2 [Redis Object Encoding EMBSTR](Redis%20Object%20Encoding%20EMBSTR.md)
## 2.3 [Redis Object Encoding RAW](Redis%20Object%20Encoding%20RAW.md)

# 3 `SDS` 数据结构

Redis 没有使用 C 的字符串作为 Redis 字符串的实现，而是使用了 `SDS` 的数据结构，Simple Dynamic Sring。

针对不同的字符串长度，为了更加高效得使用内存，Redis 定义了多个 SDS 用来表示不同长度的字符串：
* `sdshdr5`，不会在 Value 中使用，只会在 Key 中使用。
* `sdshdr8`
* `sdshdr16`
* `sdshdr32`
* `sdshdr64`

源码如下：
```c
//sds.h
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
// 其他类似
```

## 3.1 使用 SDS 有什么好处？

* `O(1)` 获取字符串长度
* 记录了内存大小和字符串长度，在类似拼接字符串等操作时，不会造成缓冲区溢出
* 内存空间预分配可以减少内存分配次数，但同时也带来了内存碎片的问题
* 兼容 Linux C 字符串函数

## 3.2 预留空间分配逻辑

当需要扩容时，Redis 在给 SDS 分配内存时会采用预留空间机制，具体来说当字符串大小小于 `SDS_MAX_PREALLOC` （`#define SDS_MAX_PREALLOC (1024*1024)` 1 M）时分配两倍，大于 `SDS_MAX_PREALLOC` 分配 `SDS_MAX_PREALLOC` 大小。

例如 Append 字符串时，扩容的大小为（原字符串 Length + Append 字符串 Length）* 2。

源码可参考方法：`sdsMakeRoomFor`

![image.png](https://images.hnzhrh.com/note/20241212165917.png)

同样，既然有扩容，那肯定有缩容，当判断到可用空间大于字符串长度的十分之一时，会进行缩容操作。

源代码：

```c
/* Optimize the SDS string inside the string object to require little space,
 * in case there is more than 10% of free space at the end of the SDS. */
void trimStringObjectIfNeeded(robj *o, int trim_small_values) {
    if (o->encoding != OBJ_ENCODING_RAW) return;
    /* A string may have free space in the following cases:
     * 1. When an arg len is greater than PROTO_MBULK_BIG_ARG the query buffer may be used directly as the SDS string.
     * 2. When utilizing the argument caching mechanism in Lua. 
     * 3. When calling from RM_TrimStringAllocation (trim_small_values is true). */
    size_t len = sdslen(o->ptr);
    if (len >= PROTO_MBULK_BIG_ARG ||
        trim_small_values||
        (server.executing_client && server.executing_client->flags & CLIENT_SCRIPT && len < LUA_CMD_OBJCACHE_MAX_LEN)) {
        if (sdsavail(o->ptr) > len/10) {
            o->ptr = sdsRemoveFreeSpace(o->ptr, 0);
        }
    }
}

```

![image.png](https://images.hnzhrh.com/note/20241212170515.png)
# 4 字符串能有多大？
Redis 最大字符串能有多大？Redis 6.0 版本之前，写死在代码里面的，512 MB，6.0 版本之后可以根据配置进行修改，最小为 1 MB。
```properties
# In the Redis protocol, bulk requests, that are, elements representing single
# strings, are normally limited to 512 mb. However you can change this limit
# here, but must be 1mb or greater
#
# proto-max-bulk-len 512mb
```
# 5 References
* [How is the memory usage for the key-value calculated? · redis/redis · Discussion #13677 · GitHub](https://github.com/redis/redis/discussions/13677)
* [Analyzing Redis Source Code: Simple Dynamic Strings (SDS) – An Efficient and Flexible String Implementation \| Johnson Lin](https://www.linjiangxiong.com/2024/09/10/analyzing-redis-source-code-sds/index.html)
* [🚀深入理解redis的简单动态字符串（SDS）🚀Redis是一款流行的高性能键值存储数据库，而简单动态字符串SDS是 - 掘金](https://juejin.cn/post/7304183129896173568)
* [Redis Strings \| Docs](https://redis.io/docs/latest/develop/data-types/strings/)
* [Redis源码-String：Redis String命令、Redis String存储原理、Redis String三种编码类型、Redis字符串SDS源码解析、Redis String应用场景\_redis的string存储原理-CSDN博客](https://blog.csdn.net/qq_41929714/article/details/126060599)
* [04 \| 内存友好的数据结构该如何细化设计？-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/402223)