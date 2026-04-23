# 09 - Redis

---

## 一、数据结构

1. Redis 的五种基础数据类型？各自的使用场景？
2. Redis 底层数据结构？SDS、ziplist、quicklist、skiplist、hashtable、intset？
3. String 的底层实现？SDS 和 C 字符串的区别？
4. Hash 的底层实现？什么时候从 ziplist 转为 hashtable？
5. Sorted Set 为什么用跳表而不是红黑树？
6. Redis 的新数据类型？Stream、HyperLogLog、Bitmap、GEO？

## 二、持久化

1. RDB 持久化的原理？触发方式？优缺点？
2. AOF 持久化的原理？三种写回策略？AOF 重写？
3. RDB 和 AOF 的区别？如何选择？
4. 混合持久化是什么？怎么配置？
5. Redis 重启时如何恢复数据？

## 三、缓存问题

1. 缓存穿透是什么？怎么解决？（布隆过滤器、空值缓存）
2. 缓存击穿是什么？怎么解决？（互斥锁、逻辑过期）
3. 缓存雪崩是什么？怎么解决？（随机过期时间、多级缓存）
4. 缓存和数据库的一致性问题？Cache Aside、Read/Write Through、Write Behind？
5. 先更新数据库还是先删缓存？延迟双删？
6. 热 Key 问题怎么处理？
7. 大 Key 问题怎么排查和解决？

## 四、高可用架构

1. Redis 主从复制的原理？全量同步和增量同步？
2. 哨兵（Sentinel）模式的原理？故障转移流程？
3. Redis Cluster 的原理？哈希槽（Hash Slot）分配？
4. Redis Cluster 的节点通信？Gossip 协议？
5. Redis Cluster 如何处理节点故障？
6. Redis 集群方案的对比？主从 vs 哨兵 vs Cluster？

## 五、分布式锁

1. Redis 实现分布式锁的方式？SETNX + EXPIRE？
2. Redisson 分布式锁的原理？看门狗机制？
3. RedLock 算法是什么？有什么争议？
4. Redis 分布式锁和 ZooKeeper 分布式锁的对比？

## 六、性能优化与运维

1. Redis 为什么这么快？单线程模型？IO 多路复用？
2. Redis 6.0 的多线程是怎么回事？和单线程有什么区别？
3. Redis 的内存淘汰策略有哪些？LRU vs LFU？
4. Redis 的过期删除策略？惰性删除 + 定期删除？
5. Pipeline 的作用和使用场景？
6. Redis 事务和 Lua 脚本的区别？
7. Redis 的内存优化手段？

## 七、实战场景

1. Redis 在你项目中的具体使用场景？
2. 如何用 Redis 实现排行榜？
3. 如何用 Redis 实现限流？（滑动窗口、令牌桶）
4. 如何用 Redis 实现延迟队列？
5. 如何用 Redis 实现分布式 Session？
