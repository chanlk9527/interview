# 08 - MySQL

---

## 一、存储引擎

1. InnoDB 和 MyISAM 的区别？为什么推荐 InnoDB？
2. InnoDB 的架构？Buffer Pool、Redo Log、Undo Log、Binlog？
3. 一条 SQL 的执行流程？（连接器 → 分析器 → 优化器 → 执行器）

## 二、索引

1. 索引的数据结构？为什么用 B+ 树而不是 B 树、Hash、红黑树？
2. 聚簇索引和非聚簇索引（二级索引）的区别？
3. 回表查询是什么？如何避免回表？覆盖索引？
4. 联合索引的最左前缀原则？
5. 索引下推（Index Condition Pushdown）是什么？
6. 前缀索引的使用场景？
7. 索引失效的常见场景？（函数、隐式转换、OR、LIKE '%xx'、不等于等）
8. 如何分析和优化慢查询？EXPLAIN 各字段的含义？
9. 什么时候不适合建索引？

## 三、事务

1. 事务的 ACID 特性？各自是如何保证的？
2. 四种事务隔离级别？各自解决了什么问题？
3. MVCC 的实现原理？ReadView、Undo Log 版本链？
4. 当前读和快照读的区别？
5. 可重复读（RR）隔离级别下能完全避免幻读吗？

## 四、锁机制

1. MySQL 的锁分类？全局锁、表级锁、行级锁？
2. InnoDB 的行锁类型？记录锁、间隙锁、临键锁（Next-Key Lock）？
3. 什么情况下行锁会升级为表锁？
4. 死锁的产生条件？如何排查和解决死锁？
5. 乐观锁和悲观锁的区别？在 MySQL 中如何实现？
6. 意向锁的作用？

## 五、日志系统

1. Redo Log 和 Binlog 的区别？各自的作用？
2. 两阶段提交（2PC）是什么？为什么需要？
3. Undo Log 的作用？和 MVCC 的关系？
4. Binlog 的三种格式？Statement、Row、Mixed？

## 六、SQL 优化

1. 大表查询优化的常见手段？
2. 深度分页（LIMIT 100000, 10）的优化方案？
3. COUNT(*) 和 COUNT(1) 和 COUNT(列名) 的区别？
4. JOIN 的原理？Nested Loop Join、Hash Join？
5. 子查询 vs JOIN 的性能对比？
6. ORDER BY 的优化？filesort 和 index sort？

## 七、分库分表

1. 什么时候需要分库分表？垂直拆分 vs 水平拆分？
2. 分片键如何选择？
3. 分库分表后的问题？（跨库 JOIN、分布式事务、全局 ID、数据迁移）
4. 常用的分库分表中间件？ShardingSphere、MyCat？
5. 读写分离的实现方案？主从复制的原理？主从延迟怎么处理？
