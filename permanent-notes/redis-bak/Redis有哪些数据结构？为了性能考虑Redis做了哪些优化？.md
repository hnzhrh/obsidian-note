---
title: Redis有哪些数据结构？为了性能考虑Redis做了哪些优化？
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2024-12-05
time: 15:52
aliases:
---
![](https://img.shields.io/badge/version-7.0-blue)

Redis 有 7 种数据对象：
* String object
* List object
* Set object
* Sorted set object
* Hash object
* Module object
* Stream object

某些 Redis 数据对象（比如 String、Hash 等）根据数据长度和数据量而采用不同的数据结构，称为数据对象编码方式，Redis 提供了非常多的编码方式：

```c
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
#define OBJ_ENCODING_LISTPACK_EX 12 /* Encoded as listpack, extended with metadata */
```

数据对象对应的编码方式如下：
```plantuml

@startmindmap
!theme cerulean

* String
	* INT
	* EMBSTR
	* RAW
* List
	* LISTPACK
	* QUICKLIST
	* ~~ZIPLIST~~
	* ~~LINKEDLIST~~
* Set
	* INTSET
	* LISTPACK
	* HT
* Zset
	* LISTPACK
	* SKIPLIST
* Hash
	* HT
	* LISTPACK
	* ~~ZIPLIST~~

@endmindmap
```



# Reference