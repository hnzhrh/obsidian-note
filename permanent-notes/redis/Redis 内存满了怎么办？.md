---
title: Redis 内存满了怎么办？
tags:
  - permanent-note
  - middleware/redis/eviction
date: 2025-03-06
time: 14:16
aliases:
---
# 1 合理选择需要缓存的数据

缓存的硬件资源是比较昂贵的，可以根据二八定律，选择需要真正缓存的数据，这是一个 trade-off 的过程，经验值一般设置缓存容量为总容量的 15% 到 30% 之间，兼顾访问习性能和内存开销。

# 2 配置最大内存

使用 Redis 时，强烈建议配置 Redis 实例的最大内存，如果没有配置，当实例使用的内存超过了 Server 的物理内存时，会发生 swap，swap 相当于与磁盘交互，会严重拖慢 Redis 性能。建议配置 Redis 实例最大内存为物理机的 70%左右（极限情况下预留一半最为安全），配置项为：

```c
maxmemory <bytes>
```

# 3 内存淘汰

当内存占用满了之后，Redis 会根据配置进行数据淘汰，默认为不淘汰数据，写请求会报错，读请求正常处理，配置为：

```c
maxmemory-policy noeviction
```


![Redis 内存淘汰策略.png](https://images.hnzhrh.com/note/Redis%20%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5.png)

## 3.1 内存淘汰的时机

如果设置了最大内存，且配置了非 `noeviction` 的内粗淘汰策略，执行命令时都会进行判断是否需要释放内存淘汰数据，因此内存数据淘汰是会造成延迟的（未开启异步删除时）。
![image.png](https://images.hnzhrh.com/note/20241205211234.png)
Redis 为了保证淘汰数据不会阻塞太久，导致延迟，可以通过配置 `maxmemory-eviction-tenacity` 来控制内存淘汰带来的延迟消耗。

```c
# Eviction processing is designed to function well with the default setting.
# If there is an unusually large amount of write traffic, this value may need to
# be increased.  Decreasing this value may reduce latency at the risk of
# eviction processing effectiveness
#   0 = minimum latency, 10 = default, 100 = process without regard to latency
#
# maxmemory-eviction-tenacity 10
```

如果耗时超过了配置的限制时间，会添加一个时间循环事件进行内存淘汰。

内存淘汰直到下降到 `maxmemory` 以下。

## 3.2 Redis 内存淘汰 LRU 算法

Redis LRU 并没有实现标准的 LRU 算法，采用近似随机 LRU 算法，主要原因在于：标准 LRU 需要维护全局链表，需要额外的内存且有额外的性能开销。

Redis 在 `redisObject` 对象中，维护了一个 24 位的 LRU 时间，对象添加到 Redis DB 时进行初始化，当对象被访问时，更新 LRU 时间。
![image.png](https://images.hnzhrh.com/note/20241205220558.png)

Redis 创建一个大小为 `EVPOOL_SIZE` (16) 的池子，LRU 算法下该池子按照时间从小到大排序。淘汰时每次选取 N（配置 `maxmemory-samples` 默认为 5）个 Key 放入池子，淘汰 LRU 时间最小的那个 Key。

1. **采样阶段**
    - 当需要淘汰键时，从数据库中**随机选取 N 个键**（N 通常为 5）。
    - 这些键的访问时间戳（`lru` 字段）将与**候选池**中的键比较。
2. **候选池维护**
    - **池结构**：固定大小（由 `EVPOOL_SIZE` 定义，默认 16），按访问时间排序（最久未访问在前）。    
    - **更新策略**：若采样键的 `lru` 比池中某键更旧，则替换较新的键，确保池中始终保留**全局较旧**的候选。    
3. **淘汰决策**
    - 从候选池中选择**最旧**的键（即池首）进行淘汰。    
    - **容错机制**：若该键已不存在（如被其他操作删除），则遍历池直至找到有效键。若池全失效，重新采样填充。

LRU 算法有个问题，就是只要访问一次，就会更新时间戳，在缓存中停留很长一段时间，造成污染，为了解决这个问题，Redis 4.0 引入了 LFU 算法。

## 3.3 Redis 内存淘汰 LFU 算法

LFU 算法在 LRU 的基础上，多增加了一个计数器，用来表示数据的访问频率，使用 LFU 算法淘汰数据时，先筛选访问次数少的，再比较这两个数据的时间，把距离上一次访问时间更久的数据淘汰出缓存。

LFU 并没有使用新的字段来表示，而是复用了 lru 字段，把 24 位拆分了，高 16 位表示时间戳，低 8 位表示访问次数，访问次数初始化的时候会被置为 `LFU_INIT_VAL = 5`，这样做的目的是为了避免新 Key 创建不久后立马被淘汰的情况发生。

因为只有 8 位，最大计数则为 255，所以每访问一次就增加一次计数显然是不可行的。LFU 策略实现的计数规则是：每当数据被访问一次时，首先，用计数器当前的值乘以配置项 `lfu-log-factor` 再加 1，再取其倒数，得到一个 p 值；然后，把这个 p 值和一个取值范围在（0，1）间的随机数 r 值比大小，只有 p 值大于 r 值时，计数器才加 1。

```shell
+--------+------------+------------+------------+------------+------------+
| factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
+--------+------------+------------+------------+------------+------------+
| 0      | 104        | 255        | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 1      | 18         | 49         | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 10     | 10         | 18         | 142        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 100    | 8          | 11         | 49         | 143        | 255        |
+--------+------------+------------+------------+------------+------------+
```

源码：

```c
/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really incremented. Saturate it at 255. */
#define LFU_INIT_VAL 5
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}
```

如果一个数据短时间被大量访问之后，导致计数很大，则在短时间内不会被淘汰掉了，因此 LFU 还会根绝配置项 `lfu-decay-time` (默认为 1) 做衰减，LFU 策略会计算当前时间和数据最近一次访问时间的差值，并把这个差值换算成以分钟为单位。然后，LFU 策略再把这个差值除以 `lfu-decay-time` 值，所得的结果就是数据 counter 要衰减的值。

源码：

```c
/* If the object's ldt (last access time) is reached, decrement the LFU counter but
 * do not update LFU fields of the object, we update the access time
 * and counter in an explicit way when the object is really accessed.
 * And we will decrement the counter according to the times of
 * elapsed time than server.lfu_decay_time.
 * Return the object frequency counter.
 *
 * This function is used in order to scan the dataset for the best object
 * to fit: as we check for the candidate, we incrementally decrement the
 * counter of the scanned objects if needed. */
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    return counter;
}
```

# 4 Reference

* [Key eviction \| Docs](https://redis.io/docs/latest/develop/reference/eviction/)
* [LRU算法及其优化策略——算法篇LRU算法全称是最近最少使用算法（Least Recently Use），广泛的应用于缓 - 掘金](https://juejin.cn/post/6844904049263771662)
* [Redis源码剖析之内存淘汰策略(Evict)-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2442586)
* [15 \| 为什么LRU算法原理和代码实现不一样？-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/412164)
* [How we diagnosed and resolved Redis latency spikes with BPF and other tools](https://about.gitlab.com/blog/2022/11/28/how-we-diagnosed-and-resolved-redis-latency-spikes/)
* [redis的内存管理机制——过期键删除与内存淘汰本文内存管理的内容包括： 过期键的懒性删除和过期删除 内存溢出控制策略。 - 掘金](https://juejin.cn/post/7026901224171520013)
* [20 \| 删除数据后，为什么内存占用率还是很高？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/289140)
* [Redis 数据过期及数据淘汰机制深度剖析](https://time.geekbang.org/qconplus/detail/100110468)
* 内存淘汰核心代码：
	* [redis/src/evict.c at f364dcca2d8288dae95fffd3eba6f8b7ac6c01ee · redis/redis · GitHub](https://github.com/redis/redis/blob/f364dcca2d8288dae95fffd3eba6f8b7ac6c01ee/src/evict.c#L520)
	* [redis/src/evict.c at f364dcca2d8288dae95fffd3eba6f8b7ac6c01ee · redis/redis · GitHub](https://github.com/redis/redis/blob/f364dcca2d8288dae95fffd3eba6f8b7ac6c01ee/src/evict.c#L103)