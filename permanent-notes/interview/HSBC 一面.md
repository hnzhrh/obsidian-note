
# 你知道的 Redis 的缓存淘汰策略有哪些？
* 不淘汰
* 所有 Key
	* LRU
	* LFU
* 过期 Key
	* TTL
	* LRU
	* LFU
# 如果一个 key 的过期时间没到有可能会被删除吗？

可能的，除非没有配置淘汰策略。

# 你对 Redis 做了什么优化？
设备号存储优化，长 ID 映射短 ID，Protobuf 压缩，Ziplist 编码压缩存储

# 什么是跳跃表？

Zset 的一种编码方式，使用空间换时间，根据多级索引模拟二分查找。

# 说一下你对微服务的理解？优缺点是啥？怎么平衡有多微？

从复杂系统中拆分出来的单一职责的服务，方便部署，可以做到技术异构。
缺点是可能拆的过细，导致蜘蛛网调用，不好维护。

根据具体业务理解，可以使用时间风暴拆聚合，如果依赖出现上下级调用，需要重新拆分或者合并，进行重构。

# 多个微服务调用涉及的数据一致性要怎么解决？

不太清楚，没有做过。我了解到的就分布式事务、幂等重试、事务消息

# RPC 依赖的上下游，对于接口的超时时间基于哪些因素去做设置？

考虑整体链路的 RT，防止整个链路被当前阻塞等待超时拖垮。

# 分享一个 MySQL 索引优化的经验

慢查询发现、优化覆盖索引

# 影响索引效率有哪些因素？

* 行数，索引再覆盖行数过多导致树过高，IO 次数增多，效率明显降低
* 列数或者索引列数据太大，也会导致树过高，IO 次数增多

# 建索引的区分度有多高？

不知道

# 有一个很长的字段，但需求里面就要去撞这个字段，这个场景有没有什么解决方案？

方案一：前缀索引
方案二：做一次 Hash，函数索引列

好像都不满意，提示了个什么图索引，听都没听过
