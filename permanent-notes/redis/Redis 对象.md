---
title: Redis 对象
tags:
  - permanent-note
  - middleware/redis/data-object
date: 2025-03-04
time: 08:41
aliases:
---
# Redis 对象的结构

Redis 对象结构如下：

```c
#define LRU_BITS 24
struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
};
```

因此 Value 的元数据占用 `16 Bytes`：
* 4 bits type
* 4 bits encoding
* 24 bits lru
* 4 bytes refcount
* 8 bytes 指针

# 字段解释

## Type 字段

Type 为对象类型，encoding 为 Redis 为了存储空间、性能做了权衡的优化。

```c
/*-----------------------------------------------------------------------------
 * Data types
 *----------------------------------------------------------------------------*/

/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

/* The "module" object type is a special one that signals that the object
 * is one directly managed by a Redis module. In this case the value points
 * to a moduleValue struct, which contains the object value (which is only
 * handled by the module itself) and the RedisModuleType struct which lists
 * function pointers in order to serialize, deserialize, AOF-rewrite and
 * free the object.
 *
 * Inside the RDB file, module types are encoded as OBJ_MODULE followed
 * by a 64 bit module type ID, which has a 54 bits module-specific signature
 * in order to dispatch the loading to the right module, plus a 10 bits
 * encoding version. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
#define OBJ_TYPE_MAX 7  /* Maximum number of object types */
```

## Encoding 字段

Redis 为了性能考虑，一种数据对象可以使用多种编码方式进行实现，该字段指定了对象的实现方式。

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
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

## Refcount 字段

其中的 `refcount` 字段用于引用计数（Reference Counting），它是一个重要的内存管理和对象生命周期控制机制。

`refcount` 字段的主要用途包括：

1. **共享对象**：Redis 为了节省内存，会尽可能地共享一些相同的对象。例如，对于小整数，Redis 会创建一次这些整数的对象，并让所有需要这些数值的地方共享同一个对象。`refcount` 用来跟踪有多少地方正在使用这个对象。

2. **惰性删除和复制**：当一个对象被多个键引用时，如果其中一个键对应的值要被修改，Redis 不会立即复制该对象，而是等到真正需要修改时才进行复制（即写时复制 Copy-On-Write）。`refcount` 帮助判断对象是否被共享，以及何时需要复制对象以确保每个键都有自己的副本。

3. **垃圾回收**：当 `refcount` 减少到 0 时，说明没有其他地方再引用这个对象了，此时可以安全地释放该对象所占用的内存。这是 Redis 内部实现自动垃圾回收机制的一部分。

Refcount 有几个特殊值：
* `OBJ_SHARED_REFCOUNT (INT_MAX)`
	- **作用**：表示全局共享对象永远不会被销毁。
	- **场景**：Redis 创建了一些静态的、常用的小整数值对象（如 0 到 9999 的整数），这些对象可以在多个地方被共享使用。因为这些对象是全局共享的，并且预期在整个服务器生命周期内都存在，所以它们的引用计数被设置为 `INT_MAX`，即最大可能的整数值。
	- **效果**：由于 `refcount` 不会减少到 0，这些对象永远不会被认为是可释放的，从而确保了它们在整个 Redis 实例运行期间一直可用。
*  `OBJ_STATIC_REFCOUNT (INT_MAX - 1)`
	- **作用**：表示栈上分配的对象。
	- **场景**：某些对象可能是临时性的或是在栈上分配的，这些对象不需要遵循常规的引用计数规则来管理其生命周期，因为它们的生命周期由其他机制控制（例如函数调用栈）。
	- **效果**：通过将 `refcount` 设置为 `INT_MAX - 1`，Redis 可以标记这些对象为特殊状态，避免对它们进行常规的内存管理和垃圾回收逻辑。但是，与 `OBJ_SHARED_REFCOUNT` 不同的是，这些对象并不是设计成永远存在的；它们只是暂时地存在于栈中，通常在其作用域结束时自动消失。
* `OBJ_FIRST_SPECIAL_REFCOUNT`
	- **作用**：定义了特殊 `refcount` 值的起始点。
	- **说明**：这是一个辅助宏，用来标识从哪个值开始，`refcount` 应该被视为具有特殊含义，而不是普通的引用计数值。这有助于 Redis 在检查 `refcount` 时快速判断对象是否属于上述特殊情况之一。

## Ptr 指针

指向具体数据的地址。

# Reference