---
title: Redis 有哪些缓冲区？各自有什么用？
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-11
time: 16:51
aliases: 
done: false
---
Redis 有以下几种缓冲区：

* 客户端输出缓冲区
* 客户端输入缓冲区
* 主从同步-复制积压缓冲区

# 客户端输出缓冲区

Redis 配置文件中，明确了三种客户端输出缓冲区：
* normal，比如 redis-cli 的 MONITOR 等命令的执行
* replica，复制缓冲区，也就是常说的 `replicatino buffer`
* pubsub，订阅缓冲区，需要给消费者推送消息的缓冲区

配置：

```properties
# The syntax of every client-output-buffer-limit directive is the following:
#
# client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
#
# A client is immediately disconnected once the hard limit is reached, or if
# the soft limit is reached and remains reached for the specified number of
# seconds (continuously).
# So for instance if the hard limit is 32 megabytes and the soft limit is
# 16 megabytes / 10 seconds, the client will get disconnected immediately
# if the size of the output buffers reach 32 megabytes, but will also get
# disconnected if the client reaches 16 megabytes and continuously overcomes
# the limit for 10 seconds.
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```
## 客户端输出缓冲区溢出

* normal，获取数据时 Value 过大导致客户端输出缓冲区溢出
* 执行 `MONITOR` 命令持续写入缓冲区，长时间大流量导致最终 OOM
* 主从同步生成 RDB 时，增量命令写入 replica buffer，可能导致溢出

# 客户端输入缓冲区

输入缓冲区就是用来缓存客户端发过来的命令的，客户端输入缓冲区可以通过执行命令 `client list` 进行查看:

```shell
id=8 addr=127.0.0.1:41572 laddr=127.0.0.1:6379 fd=11 name= age=9 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=5 obl=0 oll=0 omem=0 tot-mem=22298 events=r cmd=client|list user=default redir=-1 resp=2
```

* `cmd`，客户端最新执行的命令
* `qbuf`，已使用空闲输入缓冲区大小
* `qbuf-free`，空闲输入缓冲区大小

详情：[使用 client list 命令查看客户端](使用%20client%20list%20命令查看客户端.md)
## 客户端输入缓冲区溢出
Redis 主从同步传输 RDB 结束后，Master 发送命令给 Replica 的输入缓冲区，此时 Replica 可能还在处理 RDB，如果并发写很大可能会导致 Replica 输入缓冲区溢出，关闭连接。

客户端输入缓冲区大小无法更改～～～，所以避免输入缓冲区溢出只能从命令发送方和自身处理命令效率着手。

# 复制积压缓冲区

复制积压缓冲区，`replica backlog buffer`，环形缓冲区，用于主从部分同步，
详情：[7 `repl-backlog-size 1mb`](Redis%20主从复制配置.md#7%20`repl-backlog-size%201mb`)

# 共享缓冲区

Redis 7.0 进行了优化，针对输出缓冲区和复制积压缓冲区，`replica buffer` 和 `replica backlog buf`，多个 Replica 服务器实例的 `replica buffer` 共享一个，且主从复制的 `replica backlog buf` 也共享一个，减少了内存占用。
# Reference
*  [Redis缓冲区不会还有人不知道吧？-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2212515)
* [后端 - 面试官：Redis中的缓冲区了解吗 - 七淅在学Java - SegmentFault 思否](https://segmentfault.com/a/1190000041572572)
* [21 \| 缓冲区：一个可能引发“惨案”的地方-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/291277)