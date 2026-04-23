# 10 - ElasticSearch

---

## 一、核心原理

1. ES 的整体架构？Node、Cluster、Index、Shard、Replica？
2. 倒排索引的原理？和正排索引的区别？
3. 文档写入流程？Refresh、Flush、Translog？
4. 文档搜索流程？Query Phase 和 Fetch Phase？
5. 近实时搜索（NRT）是怎么实现的？

## 二、索引与映射

1. Mapping 的作用？动态映射和显式映射？
2. 常用的字段类型？text vs keyword 的区别？
3. 分词器的作用？Standard、IK、自定义分词器？
4. 索引模板（Index Template）的使用？
5. 索引别名（Alias）的作用？

## 三、查询与聚合

1. Query DSL 的基本结构？
2. match、term、range、bool 查询的区别？
3. must、should、must_not、filter 的区别？filter 为什么更快？
4. 全文检索和精确查询的区别？
5. 聚合查询？Bucket 聚合、Metric 聚合、Pipeline 聚合？
6. 高亮搜索的实现？
7. 搜索建议（Suggest）的实现？

## 四、性能优化

1. 深度分页问题？from+size vs scroll vs search_after？
2. 索引设计的最佳实践？分片数量如何确定？
3. 写入性能优化？（批量写入、调整 refresh_interval、关闭副本）
4. 查询性能优化？（路由、预热、缓存）
5. 冷热架构是什么？如何实现？

## 五、数据同步

1. ES 和 MySQL 的数据同步方案？
2. 同步双写的问题？
3. Canal + MQ 的异步同步方案？
4. Logstash 同步方案？
5. 数据一致性如何保证？

## 六、实战场景

1. ES 在你的 AI 检索项目中是怎么用的？
2. 向量检索（kNN）在 ES 中的支持？
3. ES 集群的运维经验？节点角色划分？
4. ES 的安全机制？认证和授权？
