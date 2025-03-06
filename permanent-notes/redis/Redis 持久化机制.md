---
title: Redis 持久化机制
tags:
  - permanent-note
  - middleware/redis/persistence
date: 2024-12-06
time: 13:17
aliases:
---

# 1 AOF

## 1.1 AOF 的实现原理

AOF 先执行命令，再写入日志，记录的是操作。由于是先执行命令，所以不需要语法检查，也不会阻塞当前操作。

AOF 的风险主要有两个：
* 后写日志可能导致数据丢失，比如命令执行成功，写入日志时宕机，用该日志恢复后则会造成数据丢失
* 不会阻塞当前操作，但会影响到下一个命令的执行，影响整体的吞吐量

## 1.2 AOF 三种写回策略

| 策略       | 写回时机   | 优点    | 缺点          |
| -------- | ------ | ----- | ----------- |
| Always   | 同步写回   | 可靠性最高 | 对性能影响极大     |
| Everysec | 每秒写回   | 性能适中  | 最多丢失 1 s 数据 |
| No       | 操作系统决定 | 性能最好  | 宕机时丢失数据最多   |

## 1.3 AOF 重写机制

AOF 文件中由于记录了 Redis 命令，文件会随着时间越来越大，写入文件的效率越低，如果宕机，使用 AOF 文件恢复速度也会很慢，因此 AOF 提供了重写机制。

AOF 重写机制通过分析命令，合并命令后只写入当前状态来减少 AOF 文件体积。

### 1.3.1 AOF 触发时机

* 手动触发
	* 调用了命令 `BGREWRITEAOF`
* 自动触发
	* `auto-aof-rewrite-percentage 100`，文件大小是上次 AOF 文件的两倍时触发
	* `auto-aof-rewrite-min-size 64mb`，AOF 文件超过 64 mb 时触发

### 1.3.2 AOF 重写流程

* 触发了 AOF 重写
* 主进程 `fork` 出子进程 `bgrewriteaof`，此过程阻塞，通过写时复制 COW 技术，`bgrewriteaof` 进程不会影响到主进程内存
* `bgrewriteaof` 进程重写 AOF 文件
* 如果父进程操作的是一个已经存在的key，父进程真正拷贝这个key对应的内存数据，申请新的内存空间，父子内存空间开始分裂
* 主进程把新来的操作写入两个日志缓冲区中
	* `server.aof_buf`，新命令写入老 AOF 文件，如果重写过程中宕机任然可以保证数据的完整性
	* `aof_rewrite_buf`，新命令追加写入新 AOF 文件
* 重写完成

### 1.3.3 AOF 重写风险

AOF 文件重写过程中是有可能阻塞主进程的：
1. 主进程 `fork` 子进程时阻塞
2. 父子线程内存分裂操作已有的 `key` 时，如果拷贝的是 `bigkey`，申请内存变大则可能会阻塞
3. 内存分配页默认大小 `4K`，如果操作系统开启了内存大页机制（`Huge Page 2M`），则父进程申请内存时造成延迟的概率会大大提高，Redis 建议关闭大页


### 1.3.4 Redis 7.0 对 AOF 重写的优化

Redis 7.0 以后对 `aof_rewrite_buf` 进行了优化，以上流程可以看到新写入的数据命令会存两份一模一样的在 `aof_buf` 和 `aof_rewrite_buf` 中，是否可以避免这样？

Redis 7.0 以后引入了 MP-AOF 进行优化。

假设已持久化文件为：

```c
file appendonly.aof.1.base.rdb seq 1 type b
file appendonly.aof.1.incr.aof seq 1 type i
```

开始 AOFRW 后，主线程打开 `appendonly.aof.2.incr.aof` 文件，写入 manifest 文件：

```c
file appendonly.aof.1.base.rdb seq 1 type b
file appendonly.aof.1.incr.aof seq 1 type i
file appendonly.aof.2.incr.aof seq 2 type i
```

同时写入增量命令， 重写当前数据 snapshot 到临时文件中，完成后修改文件名为 `appendonly.aof.2.base.rdb` , 修改 manifest 文件内容为：
```c
file appendonly.aof.2.base.rdb seq 2 type b
file appendonly.aof.2.incr.aof seq 2 type i
```

如果重写时系统宕机了，Redis 依然可以根据 manifest 文件恢复数据，只需要加载 base 和所有的 incr 文件即可。
```c
file appendonly.aof.1.base.rdb seq 1 type b
file appendonly.aof.1.incr.aof seq 1 type i
file appendonly.aof.2.incr.aof seq 2 type i
```

源码注释的流程：
 * 1) The user calls `BGREWRITEAOF`
 * 2) Redis calls this function, that `forks ()`:
 *    2 a) the child rewrite the append only file in a temp file.
 *    2 b) the parent open a new `INCR AOF` file to continue writing.
 * 3) When the child finished '2 a' exists.
 * 4) The parent will trap the exit code, if it's OK, it will:
 *    4 a) get a new BASE file name and mark the previous (if we have) as the HISTORY type
 *    4 b) rename (2) the temp file in new BASE file name
 *    4 c) mark the rewritten INCR AOFs as history type
 *    4 d) persist AOF manifest file
 *    4 e) Delete the history files use bio

## 1.4 AOF 相关配置

```properties
# 是否开启AOF
appendonly no

# AOF 文件名，7.0 以后 Base 文件和 Incr 文件基础名
appendfilename "appendonly.aof"
# AOF文件存储路径
appenddirname "appendonlydir"

# 配置刷盘策略
# appendfsync always
appendfsync everysec
# appendfsync no

# 启用后，BGSAVE 和 BGREWRITEAOF 期间禁用fsync() 避免阻塞，等同于降级刷盘策略为appendonly no
no-appendfsync-on-rewrite no

# AOF rewrite 阈值配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 配置为 yes 时实例启动初始化时会加载受损的 AOF 文件
aof-load-truncated yes

# AOF基础文件采用 RDB 格式，生成更快并且占用空间更小
aof-use-rdb-preamble yes

# 时间戳 AOF 命名，提供point-in-time 恢复能力，会改变 AOF 文件格式
aof-timestamp-enabled no
```

完整配置文件参考 [4 AOF](Redis%20配置.md#4%20AOF)


# 2 RDB

## 2.1 RDB 的实现原理

RDB 通过内存全量快照进行保存，Redis 提供了两个命令来生成 RDB 文件，分别是 `save` 和 `bgsave`：
* `save`：在主线程中执行，会导致阻塞
* `bgsave`：创建一个子进程，专门用于写入 RDB 文件，避免了主线程的阻塞，这也是 Redis RDB 文件生成的默认配置。

Redis 借助操作系统提供的写时复制技术（`Copy-On-Write, COW`），在执行快照的同时，正常处理写操作。

## 2.2 RDB 相关配置

```properties
# 配置 RDB 触发条件，Ns 发生 X 次数据变动
# save <seconds> <changes> [<seconds> <changes> ...]
save 3600 1 300 100 60 10000

# 开启该配置则最近一次bgsave失败后实例会拒绝新的写入请求，确保数据丢失不被放大
stop-writes-on-bgsave-error yes

# 使用LZF压缩String对象，开启有助于节省磁盘空间，并且在网络传输过程中也能减少带宽占用
rdbcompression yes

# 使用 CRC64 校验 RDB 文件，在生成和加载 RDB 文件时有 10% 左右的性能损耗
rdbchecksum yes

# Enables or disables full sanitization checks for ziplist and listpack etc when
# loading an RDB or RESTORE payload. This reduces the chances of a assertion or
# crash later on while processing commands.
# Options:
#   no         - Never perform full sanitization
#   yes        - Always perform full sanitization
#   clients    - Perform full sanitization only for user connections.
#                Excludes: RDB files, RESTORE commands received from the master
#                connection, and client connections which have the
#                skip-sanitize-payload ACL flag.
# The default should be 'clients' but since it currently affects cluster
# resharding via MIGRATE, it is temporarily set to 'no' by default.
#
# sanitize-dump-payload no

# RDB 文件名
dbfilename dump.rdb

# 只有AOF和RDB都禁用时才会有效
# 完成主从复制后不会删除 RDB 文件
rdb-del-sync-files no

# 工作目录
dir ./
```

完整配置参考 [3 内存 Snapshot](Redis%20配置.md#3%20内存%20Snapshot)

## 2.3 RDB 触发时机

### 2.3.1 主动执行命令 `save`

### 2.3.2 主动执行命令 `bgsave`

### 2.3.3 达到 RDB 触发阈值

### 2.3.4 实例正常关闭时

### 2.3.5 主从全量复制时执行一次 RDB 持久化

# 3 混合持久化

默认开启混合持久化。AOF 文件会使用 RDB 格式加快预加载速度。

```properties
# AOF基础文件采用 RDB 格式，生成更快并且占用空间更小
aof-use-rdb-preamble yes
```

需要注意的是，这个配置并不会影响 RDB 的文件生成，两者之间是独立的。

# 4 持久化带来的风险

持久化风险主要来自于写时复制机制（COW），当 Redis 主进程 `fork` 子进程时会拷贝页表，这期间是阻塞的，如果主进程写操作了已有数据，主进程需要拷贝物理内存，这期间也是阻塞的，如果拷贝的还是 `bigkey`，则风险更大。因此在持久化期间，如果有大量写操作，会导致父子进程内存分裂，如果开启了 `swap` 机制，则性能大幅下降，如果没有开启则有可能 `OOM`。另外需要注意的是 Redis 的配置最大内存是不包括持久化时 COW 的内存的，因此配置 `maxmemory` 需要预留一定空间内存用于持久化。

## 4.1 写时复制 COW 的双重阻塞

### 4.1.1 `fork` 子进程的页表拷贝阻塞

 Redis 主进程调用 `fork()` 生成子进程时，需完整拷贝进程页表（虚拟内存映射关系）。此过程**主线程阻塞**，耗时与实例内存容量正相关（单 GB 内通常在毫秒级，但百 GB 级实例可能达秒级）。  

### 4.1.2 主进程修改已存在的数据触发物理内存拷贝

#### 4.1.2.1 `bigkey` 可能导致阻塞

子进程生成后，若主进程**修改已存在的键值数据**，COW 机制会强制拷贝被修改数据的物理内存页（原内存页由子进程只读占用）。此过程**主线程再次阻塞**，阻塞时长与拷贝量成正比，若涉及 `bigkey`（如 MB 级 Hash）或高频写入，可能引发严重延迟。

#### 4.1.2.2 内存大页可能导致阻塞

如果操作系统开启了内存大页机制(Huge Page，页面大小2M)，那么父进程申请内存时阻塞的概率将会大大提高，所以在Redis机器上需要关闭Huge Page机制。

#### 4.1.2.3 内存碎片可能导致阻塞

高内存碎片率会增加 COW 需拷贝的离散页数量。可通过 `info memory` 监控 `mem_fragmentation_ratio>1.5` 时建议重启或使用 `activedefrag `。


## 4.2 COW 的内存分裂造成的资源挤占

### 4.2.1 内存膨胀

持久化期间（如 RDB 生成或 AOF 重写），若主进程持续写入，父子进程内存页逐渐分裂，导致**实际内存占用量 ≈ 父进程数据 + 子进程副本**。极端情况下，总内存可能接近**2 倍数据集大小**。  

### 4.2.2 SWAP 和 OOM 的 trade-off

如果开启了 `SWAP` 则会造成 Redis 性能骤降，如果禁用了 `SWAP` 则内存超限 OOM，可能会造成进程被操作系统强制终止。

如果要禁用 `SWAP`，则基本要确保 $物理内存 > (数据内存占用 * 2 + 系统预留内存)$，尽量避免 OOM 导致进程被操作系统杀死。

### 4.2.3 `maxmemory` 配置

Redis 的 `maxmemory` 不包括 `COW` 产生的内存，如果没有预留缓冲内存空间，极易造成 `SWAP` 和 `OOM`。

# 5 Reference
* [04 \| AOF日志：宕机了，Redis如何避免数据丢失？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/271754)
* [05 \| 内存快照：宕机后，Redis如何实现快速恢复？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/271839)
* [VLDB 顶会论文 Async-fork 解读与 Redis 实践 ｜ 得物技术](https://mp.weixin.qq.com/s/c7OFBEVUfeJ5YlWE5dCyhg)
* [Redis AOF 持久化- Redis源码分析 - 我们](https://gsmtoday.github.io/2018/07/30/redis-01/)
* [Redis persistence \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
* [【Redis】- Redis 7.0 Multi Part AOF的设计和实现\_redis multi part aof-CSDN博客](https://blog.csdn.net/weixin_42201180/article/details/128733771)
* [A new AOF persistence mechanism by chenyang8094 · Pull Request #9539 · redis/redis · GitHub](https://github.com/redis/redis/pull/9539#issuecomment-964737334)
* [Implement Multi Part AOF mechanism to avoid AOFRW overheads. by chenyang8094 · Pull Request #9788 · redis/redis · GitHub](https://github.com/redis/redis/pull/9788)
* [DeepSeek - Into the Unknown](https://chat.deepseek.com/a/chat/s/a0f79063-212c-4864-ab0e-b3c16ebb8002)