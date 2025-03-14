---
title: Redis 有哪些数据类型？各自的实现是怎样的？
tags:
  - permanent-note
  - middleware/redis/data-object
  - middleware/redis/data-types
date: 2025-03-04
time: 08:34
aliases:
---
![](https://img.shields.io/badge/version-7.0-blue)

# 1 Redis 数据类型

Redis 提供了丰富的数据类型帮助用户解决需要缓存、队列和事件处理的各种问题，Redis 社区版提供了以下数据类型：
- [String](https://redis.io/docs/latest/develop/data-types/#strings)
- [Hash](https://redis.io/docs/latest/develop/data-types/#hashes)
- [List](https://redis.io/docs/latest/develop/data-types/#lists)
- [Set](https://redis.io/docs/latest/develop/data-types/#sets)
- [Sorted set](https://redis.io/docs/latest/develop/data-types/#sorted-sets)
- [Stream](https://redis.io/docs/latest/develop/data-types/#streams)
- [Bitmap](https://redis.io/docs/latest/develop/data-types/#bitmaps)
- [Bitfield](https://redis.io/docs/latest/develop/data-types/#bitfields)
- [Geospatial](https://redis.io/docs/latest/develop/data-types/#geospatial-indexes)
# 2 Redis 数据对象和其编码实现的关系
![Redis 数据对象和编码.png](https://images.hnzhrh.com/note/Redis%20%E6%95%B0%E6%8D%AE%E5%AF%B9%E8%B1%A1%E5%92%8C%E7%BC%96%E7%A0%81.png)
# 3 Redis 数据对象

Redis Database 整体而言是一个巨大的哈希表，都是以 key-value 键值对的形式存储的，其中 key 以 String 对象存储，Value 以 [Redis 对象](Redis%20对象.md) 存储，Redis 有 7 种数据对象：

## 3.1 [Redis String Object](Redis%20数据对象之%20String.md)

## 3.2 [Redis 数据对象之 String](Redis%20数据对象之%20String.md)

## 3.3 [[Redis 数据对象之 Set]]

## 3.4 [[Redis 数据对象之 Zset]]

## 3.5 [[Redis 数据对象之 Hash]]

## 3.6 Redis Module Object

## 3.7 Redis Stream Object

# 4 Redis 数据对象编码

某些 Redis 数据对象（比如 String、Hash 等）根据数据长度和数据量而采用不同的数据结构，称为数据对象编码方式，Redis 数据对象编码有以下几种：

## 4.1 [[Redis Object Encoding RAW]]  
  
## 4.2 [[Redis Object Encoding INT]]  
  
## 4.3 [[Redis Object Encoding HT]]  
  
## 4.4 Redis Object Encoding ZIPMAP  
  
## 4.5 Redis Object Encoding LINKEDLIST  
  
## 4.6 [[Redis Object Encoding ZIPLIST]]  
  
## 4.7 [[Redis Object Encoding INTSET ]] 
  
## 4.8 [[Redis Object Encoding SKIPLIST]]  
  
## 4.9 [[Redis Object Encoding EMBSTR ]] 
  
## 4.10 [[Redis Object Encoding QUICKLIST]]  
  
## 4.11 Redis Object Encoding STREAM  
  
## 4.12 [[Redis Object Encoding LISTPACK]]  
  
## 4.13 Redis Object Encoding LISTPACK_EX

# 5 References
* [Understand Redis data types \| Docs](https://redis.io/docs/latest/develop/data-types/)

