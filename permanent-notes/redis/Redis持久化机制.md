---
title: Redis持久化机制
tags:
  - permanent-note
  - middleware/redis/persistence
date: 2024-12-06
time: 13:17
aliases:
---
Redis 在缓存场景中，由于 Redis 是内存数据库，如果重启会导致数据丢失，因此 Redis 需要提供必要的持久化工具，Redis 提供了 RDB 和 AOF 两种持久化方式。

# AOF（Append Only File） 日志

不同于 MySQL 数据库的 WAL 日志（Write Ahead Log），AOF 正好相反，先写入数据，再写入日志，AOF 写入的数据是可读的命令，这样做有两个好处：

1. 先执行命令再写入时不需要额外的命令错误检查开销
2. 不会阻塞当前命令，但可能会影响下一个操作

Redis 会将命令追加到 `server.aof_buf` 这个 sds 中，通过函数 `flushAppendOnlyFile` 调用 `write` 写入系统缓冲区中，根据写回策略，调用 `fedis_fsync` 将系统缓冲区刷入硬盘。

AOF 提供了三种写回策略：
* Always
* Everysec
* No
AOF 写回策略的选择需要 tradeoff，根据实际情况选择，性能损耗 Always >> Everysec > No。

AOF 文件随着时间的流逝，体积越来越大，追加写入性能也会越来越低，宕机后，Redis 从 AOF 文件恢复数据也会非常缓慢，因此 Redis 提供了 AOF 重写机制。

## AOF 文件重写

AOF 重写为了避免阻塞主线程，影响性能，通过后台子进程 `bgrewriteaof` 完成。执行重写时，主进程 `fork` 子进程采用 COW 机制避免一次性大量拷贝内存导致的阻塞问题，`fork` 时主进程拷贝页表的操作依然是阻塞的，实例越大，阻塞时间越长。子进程此时把内存中的数据以命令形式写入 AOF 文件中，此时主进程如果有流量写入已存在的 Key，主进程会真正拷贝内存数据，父子进程内存开始分离，各自拥有独立的内存空间。因此，AOF 文件重写过程中是有可能阻塞主进程的：
1. 主进程 `fork` 子进程时阻塞
2. COW 内存实际拷贝时，如果操作的是 bigkey，申请内存变大则可能会阻塞
3. 内存分配已页为单位默认 4 K，如果操作系统开启了内存大页机制（Huge Page，2 M），则父进程申请内存时造成延迟的概率会大大提高
在 AOF 文件重写期间，主进程会将命令写入两个缓冲区，一个是正常的 `aof_buf` ，用来恢复数据，另外也会写入到 `aof_rewrite_buf` 中。正常情况下 AOF 重写内存数据结束后，会追加写入 `aof_rewrite_buf` 的命令，这样重写后的 AOF 文件不会缺少重写期间新写入的数据，最后用新的 AOF 文件替代旧文件。

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
同时写入增量命令，Rewrite 线程重写当前数据 snapshot 到临时文件中，完成后修改文件名为 `appendonly.aof.2.base.rdb` , 修改 manifest 文件内容为：
```c
file appendonly.aof.2.base.rdb seq 2 type b
file appendonly.aof.2.incr.aof seq 2 type i
```
假设系统在 rewrite 正常结束前宕机，系统依然可以通过 manifest 重放数据。
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
# RDB （Redis DataBase）

Redis 也支持使用 RDB 做持久化，使用 RDB 的一个好处是二进制恢复数据快。RDB 也采用 COW 机制，因此也有阻塞的风险。且当数据写入占比过大时，会占用更多的内存，可能触发系统 swap，如果没有开启 swap，会直接触发 OOM，面临被系统 Kill 掉的风险。

# Reference
* [04 \| AOF日志：宕机了，Redis如何避免数据丢失？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/271754)
* [Redis AOF 持久化- Redis源码分析 - 我们](https://gsmtoday.github.io/2018/07/30/redis-01/)
* [Redis persistence \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
* [【Redis】- Redis 7.0 Multi Part AOF的设计和实现\_redis multi part aof-CSDN博客](https://blog.csdn.net/weixin_42201180/article/details/128733771)
* [A new AOF persistence mechanism by chenyang8094 · Pull Request #9539 · redis/redis · GitHub](https://github.com/redis/redis/pull/9539#issuecomment-964737334)
* [Implement Multi Part AOF mechanism to avoid AOFRW overheads. by chenyang8094 · Pull Request #9788 · redis/redis · GitHub](https://github.com/redis/redis/pull/9788)