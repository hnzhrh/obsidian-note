---
title: Redis内存满了怎么办？
tags:
  - permanent-note
  - middleware/redis/eviction
date: 2024-12-05
time: 16:45
aliases:
---
使用 Redis 时，强烈建议配置 Redis 实例的最大内存，如果没有配置，当实例使用的内存超过了 Server 的物理内存时，会发生 swap，swap 相当于与磁盘交互，会严重拖慢 Redis 性能。建议配置 Redis 实例最大内存为物理机的 70%左右（极限情况下预留一半最为安全），配置项为：
```c
maxmemory <bytes>
```

当内存占用满了之后，Redis 会根据配置进行数据淘汰，默认为不淘汰数据，写请求会报错，读请求正常处理，配置为：
```c
maxmemory-policy noeviction
```

Redis 提供了以下几种淘汰策略：

```plantuml

@startmindmap
!theme cerulean

* Max memory policy
	* Volatile
		* LRU
		* LFU
		* Random
		* TTL
	* All keys
		* LRU
		* LFU
		* Random
	* Noeviction

@endmindmap
```
默认为 `noeviction`，`volatile` 表示在过期键中进行淘汰，`allkeys` 表示在所有键空间中淘汰数据，在 Redis 中，`LRU` `LFU` `TTL` 算法的实现都采用了近似随机算法（approximated randomized algorithms）。 

## 什么时候淘汰数据？

如何设置了最大内存，且配置了非 `noeviction` 的内粗淘汰策略，执行命令时都会进行判断是否需要释放内存淘汰数据，因此内存数据淘汰是会造成延迟的（未开启异步删除时）。
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

## Redis LRU 算法

Redis LRU 并没有实现标准的 LRU 算法，采用近似随机 LRU 算法，主要原因在于：标准 LRU 需要维护全局链表，需要额外的内存且有额外的性能开销。

Redis 在 `redisObject` 对象中，维护了一个 24 位的 LRU 时间，对象添加到 Redis DB 时进行初始化，当对象被访问时，更新 LRU 时间。
![image.png](https://images.hnzhrh.com/note/20241205220558.png)


Redis 创建一个大小为 `EVPOOL_SIZE` (16) 的池子，LRU 算法下该池子按照时间从小到大排序。淘汰时每次选取 N（配置 `maxmemory-samples`）个 Key 放入池子，淘汰 LRU 时间最小的那个 Key。

```c
/* LRU approximation algorithm
 *
 * Redis uses an approximation of the LRU algorithm that runs in constant
 * memory. Every time there is a key to expire, we sample N keys (with
 * N very small, usually in around 5) to populate a pool of best keys to
 * evict of M keys (the pool size is defined by EVPOOL_SIZE).
 *
 * The N keys sampled are added in the pool of good keys to expire (the one
 * with an old access time) if they are better than one of the current keys
 * in the pool.
 *
 * After the pool is populated, the best key we have in the pool is expired.
 * However note that we don't remove keys from the pool when they are deleted
 * so the pool may contain keys that no longer exist.
 *
 * When we try to evict a key, and all the entries in the pool don't exist
 * we populate it again. This time we'll be sure that the pool has at least
 * one key that can be evicted, if there is at least one key that can be
 * evicted in the whole database. */

/* Create a new eviction pool. */
void evictionPoolAlloc(void) {
    struct evictionPoolEntry *ep;
    int j;

    ep = zmalloc(sizeof(*ep)*EVPOOL_SIZE);
    for (j = 0; j < EVPOOL_SIZE; j++) {
        ep[j].idle = 0;
        ep[j].key = NULL;
        ep[j].cached = sdsnewlen(NULL,EVPOOL_CACHED_SDS_SIZE);
        ep[j].dbid = 0;
    }
    EvictionPoolLRU = ep;
}
```

具体代码如下：
```c
/* Check that memory usage is within the current "maxmemory" limit.  If over
 * "maxmemory", attempt to free memory by evicting data (if it's safe to do so).
 *
 * It's possible for Redis to suddenly be significantly over the "maxmemory"
 * setting.  This can happen if there is a large allocation (like a hash table
 * resize) or even if the "maxmemory" setting is manually adjusted.  Because of
 * this, it's important to evict for a managed period of time - otherwise Redis
 * would become unresponsive while evicting.
 *
 * The goal of this function is to improve the memory situation - not to
 * immediately resolve it.  In the case that some items have been evicted but
 * the "maxmemory" limit has not been achieved, an aeTimeProc will be started
 * which will continue to evict items until memory limits are achieved or
 * nothing more is evictable.
 *
 * This should be called before execution of commands.  If EVICT_FAIL is
 * returned, commands which will result in increased memory usage should be
 * rejected.
 *
 * Returns:
 *   EVICT_OK       - memory is OK or it's not possible to perform evictions now
 *   EVICT_RUNNING  - memory is over the limit, but eviction is still processing
 *   EVICT_FAIL     - memory is over the limit, and there's nothing to evict
 * */
int performEvictions(void) {
    /* Note, we don't goto update_metrics here because this check skips eviction
     * as if it wasn't triggered. it's a fake EVICT_OK. */
    if (!isSafeToPerformEvictions()) return EVICT_OK;

    int keys_freed = 0;
    size_t mem_reported, mem_tofree;
    long long mem_freed; /* May be negative */
    mstime_t latency;
    int slaves = listLength(server.slaves);
    int result = EVICT_FAIL;

    if (getMaxmemoryState(&mem_reported,NULL,&mem_tofree,NULL) == C_OK) {
        result = EVICT_OK;
        goto update_metrics;
    }

    if (server.maxmemory_policy == MAXMEMORY_NO_EVICTION) {
        result = EVICT_FAIL;  /* We need to free memory, but policy forbids. */
        goto update_metrics;
    }

    unsigned long eviction_time_limit_us = evictionTimeLimitUs();

    mem_freed = 0;

    latencyStartMonitor(latency);

    monotime evictionTimer;
    elapsedStart(&evictionTimer);

    /* Try to smoke-out bugs (server.also_propagate should be empty here) */
    serverAssert(server.also_propagate.numops == 0);
    /* Evictions are performed on random keys that have nothing to do with the current command slot. */

    while (mem_freed < (long long)mem_tofree) {
        int j, k, i;
        static unsigned int next_db = 0;
        sds bestkey = NULL;
        int bestdbid;
        redisDb *db;
        dictEntry *de;

        if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
            server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
        {
            struct evictionPoolEntry *pool = EvictionPoolLRU;
            while (bestkey == NULL) {
                unsigned long total_keys = 0;

                /* We don't want to make local-db choices when expiring keys,
                 * so to start populate the eviction pool sampling keys from
                 * every DB. */
                for (i = 0; i < server.dbnum; i++) {
                    db = server.db+i;
                    kvstore *kvs;
                    if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
                        kvs = db->keys;
                    } else {
                        kvs = db->expires;
                    }
                    unsigned long sampled_keys = 0;
                    unsigned long current_db_keys = kvstoreSize(kvs);
                    if (current_db_keys == 0) continue;

                    total_keys += current_db_keys;
                    int l = kvstoreNumNonEmptyDicts(kvs);
                    /* Do not exceed the number of non-empty slots when looping. */
                    while (l--) {
                        sampled_keys += evictionPoolPopulate(db, kvs, pool);
                        /* We have sampled enough keys in the current db, exit the loop. */
                        if (sampled_keys >= (unsigned long) server.maxmemory_samples)
                            break;
                        /* If there are not a lot of keys in the current db, dict/s may be very
                         * sparsely populated, exit the loop without meeting the sampling
                         * requirement. */
                        if (current_db_keys < (unsigned long) server.maxmemory_samples*10)
                            break;
                    }
                }
                if (!total_keys) break; /* No keys to evict. */

                /* Go backward from best to worst element to evict. */
                for (k = EVPOOL_SIZE-1; k >= 0; k--) {
                    if (pool[k].key == NULL) continue;
                    bestdbid = pool[k].dbid;

                    kvstore *kvs;
                    if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
                        kvs = server.db[bestdbid].keys;
                    } else {
                        kvs = server.db[bestdbid].expires;
                    }
                    de = kvstoreDictFind(kvs, pool[k].slot, pool[k].key);

                    /* Remove the entry from the pool. */
                    if (pool[k].key != pool[k].cached)
                        sdsfree(pool[k].key);
                    pool[k].key = NULL;
                    pool[k].idle = 0;

                    /* If the key exists, is our pick. Otherwise it is
                     * a ghost and we need to try the next element. */
                    if (de) {
                        bestkey = dictGetKey(de);
                        break;
                    } else {
                        /* Ghost... Iterate again. */
                    }
                }
            }
        }

        /* volatile-random and allkeys-random policy */
        else if (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM ||
                 server.maxmemory_policy == MAXMEMORY_VOLATILE_RANDOM)
        {
            /* When evicting a random key, we try to evict a key for
             * each DB, so we use the static 'next_db' variable to
             * incrementally visit all DBs. */
            for (i = 0; i < server.dbnum; i++) {
                j = (++next_db) % server.dbnum;
                db = server.db+j;
                kvstore *kvs;
                if (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM) {
                    kvs = db->keys;
                } else {
                    kvs = db->expires;
                }
                int slot = kvstoreGetFairRandomDictIndex(kvs);
                de = kvstoreDictGetRandomKey(kvs, slot);
                if (de) {
                    bestkey = dictGetKey(de);
                    bestdbid = j;
                    break;
                }
            }
        }

        /* Finally remove the selected key. */
        if (bestkey) {
            long long key_mem_freed;
            db = server.db+bestdbid;

            enterExecutionUnit(1, 0);
            robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
            deleteEvictedKeyAndPropagate(db, keyobj, &key_mem_freed);
            decrRefCount(keyobj);
            exitExecutionUnit();
            /* Propagate the DEL command */
            postExecutionUnitOperations();

            mem_freed += key_mem_freed;
            keys_freed++;

            if (keys_freed % 16 == 0) {
                /* When the memory to free starts to be big enough, we may
                 * start spending so much time here that is impossible to
                 * deliver data to the replicas fast enough, so we force the
                 * transmission here inside the loop. */
                if (slaves) flushSlavesOutputBuffers();

                /* Normally our stop condition is the ability to release
                 * a fixed, pre-computed amount of memory. However when we
                 * are deleting objects in another thread, it's better to
                 * check, from time to time, if we already reached our target
                 * memory, since the "mem_freed" amount is computed only
                 * across the dbAsyncDelete() call, while the thread can
                 * release the memory all the time. */
                if (server.lazyfree_lazy_eviction) {
                    if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                        break;
                    }
                }

                /* After some time, exit the loop early - even if memory limit
                 * hasn't been reached.  If we suddenly need to free a lot of
                 * memory, don't want to spend too much time here.  */
                if (elapsedUs(evictionTimer) > eviction_time_limit_us) {
                    // We still need to free memory - start eviction timer proc
                    startEvictionTimeProc();
                    break;
                }
            }
        } else {
            goto cant_free; /* nothing to free... */
        }
    }
    /* at this point, the memory is OK, or we have reached the time limit */
    result = (isEvictionProcRunning) ? EVICT_RUNNING : EVICT_OK;

cant_free:
    if (result == EVICT_FAIL) {
        /* At this point, we have run out of evictable items.  It's possible
         * that some items are being freed in the lazyfree thread.  Perform a
         * short wait here if such jobs exist, but don't wait long.  */
        mstime_t lazyfree_latency;
        latencyStartMonitor(lazyfree_latency);
        while (bioPendingJobsOfType(BIO_LAZY_FREE) &&
              elapsedUs(evictionTimer) < eviction_time_limit_us) {
            if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                result = EVICT_OK;
                break;
            }
            usleep(eviction_time_limit_us < 1000 ? eviction_time_limit_us : 1000);
        }
        latencyEndMonitor(lazyfree_latency);
        latencyAddSampleIfNeeded("eviction-lazyfree",lazyfree_latency);
    }

    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);

update_metrics:
    if (result == EVICT_RUNNING || result == EVICT_FAIL) {
        if (server.stat_last_eviction_exceeded_time == 0)
            elapsedStart(&server.stat_last_eviction_exceeded_time);
    } else if (result == EVICT_OK) {
        if (server.stat_last_eviction_exceeded_time != 0) {
            server.stat_total_eviction_exceeded_time += elapsedUs(server.stat_last_eviction_exceeded_time);
            server.stat_last_eviction_exceeded_time = 0;
        }
    }
    return result;
}
```

LRU 算法有个问题，就是只要访问一次，就会更新时间戳，在缓存中停留很长一段时间，造成污染，为了解决这个问题，Redis 4.0 引入了 LFU 算法。

## Redis LFU 算法

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

如果一个数据短时间被大量访问之后，导致计数很大，则在短时间内不会被淘汰掉了，因此 LFU 还会根绝配置项 `lfu-decay-time` (默认为 1)做衰减，LFU 策略会计算当前时间和数据最近一次访问时间的差值，并把这个差值换算成以分钟为单位。然后，LFU 策略再把这个差值除以 `lfu-decay-time` 值，所得的结果就是数据 counter 要衰减的值。

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

## 内存碎片

因为内存分配机制必然存在内存碎片的问题，比如 `jemalloc` 的分配策略之一就是按照一系列固定的大小划分内存空间的。可以通过 `info` 命令查看内存的详细信息，查看指标 `mem_fragmentation_ratio`, 经验阈值：
* mem_fragmentation_ratio 大于 1 但小于 1.5。这种情况是合理的。这是因为，刚才我介绍的那些因素是难以避免的。毕竟，内因的内存分配器是一定要使用的，分配策略都是通用的，不会轻易修改；而外因由 Redis 负载决定，也无法限制。所以，存在内存碎片也是正常的。
* mem_fragmentation_ratio 大于 1.5 。这表明内存碎片率已经超过了 50%。一般情况下，这个时候，我们就需要采取一些措施来降低内存碎片率了。
```c
mem_fragmentation_ratio = used_memory_rss/ used_memory
```

当 Redis 内存实例有大量内存碎片后，该如何清理呢？
* 暴力重启 Redis 实例，可能会导致数据持久化过程中数据的丢失，恢复实例也是需要一定时间的
* Redis 4.0 以后，Redis 提供内存碎片清理的机制
默认是关闭的，可以通过命令开启：
```shell
config set activedefrag yes
```
需要注意的是，内存碎片清理需要拷贝原有的数据到新的内存地址，因此会有延迟开销，需要根据实际情况进行配置，防止实例性能降低。

相关配置如下：
![Redis配置和部署](Redis配置和部署.md#内存碎片管理)
# Reference
* [Key eviction \| Docs](https://redis.io/docs/latest/develop/reference/eviction/)
* [LRU算法及其优化策略——算法篇LRU算法全称是最近最少使用算法（Least Recently Use），广泛的应用于缓 - 掘金](https://juejin.cn/post/6844904049263771662)
* [Redis源码剖析之内存淘汰策略(Evict)-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2442586)
* [15 \| 为什么LRU算法原理和代码实现不一样？-Redis源码剖析与实战-极客时间](https://time.geekbang.org/column/article/412164)
* [How we diagnosed and resolved Redis latency spikes with BPF and other tools](https://about.gitlab.com/blog/2022/11/28/how-we-diagnosed-and-resolved-redis-latency-spikes/)
* [redis的内存管理机制——过期键删除与内存淘汰本文内存管理的内容包括： 过期键的懒性删除和过期删除 内存溢出控制策略。 - 掘金](https://juejin.cn/post/7026901224171520013)
* [20 \| 删除数据后，为什么内存占用率还是很高？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/289140)