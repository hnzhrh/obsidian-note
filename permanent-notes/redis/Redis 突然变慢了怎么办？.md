---
title: Redis 突然变慢了怎么办？
tags:
  - permanent-note
  - middleware/redis/ops
date: 2025-03-12
time: 15:16
aliases: 
done: false
---
主线程阻塞 
1 全量查询命令(SMEMBERS key等命令) 或者是聚合统计命令
2 从库全量复制 先删除本库RDB文件(bigkey删除)和主库RDB加载
3 fork过程，所以fork后是使用子线程执行相应的操作，但fork过程中也是需要在主线程上执行的，若主线程上资源太多，导致fork变慢。
4 bigkey删除 ，4.0后才能进行lazy-free（延迟删除）,并且默认是关闭的，需要开启，并且开启后根据不同的配置，参数大小来进行判断是在主线程还是在子线程(异步)删除；所以说bigkeys可能不是在子线程删除。并且开启了lazy-free 直接使用del命令也是在主线程中删除，要是想在异步删除需要6.0后开启lazyfree-lazy-user-del命令。-->详情见该内容第16篇，K神的评论
  并且删除后释放的内存块添加到相应的位置,为后续管理和分配也会消耗资源，这个可通过该实战20篇那几个配置进行缓解
5 AOF写入 AOF重写，因为AOF重写需要fork(第3点)并且重写过程会占用大量的IO资源，重写过程中发生AOF写入(当AOF配置为always)会等待重写完成后才能进行写入；此时主线程发生阻塞，或者AOF配置为everysec(子进程写) 前1s的命令还为写入又写当前1s的也是进行等待。

6 Redis Cluster 方案 而且同时正好迁移的是 bigkey 的话，就会造成主线程的阻塞，因为 Redis Cluster 使用了同步迁移 

# Reference
* [mp.weixin.qq.com/s?\_\_biz=MzIyOTYxNDI5OA==&mid=2247484679&idx=1&sn=3273e2c9083e8307c87d13a441a267d7&chksm=e8beb2b2dfc93ba4c28c95fdcb62eefc529d6a4ca2b4971ad0493319adbf8348b318224bd3d9&scene=178&cur\_album\_id=1699766580538032128#rd](https://mp.weixin.qq.com/s?__biz=MzIyOTYxNDI5OA==&mid=2247484679&idx=1&sn=3273e2c9083e8307c87d13a441a267d7&chksm=e8beb2b2dfc93ba4c28c95fdcb62eefc529d6a4ca2b4971ad0493319adbf8348b318224bd3d9&scene=178&cur_album_id=1699766580538032128#rd)