---
title: Redis-aof
tags:
  - fleet-note
  - middleware/redis/persistence/aof
date: 2024-11-27
time: 11:19
aliases:
---
Redis AOF (Append Only File) 发生在写命令执行之后，将写命令写入文件中防止丢失。
AOF 不会阻塞当前线程，但是会阻塞下一个命令的执行。
AOF 刷盘有三种方案：
* Always，同步写回：每个写命令执行完，立马同步地将日志写回磁盘；
* Everysec，每秒写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘；
* No，操作系统控制的写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

过大的 AOF 日志文件会拖慢系统启动恢复数据的速度，Redis 提供了 AOF 重写的机制，对 AOF 文件进行处理。

Redis 7.0 之前重写的流程：
1. 主线程 fork bgrewriteaof 子线程
2. 子线程不影响主线程的情况下，把当前数据写成操作计入重写 AOF 日志
3. 主线程将新写入命令同时写入 AOF 缓冲区和重写 AOF 缓冲区
4. 等到数据重写完成后，重写 AOF 缓冲区也写入新的 AOF 文件
5. 替换旧的 AOF 文件
如此过程，当系统宕机时，哪怕 AOF 重写没有完成，也不会丢失数据，使用老的 AOF 文件恢复即可
Redis 7.0 以后流程：
```c
// aof.c
/* This is how rewriting of the append only file in background works:
 *
 * 1) The user calls BGREWRITEAOF
 * 2) Redis calls this function, that forks():
 *    2a) the child rewrite the append only file in a temp file.
 *    2b) the parent open a new INCR AOF file to continue writing.
 * 3) When the child finished '2a' exists.
 * 4) The parent will trap the exit code, if it's OK, it will:
 *    4a) get a new BASE file name and mark the previous (if we have) as the HISTORY type
 *    4b) rename(2) the temp file in new BASE file name
 *    4c) mark the rewritten INCR AOFs as history type
 *    4d) persist AOF manifest file
 *    4e) Delete the history files use bio
 */
```

请教一个问题: 关于Redis 7.0 以后混合持久化 Multipart AOF 重写的理解。
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
主线程写入增量命令，
Rewrite 线程重写当前数据 snapshot 到临时文件中，完成后修改文件名为 `appendonly.aof.2.base.rdb` , 修改 manifest 文件内容为：
```c
file appendonly.aof.2.base.rdb seq 2 type b
file appendonly.aof.2.incr.aof seq 2 type i
```
假设系统在 rewrite 正常结束前宕机，系统依然可以通过 manifest 重放数据。

> 问题1，Redis采用fork子进程重写AOF文件时，潜在的阻塞风险包括：fork子进程 和 AOF重写过程中父进程产生写入的场景，下面依次介绍。
> 
> 	a、fork子进程，fork这个瞬间一定是会阻塞主线程的（注意，fork时并不会一次性拷贝所有内存数据给子进程，老师文章写的是拷贝所有内存数据给子进程，我个人认为是有歧义的），fork采用操作系统提供的写实复制(Copy On Write)机制，就是为了避免一次性拷贝大量内存数据给子进程造成的长时间阻塞问题，但fork子进程需要拷贝进程必要的数据结构，其中有一项就是拷贝内存页表（虚拟内存和物理内存的映射索引表），这个拷贝过程会消耗大量CPU资源，拷贝完成之前整个进程是会阻塞的，阻塞时间取决于整个实例的内存大小，实例越大，内存页表越大，fork阻塞时间越久。拷贝内存页表完成后，子进程与父进程指向相同的内存地址空间，也就是说此时虽然产生了子进程，但是并没有申请与父进程相同的内存大小。那什么时候父子进程才会真正内存分离呢？“写实复制”顾名思义，就是在写发生时，才真正拷贝内存真正的数据，这个过程中，父进程也可能会产生阻塞的风险，就是下面介绍的场景。
> 
> 	b、fork出的子进程指向与父进程相同的内存地址空间，此时子进程就可以执行AOF重写，把内存中的所有数据写入到AOF文件中。但是此时父进程依旧是会有流量写入的，如果父进程操作的是一个已经存在的key，那么这个时候父进程就会真正拷贝这个key对应的内存数据，申请新的内存空间，这样逐渐地，父子进程内存数据开始分离，父子进程逐渐拥有各自独立的内存空间。因为内存分配是以页为单位进行分配的，默认4k，如果父进程此时操作的是一个bigkey，重新申请大块内存耗时会变长，可能会产阻塞风险。另外，如果操作系统开启了内存大页机制(Huge Page，页面大小2M)，那么父进程申请内存时阻塞的概率将会大大提高，所以在Redis机器上需要关闭Huge Page机制。Redis每次fork生成RDB或AOF重写完成后，都可以在Redis log中看到父进程重新申请了多大的内存空间。
# References
* [04 | AOF日志：宕机了，Redis如何避免数据丢失？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/271754)
* [Redis persistence | Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
* [【Redis】- Redis 7.0 Multi Part AOF的设计和实现\_redis multi part aof-CSDN博客](https://blog.csdn.net/weixin_42201180/article/details/128733771)
* [A new AOF persistence mechanism by chenyang8094 · Pull Request #9539 · redis/redis · GitHub](https://github.com/redis/redis/pull/9539#issuecomment-964737334)
* [Implement Multi Part AOF mechanism to avoid AOFRW overheads. by chenyang8094 · Pull Request #9788 · redis/redis · GitHub](https://github.com/redis/redis/pull/9788)
* [数据库内核月报](http://mysql.taobao.org/monthly/2018/12/06/)