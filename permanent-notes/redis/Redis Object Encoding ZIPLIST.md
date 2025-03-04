---
title: Redis Object Encoding ZIPLIST
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-04
time: 13:22
aliases:
---
# 1 ZIPLIST 结构

`ZIPLIST` 设计成一种内存高效的数据结构，可以在两端 `O(1)` 操作，但其他的操作需要对 `ZIPLIST` 重新分配内存，所以整体复杂度取决于 `ZIPLIST` 的大小。 

内存结构如下：
![image.png](https://images.hnzhrh.com/note/20241214153120.png)

* `zlbytes`，表示 `ZIPLIST` 占用的总字节数
* `zltail`，表示 `ZIPLIST` 最后一个 `entry` 的偏移量，通过 `zltail` 可以 `O(1)` `POP` 操作，不需要遍历整个 `ZIPLIST`
* `zllen`，表示 `ZIPLIST` 中 `entry` 的个数，当 `ZIPLIST` 中 `entry` 个数超过了 2<sup>16</sup> -2 个，则会被设置为 2<sup>16</sup>-1，这是一个特殊标记符，表示 `ZIPLIST` 中的元素个数需要通过遍历才能获得
* `zlend`，特殊的记号表示为 `ZIPLIST` 的结尾，值为 `255`（`OxFF`）

# 2 ZIPLIST Entry 结构

`ZIPLIST` `entry` 有相当多的编码方式，大致分为两类：
* `<prevlen> <encoding> <entry-data>`
* `<prevlen> <encoding>` , 对于小整数，可以不使用 `<entry-data>` 部分

其中 `<prevlen>` 部分可能是 1 Byte 也可能是 5 Bytes，取决于前一个 `entry` 的大小，当前一个 `entry` 大小大于等于 254 时，使用 `0xFE + 4 Bytes len` 来标记。
![image.png](https://images.hnzhrh.com/note/20241214155939.png)

# 3 连锁更新问题

`<prevlen>` 的存在可能会引起连锁更新问题，假设多个 `entry` 大小都在 253，如果第一个变更了内容，`prelen` 从 1 Byte 膨胀到了 5 Bytes，之后的多个连续 `entry` 恰好都需要膨胀，则会出现连锁更新问题。

Redis 3.2 版本引入了 [Redis Object Encoding QUICKLIST](Redis%20Object%20Encoding%20QUICKLIST.md) 编码方式降低连锁更新的影响。
Redis 5.0 版本引入了 [Redis Object Encoding LISTPACK](Redis%20Object%20Encoding%20LISTPACK.md) 编码方式解决了连锁更新。
Redis 7.0 版本彻底废除了 ZIPLIST 编码方式。

# 4 各种类型的编码具体参考注释 

```c
/* 
 * ZIPLIST ENTRIES
 * The encoding field of the entry depends on the content of the
 * entry. When the entry is a string, the first 2 bits of the encoding first
 * byte will hold the type of encoding used to store the length of the string,
 * followed by the actual length of the string. When the entry is an integer
 * the first 2 bits are both set to 1. The following 2 bits are used to specify
 * what kind of integer will be stored after this header. An overview of the
 * different types and encodings is as follows. The first byte is always enough
 * to determine the kind of entry.
 *
 * |00pppppp| - 1 byte
 *      String value with length less than or equal to 63 bytes (6 bits).
 *      "pppppp" represents the unsigned 6 bit length.
 * |01pppppp|qqqqqqqq| - 2 bytes
 *      String value with length less than or equal to 16383 bytes (14 bits).
 *      IMPORTANT: The 14 bit number is stored in big endian.
 * |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
 *      String value with length greater than or equal to 16384 bytes.
 *      Only the 4 bytes following the first byte represents the length
 *      up to 2^32-1. The 6 lower bits of the first byte are not used and
 *      are set to zero.
 *      IMPORTANT: The 32 bit number is stored in big endian.
 * |11000000| - 3 bytes
 *      Integer encoded as int16_t (2 bytes).
 * |11010000| - 5 bytes
 *      Integer encoded as int32_t (4 bytes).
 * |11100000| - 9 bytes
 *      Integer encoded as int64_t (8 bytes).
 * |11110000| - 4 bytes
 *      Integer encoded as 24 bit signed (3 bytes).
 * |11111110| - 2 bytes
 *      Integer encoded as 8 bit signed (1 byte).
 * |1111xxxx| - (with xxxx between 0001 and 1101) immediate 4 bit integer.
 *      Unsigned integer from 0 to 12. The encoded value is actually from
 *      1 to 13 because 0000 and 1111 can not be used, so 1 should be
 *      subtracted from the encoded 4 bit value to obtain the right value.
 * |11111111| - End of ziplist special entry.
 *
 * Like for the ziplist header, all the integers are represented in little
 * endian byte order, even when this code is compiled in big endian systems.
 *
 * EXAMPLES OF ACTUAL ZIPLISTS
 * ===========================
 *
 * The following is a ziplist containing the two elements representing
 * the strings "2" and "5". It is composed of 15 bytes, that we visually
 * split into sections:
 *
 *  [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
 *        |             |          |       |       |     |
 *     zlbytes        zltail     zllen    "2"     "5"   end
 *
 * The first 4 bytes represent the number 15, that is the number of bytes
 * the whole ziplist is composed of. The second 4 bytes are the offset
 * at which the last ziplist entry is found, that is 12, in fact the
 * last entry, that is "5", is at offset 12 inside the ziplist.
 * The next 16 bit integer represents the number of elements inside the
 * ziplist, its value is 2 since there are just two elements inside.
 * Finally "00 f3" is the first entry representing the number 2. It is
 * composed of the previous entry length, which is zero because this is
 * our first entry, and the byte F3 which corresponds to the encoding
 * |1111xxxx| with xxxx between 0001 and 1101. We need to remove the "F"
 * higher order bits 1111, and subtract 1 from the "3", so the entry value
 * is "2". The next entry has a prevlen of 02, since the first entry is
 * composed of exactly two bytes. The entry itself, F6, is encoded exactly
 * like the first entry, and 6-1 = 5, so the value of the entry is 5.
 * Finally the special entry FF signals the end of the ziplist.
 *
 * Adding another element to the above string with the value "Hello World"
 * allows us to show how the ziplist encodes small strings. We'll just show
 * the hex dump of the entry itself. Imagine the bytes as following the
 * entry that stores "5" in the ziplist above:
 *
 * [02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]
 *
 * The first byte, 02, is the length of the previous entry. The next
 * byte represents the encoding in the pattern |00pppppp| that means
 * that the entry is a string of length <pppppp>, so 0B means that
 * an 11 bytes string follows. From the third byte (48) to the last (64)
 * there are just the ASCII characters for "Hello World".
 *
 */
```

# 5 配置

## 5.1 Redis 6.0 版本配置

```properties
# Hashes are encoded using a memory efficient data structure when they have a
# small number of entries, and the biggest entry does not exceed a given
# threshold. These thresholds can be configured using the following directives.
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

# Lists are also encoded in a special way to save a lot of space.
# The number of entries allowed per internal list node can be specified
# as a fixed maximum size or a maximum number of elements.
# For a fixed maximum size, use -5 through -1, meaning:
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
# Positive numbers mean store up to _exactly_ that number of elements
# per list node.
# The highest performing option is usually -2 (8 Kb size) or -1 (4 Kb size),
# but if your use case is unique, adjust the settings as necessary.
list-max-ziplist-size -2

# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```

# 6 Reference
* [Redis Ziplist - Redis](https://redis.io/glossary/redis-ziplist/)
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)