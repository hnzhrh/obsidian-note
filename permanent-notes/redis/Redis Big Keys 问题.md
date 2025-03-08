---
title: Redis Big Keys 问题
tags:
  - permanent-note
  - middleware/redis/application
date: 2025-03-06
time: 21:07
aliases:
---
# 1 什么是 Big keys？

Big keys 具体表现为 Redis 中某个 Key 的 Value 占用了较大的内存空间，或者某个 Key 的 Value 很多。

针对不同的数据对象类型，常见的实例如下：
* String 类型，值超过了 5 MB
* Set 类型，超过 10000 个成员
* List 类型，超过了 10000 个成员
* Hash 类型，成员超过 1000 个，占用大小超过 100 MB

# 2 Big keys 是如何产生的？

* 存储不合理的数据，比如 String 数据对象存储了很大的二进制数据等
* 没有设置过期时间，且没有定期清理过期数据，导致 List、Set、Hash 等数据对象膨胀成 Big key
* 业务设计不合理，没有合理拆分 Key

# 3 Big keys 的影响

* 客户端执行命令的时长变慢，Redis 是单线程架构，操作 Big key 耗时久
* Big key 容易造成内存很快打满，达到 `maxmemory` 配置，可能造成 SWAP 大幅度降低性能，如果没有开启 SWAP 可能会 OOM
* 集群架构下，容易出现内存分布不均衡，某个数据分片的节点的内存占用可能远大于其他节点
* 对大Key执行读请求，会使实例的带宽使用率被占满，导致自身服务变慢，同时易波及相关的服务。
* 对大Key执行删除操作，易造成主库较长时间的阻塞，进而可能引发同步中断或主从切换。

# 4 如何发现 Big keys？

## 4.1 在从节点使用 `redis-cli` 命令分析 Big keys

```shell
redis-cli --bigkeys
redis-cli --memkeys
redis-cli --keystats
```

底层实际上是使用了各种 `SCAN` 命令，因此会有全量扫描带来的性能问题，推荐在从节点上进行执行。

需要注意的是，`--bigkeys` 和 `--memkeys` 只会找出各个类型最大的 key，所以不够灵活。

### 4.1.1 `redis-cli --bigkeys`

```shell
$ redis-cli --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

100.00% ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
Keys sampled: 55

-------- summary -------

Total key length in bytes is 495 (avg len 9.00)

Biggest   list found "bikes:finished" has 1 items
Biggest string found "all_bikes" has 36 bytes
Biggest   hash found "bike:1:stats" has 3 fields
Biggest stream found "race:france" has 4 entries
Biggest    set found "bikes:racing:france" has 3 members
Biggest   zset found "racer_scores" has 8 members

1 lists with 1 items (01.82% of keys, avg size 1.00)
16 strings with 149 bytes (29.09% of keys, avg size 9.31)
1 MBbloomCFs with 0 ? (01.82% of keys, avg size 0.00)
1 hashs with 3 fields (01.82% of keys, avg size 3.00)
3 streams with 8 entries (05.45% of keys, avg size 2.67)
2 TDIS-TYPEs with 0 ? (03.64% of keys, avg size 0.00)
1 TopK-TYPEs with 0 ? (01.82% of keys, avg size 0.00)
2 sets with 5 members (03.64% of keys, avg size 2.50)
1 CMSk-TYPEs with 0 ? (01.82% of keys, avg size 0.00)
2 zsets with 11 members (03.64% of keys, avg size 5.50)
25 ReJSON-RLs with 0 ? (45.45% of keys, avg size 0.00)
```

### 4.1.2 `redis-cli --memkeys`

```shell
$ redis-cli --memkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

100.00% ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
Keys sampled: 55

-------- summary -------

Total key length in bytes is 495 (avg len 9.00)

Biggest   list found "bikes:finished" has 104 bytes
Biggest string found "all_bikes" has 120 bytes
Biggest MBbloomCF found "bikes:models" has 1048680 bytes
Biggest   hash found "bike:1:stats" has 104 bytes
Biggest stream found "race:italy" has 7172 bytes
Biggest TDIS-TYPE found "bikes:sales" has 9832 bytes
Biggest TopK-TYPE found "bikes:keywords" has 114256 bytes
Biggest    set found "bikes:racing:france" has 120 bytes
Biggest CMSk-TYPE found "bikes:profit" has 144056 bytes
Biggest   zset found "racer_scores" has 168 bytes
Biggest ReJSON-RL found "bikes:inventory" has 4865 bytes

1 lists with 104 bytes (01.82% of keys, avg size 104.00)
16 strings with 1360 bytes (29.09% of keys, avg size 85.00)
1 MBbloomCFs with 1048680 bytes (01.82% of keys, avg size 1048680.00)
1 hashs with 104 bytes (01.82% of keys, avg size 104.00)
3 streams with 16960 bytes (05.45% of keys, avg size 5653.33)
2 TDIS-TYPEs with 19648 bytes (03.64% of keys, avg size 9824.00)
1 TopK-TYPEs with 114256 bytes (01.82% of keys, avg size 114256.00)
2 sets with 208 bytes (03.64% of keys, avg size 104.00)
1 CMSk-TYPEs with 144056 bytes (01.82% of keys, avg size 144056.00)
2 zsets with 304 bytes (03.64% of keys, avg size 152.00)
25 ReJSON-RLs with 15748 bytes (45.45% of keys, avg size 629.92)
```

### 4.1.3 `redis-cli --keystats`

7.4 版本后新加的，是以上两个命令的结合体，可以使用 `--top <n>` 来查找更多的值。

```shell
$ redis-cli --keystats

# Scanning the entire keyspace to find the biggest keys and distribution information.
# Use -i 0.1 to sleep 0.1 sec per 100 SCAN commands (not usually needed).
# Use --cursor <n> to start the scan at the cursor <n> (usually after a Ctrl-C).
# Use --top <n> to display <n> top key sizes (default is 10).
# Ctrl-C to stop the scan.

100.00% ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
Keys sampled: 55
Keys size:    1.30M

--- Top 10 key sizes ---
  1    1.00M MBbloomCF  "bikes:models"
  2  140.68K CMSk-TYPE  "bikes:profit"
  3  111.58K TopK-TYPE  "bikes:keywords"
  4    9.60K TDIS-TYPE  "bikes:sales"
  5    9.59K TDIS-TYPE  "racer_ages"
  6    7.00K stream     "race:italy"
  7    4.92K stream     "race:france"
  8    4.75K ReJSON-RL  "bikes:inventory"
  9    4.64K stream     "race:usa"
 10    1.26K ReJSON-RL  "bicycle:7"

--- Top size per type ---
list       bikes:finished is 104B
string     all_bikes is 120B
MBbloomCF  bikes:models is 1.00M
hash       bike:1:stats is 104B
stream     race:italy is 7.00K
TDIS-TYPE  bikes:sales is 9.60K
TopK-TYPE  bikes:keywords is 111.58K
set        bikes:racing:france is 120B
CMSk-TYPE  bikes:profit is 140.68K
zset       racer_scores is 168B
ReJSON-RL  bikes:inventory is 4.75K

--- Top length and cardinality per type ---
list       bikes:finished has 1 items
string     all_bikes has 36B
hash       bike:1:stats has 3 fields
stream     race:france has 4 entries
set        bikes:racing:france has 3 members
zset       racer_scores has 8 members

Key size Percentile Total keys
-------- ---------- -----------
     64B    0.0000%           3
    239B   50.0000%          28
    763B   75.0000%          42
   4.92K   87.5000%          49
   9.60K   93.7500%          52
 140.69K   96.8750%          54
   1.00M  100.0000%          55
Note: 0.01% size precision, Mean: 24.17K, StdDeviation: 138.12K

Key name length Percentile Total keys
--------------- ---------- -----------
            19B  100.0000%          55
Total key length is 495B (9B avg)

Type        Total keys  Keys % Tot size Avg size  Total length/card Avg ln/card
--------- ------------ ------- -------- -------- ------------------ -----------
list                 1   1.82%     104B     104B            1 items        1.00
string              16  29.09%    1.33K      85B               149B          9B
MBbloomCF            1   1.82%    1.00M    1.00M                 -           - 
hash                 1   1.82%     104B     104B           3 fields        3.00
stream               3   5.45%   16.56K    5.52K          8 entries        2.67
TDIS-TYPE            2   3.64%   19.19K    9.59K                 -           - 
TopK-TYPE            1   1.82%  111.58K  111.58K                 -           - 
set                  2   3.64%     208B     104B          5 members        2.50
CMSk-TYPE            1   1.82%  140.68K  140.68K                 -           - 
zset                 2   3.64%     304B     152B         11 members        5.50
ReJSON-RL           25  45.45%   15.38K     629B                 -           - 
```

## 4.2 使用 `memory usage` 命令

使用该命令查看 Key 的内存占用大小，需要注意的是，除了 String 类型，其他类型的内存占用都是采用了采样的方式进行的。

```shell
MEMORY USAGE key [SAMPLES count]
```

## 4.3 离线使用开源工具

### 4.3.1 [GitHub - sripathikrishnan/redis-rdb-tools: Parse Redis dump.rdb files, Analyze Memory, and Export Data to JSON](https://github.com/sripathikrishnan/redis-rdb-tools)
### 4.3.2 [GitHub - xueqiu/rdr](https://github.com/xueqiu/rdr)

# 5 如何优化 Big keys？

## 5.1 压缩

如果是数据大，可以通过压缩缩小内存占用

## 5.2 拆分

将一个 Value 特别大的 Key 拆分为几个 Key。 
比如 Key 为 `123456789` 可以拆分成 `12345678:1` 、`12345678:2` 这样的。

## 5.3 删除

将 Big key 删除，4.0 版本以后要使用 `UNLINK` 安全删除。

## 5.4 失效过期数据定期清理

对过期的数据进行定期清理，比如 `Hash` 结构的过期 Key，可以通过定时任务 `HSCAN` 和 `HDEL` 清理过期的数据。

Redis 7.4 版本以后可以为 `Hash` 的 Key 单独设置过期时间了。

```shell
HEXPIRE key seconds [NX | XX | GT | LT] FIELDS numfields field
  [field ...]
```

## 5.5 监控

通过监控系统，监控 Redis 中的内存占用大小和网络带宽的占用大小，以及固定时间内的内存占用增长率，当超过设定的阈值的时候，进行报警通知处理。

# 6 Reference
* [Redis性能瓶颈揭秘：如何优化大key问题？](https://zhuanlan.zhihu.com/p/622474134)
* [如何找出优化大Key与热Key,产生的原因和问题_云数据库 Tair（兼容 Redis®）(Tair)-阿里云帮助中心](https://help.aliyun.com/zh/redis/user-guide/identify-and-handle-large-keys-and-hotkeys/)
* [云数据库 Redis热 Key 与 大 Key-实践教程-文档中心-腾讯云](https://cloud.tencent.com/document/product/239/89468)
* [5.3 Identifying Issues](https://redis.io/learn/operate/redis-at-scale/observability/identifying-issues)
* [深度干货｜一文详解 Redis 中 BigKey、HotKey 的发现与处理](https://developer.aliyun.com/article/788845)
* [Redis CLI Scan for big keys and memory usage \| Docs](https://redis.io/docs/latest/develop/tools/cli/#scan-for-big-keys-and-memory-usage)
* [MEMORY USAGE \| Docs](https://redis.io/docs/latest/commands/memory-usage/)
* [Prometheus OSS \| Redis exporter](https://grafana.com/oss/prometheus/exporters/redis-exporter/)