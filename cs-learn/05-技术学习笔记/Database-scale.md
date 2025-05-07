

### Federation

Federation (or functional partitioning) splits up databases by function. For example, instead of a single, monolithic database, you could have three databases: **forums**, **users**, and **products**, resulting in less read and write traffic to each database and therefore less replication lag. Smaller databases result in more data that can fit in memory, which in turn results in more cache hits due to improved cache locality. With no single central master serializing writes you can write in parallel, increasing throughput.

### Sharding

Sharding distributes data across different databases such that each database can only manage a subset of the data. Taking a users database as an example, as the number of users increases, more shards are added to the cluster.






### Denormalization

**Denormalization（反规范化）** 是数据库设计中的一种策略，其核心思想是**通过增加冗余数据来提高读取性能**，即将原本规范化（Normalization）后分散在多个表中的数据，**重新合并、复制或嵌套**在一起，以减少查询时的关联操作（如 JOIN）。


#### 为何采用 Denormalization？——动因解析

在实际业务系统中，**性能瓶颈常常出现在数据读取（Read-heavy）的场景**。当一个用户请求需要跨多个表 JOIN 才能得出结果时，尤其是在大数据量和分布式数据库中，这样的查询代价非常高。

于是，反规范化应运而生，主要应对如下场景：

- **高频读场景**：如推荐系统、产品展示页、仪表盘。
    
- **分布式数据库/NoSQL 架构**：如 MongoDB、Cassandra，不支持或不擅长 JOIN。
    
- **数据仓库**：为支持多维分析，倾向于反规范化的星型/雪花模型。
    
- **缓存系统结合**：将反规范化后的数据结构写入 Redis 等缓存中。

#### 常见的形式

- **合并表**：如把订单和订单项合并成一个表，避免 JOIN。
    
- **嵌套数据结构**：如在 JSON 文档中嵌套子对象（NoSQL 常见）。
    
- **字段复制**：在多个表中重复某个字段（如用户名），避免多表关联。
    
- **预聚合（Pre-aggregation）**：如提前计算总和、平均值等聚合指标。


#### 适用场景

|                   |
| ----------------- |
| 高性能读取场景（如报表、推荐系统） |



### SQL-Tuning

**SQL 调优**是指通过优化 SQL 语句的写法、查询逻辑、索引使用方式等手段，以**降低查询延迟、减少资源消耗、提升性能**。

常见 SQL Tuning 技术

| 技术手段            | 说明                   | 示例                                    |
| --------------- | -------------------- | ------------------------------------- |
| 使用索引            | 建立适当索引，<u>避免全表扫描</u> | `WHERE email = 'abc'` 应该对 `email` 建索引 |
| 避免 SELECT *     | 明确字段减少数据传输           | 改为 `SELECT name, age`                 |
| 减少嵌套子查询         | 子查询可能重复执行            | 尽量用 JOIN 替代                           |
| 使用 LIMIT / 分页   | 控制数据量                | `LIMIT 100 OFFSET 0`                  |
| 拆解复杂查询          | 分步执行再聚合              | 用临时表或 WITH 语句                         |
| 分区表优化           | 按时间或地区分区，提高查询效率      | 按月分区日志表                               |
| 使用执行计划（EXPLAIN） | 诊断 SQL 是否使用索引        | `EXPLAIN SELECT ...`                  |





















### 物化视图   Materialized View

In [computing](https://en.wikipedia.org/wiki/Computing "Computing"), a **materialized view** is a [database](https://en.wikipedia.org/wiki/Database "Database") object that contains the results of a [query](https://en.wikipedia.org/wiki/Query_\(databases\) "Query (databases)"). For example, it may be a local copy of data located remotely, or may be a subset of the rows and/or columns of a table or [join](https://en.wikipedia.org/wiki/Join_\(SQL\) "Join (SQL)") result, or may be a summary using an [aggregate function](https://en.wikipedia.org/wiki/Aggregate_function "Aggregate function").

相对于常规的view， 虽然数据一致性没有那么好，但是极大的提高了查询性能，特别是在数据规模大，设计的数据库多时


**普通视图（View） vs. 物化视图（Materialized View）**

1. **普通视图 (View):**
    
    - 你可以把生成这份报告的复杂查询语句保存起来，起个名字，这就是一个“视图”。
    - **特点：** 它就像一个“快捷方式”或者“菜谱”。每次你访问这个视图（要这份报告），数据库都得**重新**按照这个“菜谱”（查询语句）去原始的几个大表格里捞数据、计算、汇总，然后把结果给你。
    - **优点：** 总是能看到最新的数据（因为是实时查询的），不额外占用存储空间（只存查询语句本身）。
    - **缺点：** 如果原始查询很复杂、涉及的数据量很大，那么每次访问视图都会很慢，而且会给数据库带来压力。
2. **物化视图 (Materialized View):**
    
    - 物化视图也是基于一个查询语句创建的。
    - **核心区别：** 它不仅仅是保存查询语句（“菜谱”），它还会**实际执行一次这个查询**，然后把查询的**结果集**像一张真实的表格一样**物理存储**起来。
    - **特点：** 它就像你按照“菜谱”做好了一大份菜，然后放进了冰箱（物理存储）。下次你要吃的时候，直接从冰箱里拿出来就行了，速度非常快。
    - **优点：** 查询速度极快！因为数据已经是预先计算好并存起来的，访问物化视图就像访问一张普通的表一样，避免了复杂的实时计算。特别适合那些查询慢但结果又需要频繁访问的场景（比如报表、仪表盘）。可以显著降低对原始表的查询压力。
    - **缺点：**
        - **存储空间：** 因为它存储了实际的数据，所以会占用额外的磁盘空间。
        - **数据新鲜度（关键！）：** 最大的问题在于，物化视图里的数据是某个时间点的“快照”。原始表的数据更新后，物化视图里的数据**不会自动实时更新**，它会变成“旧”数据（Stale Data）。你需要设定策略去**刷新（Refresh）**它，让它重新执行查询，用最新的结果覆盖掉旧的结果。



##### 物化视图的刷新
