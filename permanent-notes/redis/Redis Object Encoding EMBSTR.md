---
title: Redis Object Encoding EMBSTR
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 15:40
aliases:
---
当字符串长度小于等于 44（`OBJ_ENCODING_EMBSTR_SIZE_LIMIT`） 时，会创建 `EMBSTR` 的 String 类型，redisObject 对象和 sds 对象一起分配，这种编码方式内存更加紧凑。

![image.png](https://images.hnzhrh.com/note/20241210222742.png)
`EMBSTR` 为只读字符串，当发生了修改，会变更为 `RAW` 编码。这里需要注意 `SET` 一个已有的值属于覆盖，而不属于修改。

```shell
127.0.0.1:6379> set name erpang
OK
127.0.0.1:6379> OBJECT ENCODING name
"embstr"
127.0.0.1:6379> APPEND name -shi
(integer) 10
127.0.0.1:6379> OBJECT ENCODING name
"raw"
127.0.0.1:6379> set name erpang
OK
127.0.0.1:6379> OBJECT ENCODING name
"embstr"
127.0.0.1:6379> 
```

# 为什么阈值是 44 呢？

首先需要知道 `jemalloc` 内存分配器是按照固定的块分配内存大小的，[Redis 对象](Redis%20对象.md) 占 16 Bytes，`SDS` 的字段占 3 Bytes，加上 `char` 数组的终止符 `\0` 占 1 Byte，$44 = 64 - 16 -3 -1$

另外在 Redis 版本 3.2 以前，`SDS` 是没有优化过的：
```c
struct sdshdr {
    unsigned int len;
    unsigned int free;
    char[] buf;
};
```

所以 3.2 以前版本阈值是 $39 = 64 - 16 - 4 -4 - 1$
# Reference