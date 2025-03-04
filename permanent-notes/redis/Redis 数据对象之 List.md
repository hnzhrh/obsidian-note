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

`ZIPLIST` 设计成一种内存高效的数据结构，可以在两端 `O(1)` 操作，但其他的操作需要对 `ZIPLIST` 重新分配内存，所以整体复杂度取决于 `ZIPLIST` 的大小。 ^9786 bc

内存结构如下：
![image.png](https://images.hnzhrh.com/note/20241214153120.png)

* `zlbytes`，表示 `ZIPLIST` 占用的总字节数
* `zltail`，表示 `ZIPLIST` 最后一个 `entry` 的偏移量，通过 `zltail` 可以 `O(1)` `POP` 操作，不需要遍历整个 `ZIPLIST`
* `zllen`，表示 `ZIPLIST` 中 `entry` 的个数，当 `ZIPLIST` 中 `entry` 个数超过了 2<sup>16</sup> -2 个，则会被设置为 2<sup>16</sup>-1，这是一个特殊标记符，表示 `ZIPLIST` 中的元素个数需要通过遍历才能获得
* `zlend`，特殊的记号表示为 `ZIPLIST` 的结尾，值为 `255`（`OxFF`）


`ZIPLIST` `entry` 有相当多的编码方式，大致分为两类：
* `<prevlen> <encoding> <entry-data>`
* `<prevlen> <encoding>` , 对于小整数，可以不使用 `<entry-data>` 部分

其中 `<prevlen>` 部分可能是 1 Byte 也可能是 5 Bytes，取决于前一个 `entry` 的大小，当前一个 `entry` 大小大于等于 254 时，使用 `0xFE + 4 Bytes len` 来标记。
![image.png](https://images.hnzhrh.com/note/20241214155939.png)

`<prevlen>` 的存在可能会引起连锁更新问题，假设多个 `entry` 大小都在 253，如果第一个变更了内容，`prelen` 从 1 Byte 膨胀到了 5 Bytes，之后的多个连续 `entry` 恰好都需要膨胀，则会出现连锁更新问题。

各种类型的编码具体参考注释：

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


# 2 LISTPACK 编码

Listpack 也叫紧凑列表，它的特点就是用一块连续的内存空间来紧凑地保存数据，同时为了节省内存空间，listpack 列表项使用了多种编码方式，来表示不同长度的数据，这些数据包括整数和字符串。

![image.png](https://images.hnzhrh.com/note/20241214190629.png)



# 3 Reference
* [Redis源码-List：Redis List存储原理、Redis List命令、 Redis List存储底层编码quicklist、Redis List应用场景\_redis的list源码-CSDN博客](https://blog.csdn.net/qq_41929714/article/details/126342953)
* [Redis Ziplist Data Structure. Redis is an in-memory storage server… \| by Shubhi Jain \| InterviewNoodle](https://interviewnoodle.com/redis-ziplist-data-structure-23c8e7e3266d)
* [listpack/listpack.md at master · antirez/listpack · GitHub](https://github.com/antirez/listpack/blob/master/listpack.md)
* [06 \| 从ziplist到quicklist，再到listpack的启发-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/405387)