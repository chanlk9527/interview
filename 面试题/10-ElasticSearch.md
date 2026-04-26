# 10 - ElasticSearch

---

## 一、核心原理

1. ES 的整体架构？Node、Cluster、Index、Shard、Replica？

**回答：**

ES 的整体架构是一个分布式搜索和分析引擎，核心概念如下：

- **Cluster（集群）**：由一个或多个 Node 组成，共享同一个集群名称，协同工作提供数据存储和搜索能力。集群有一个 Master 节点负责管理集群状态（如索引创建、节点加入/离开）。
- **Node（节点）**：集群中的一个 ES 实例。节点角色包括：
  - Master Node：负责集群元数据管理和分片分配决策
  - Data Node：存储数据并执行搜索和聚合操作
  - Coordinating Node：接收客户端请求，转发到相关分片，汇总结果返回
  - Ingest Node：数据预处理（类似 pipeline）
- **Index（索引）**：类似于数据库中的"表"，是文档的逻辑集合。每个索引有自己的 Mapping 和 Settings。
- **Shard（分片）**：索引被水平拆分为多个分片，每个分片是一个独立的 Lucene 索引。分片使得数据可以分布在多个节点上，实现水平扩展。
- **Replica（副本）**：每个主分片（Primary Shard）可以有零个或多个副本分片。副本的作用：一是提高数据可用性（主分片所在节点宕机时副本可以提升为主分片）；二是提高查询吞吐量（查询可以在副本上并行执行）。

一个典型的架构示例：一个集群有 3 个节点，一个索引设置 3 个主分片、1 个副本，那么总共有 6 个分片（3 主 + 3 副本），分布在 3 个节点上。
2. 倒排索引的原理？和正排索引的区别？

**回答：**

**倒排索引（Inverted Index）**：
- 核心思想：从"词项"到"文档"的映射。先对文档内容进行分词，然后建立 **词项 → 文档ID列表** 的映射关系。
- 结构包含两部分：
  - **Term Dictionary（词项字典）**：所有不重复的词项，通常用 B+ 树或 FST（Finite State Transducer）存储，支持快速查找。
  - **Posting List（倒排列表）**：每个词项对应的文档ID列表，还可能包含词频（TF）、位置信息（Position）等。
- 例如：文档1="Java 面试"，文档2="Java 基础"，则倒排索引为：Java → [1, 2]，面试 → [1]，基础 → [2]。

**正排索引（Forward Index）**：
- 从"文档"到"内容"的映射，即 **文档ID → 文档内容**。
- ES 中的 `_source` 字段和 `doc_values` 就是正排索引的体现。

**区别：**

| 对比项 | 正排索引 | 倒排索引 |
|--------|---------|---------|
| 映射方向 | 文档 → 词项 | 词项 → 文档 |
| 适用场景 | 根据文档ID获取内容 | 根据关键词搜索文档 |
| 查询效率 | 全文检索慢（需遍历所有文档） | 全文检索快（直接定位文档） |
| ES中的体现 | _source、doc_values | 倒排索引文件 |
3. 文档写入流程？Refresh、Flush、Translog？

**回答：**

ES 文档写入流程：

1. **客户端发送写入请求**到协调节点，协调节点根据文档 `_id` 的路由算法（`hash(_id) % 主分片数`）确定目标主分片。
2. **请求转发到主分片所在节点**，主分片执行写入操作。
3. **写入 In-Memory Buffer**：文档先写入内存缓冲区，此时文档还不可搜索。
4. **同时写入 Translog**：为了防止数据丢失，写入操作会同时追加到 Translog（事务日志）。
5. **主分片写入成功后，并行复制到副本分片**，所有副本确认后返回客户端成功。

三个关键机制：

- **Refresh（刷新）**：
  - 默认每 1 秒执行一次，将 In-Memory Buffer 中的数据写入一个新的 Segment（Lucene 段）到文件系统缓存（OS Cache）。
  - Refresh 后文档变为可搜索状态，这就是 ES "近实时"（NRT）的原因——最多有 1 秒延迟。
  - Refresh 不会清除 Translog。

- **Translog（事务日志）**：
  - 类似于数据库的 WAL（Write-Ahead Log），每次写入操作都会记录到 Translog。
  - 作用是在节点宕机时恢复还未持久化到磁盘的数据。
  - 默认每次写入请求后 fsync 到磁盘（`request` 级别持久化）。

- **Flush（冲刷）**：
  - 将文件系统缓存中的 Segment 通过 fsync 真正持久化到磁盘。
  - 清空 Translog（因为数据已经安全落盘）。
  - 默认当 Translog 达到 512MB 或每 30 分钟触发一次。

整体流程：写入 → Buffer + Translog → Refresh（可搜索）→ Flush（持久化）。
4. 文档搜索流程？Query Phase 和 Fetch Phase？

**回答：**

ES 的搜索采用 **两阶段查询**（scatter-gather 模式）：

**Query Phase（查询阶段）**：
1. 客户端发送搜索请求到协调节点。
2. 协调节点将请求广播到索引的所有相关分片（主分片或副本分片，每组选一个）。
3. 每个分片在本地执行查询，返回一个**轻量级结果集**：仅包含文档 ID 和排序值（如 `_score`），不包含文档内容。
4. 协调节点收集所有分片的结果，进行**全局排序和分页**，确定最终需要返回的文档 ID 列表。

**Fetch Phase（获取阶段）**：
1. 协调节点根据 Query Phase 确定的文档 ID 列表，向相关分片发送 multi-get 请求。
2. 各分片根据文档 ID 从 `_source` 中获取完整文档内容返回。
3. 协调节点汇总所有文档，返回给客户端。

**为什么分两个阶段？**
- 避免每个分片都返回完整文档内容，减少网络传输量。
- Query Phase 只传输 ID 和分数，非常轻量；Fetch Phase 只获取最终需要的少量文档。
- 例如查询 `from=0, size=10`，如果有 5 个分片，Query Phase 每个分片返回 top 10 的 ID（共 50 个），协调节点排序后取 top 10，Fetch Phase 只需获取这 10 个文档的完整内容。
5. 近实时搜索（NRT）是怎么实现的？

**回答：**

ES 的近实时搜索（Near Real-Time）是通过 **Refresh 机制** 实现的：

1. 文档写入后先进入 In-Memory Buffer，此时不可搜索。
2. 默认每隔 **1 秒**执行一次 Refresh 操作，将 Buffer 中的数据生成一个新的 **Segment** 写入文件系统缓存（OS Page Cache）。
3. 新的 Segment 一旦生成，就会被打开供搜索使用，文档变为可搜索状态。

关键点：
- Refresh 操作不涉及磁盘 I/O（只写到 OS Cache），所以开销较小。
- 这意味着从文档写入到可搜索，最多有 **1 秒的延迟**，这就是"近实时"的含义。
- 可以通过 `index.refresh_interval` 调整 Refresh 间隔。设为 `-1` 可以禁用自动 Refresh（适用于大批量导入场景）。
- 也可以手动调用 `POST /index/_refresh` 强制刷新，但频繁调用会影响性能。

底层原理是利用了 Lucene 的 **Segment 不可变性**：每次 Refresh 生成新的 Segment，旧的 Segment 不会被修改，通过打开新的 IndexSearcher 来包含新 Segment，从而实现近实时搜索。

## 二、索引与映射

1. Mapping 的作用？动态映射和显式映射？

**回答：**

**Mapping** 是 ES 中定义文档结构的方式，类似于数据库的表结构定义（Schema），它定义了：
- 索引中有哪些字段
- 每个字段的数据类型（text、keyword、integer、date 等）
- 字段如何被索引和存储（是否分词、是否存储原始值等）

**动态映射（Dynamic Mapping）**：
- ES 默认开启，当写入文档包含未定义的字段时，ES 会自动推断字段类型并创建映射。
- 推断规则：字符串 → text + keyword 子字段，整数 → long，浮点数 → float，布尔 → boolean，日期格式字符串 → date。
- 优点：开箱即用，方便快速开发。
- 缺点：类型推断可能不准确（如数字字符串 "123" 可能被映射为 text），且映射一旦创建**字段类型不可修改**。

**显式映射（Explicit Mapping）**：
- 在创建索引时手动定义 Mapping，精确控制每个字段的类型和索引方式。
- 生产环境推荐使用显式映射，避免动态映射带来的不确定性。
- 可以通过 `dynamic: strict` 禁止自动创建新字段，写入未定义字段时直接报错。

**最佳实践**：生产环境使用显式映射 + `dynamic: strict`，开发环境可以用动态映射快速迭代。
2. 常用的字段类型？text vs keyword 的区别？

**回答：**

**常用字段类型：**
- **字符串**：`text`（全文检索）、`keyword`（精确匹配）
- **数值**：`integer`、`long`、`float`、`double`
- **日期**：`date`
- **布尔**：`boolean`
- **对象**：`object`（JSON 对象）、`nested`（嵌套对象，独立索引）
- **特殊类型**：`geo_point`（地理坐标）、`ip`、`completion`（自动补全）、`dense_vector`（向量）

**text vs keyword 的区别：**

| 对比项 | text | keyword |
|--------|------|---------|
| 是否分词 | ✅ 写入时会经过分词器分词 | ❌ 不分词，作为整体存储 |
| 适用场景 | 全文检索（如文章内容、商品描述） | 精确匹配、排序、聚合（如状态码、邮箱、标签） |
| 查询方式 | match 查询 | term 查询 |
| 是否支持聚合 | ❌ 默认不支持（需开启 fielddata，不推荐） | ✅ 原生支持 |
| 是否支持排序 | ❌ 默认不支持 | ✅ 原生支持 |

**实际使用**：ES 动态映射对字符串默认创建 `text` 类型 + `keyword` 子字段（`field.keyword`），兼顾全文检索和精确匹配。
3. 分词器的作用？Standard、IK、自定义分词器？

**回答：**

**分词器（Analyzer）** 的作用是将文本拆分为一个个词项（Term），用于建立倒排索引和执行全文检索。

分词器由三部分组成：
1. **Character Filter（字符过滤器）**：预处理原始文本（如去除 HTML 标签、字符替换）
2. **Tokenizer（分词器）**：将文本切分为词项
3. **Token Filter（词项过滤器）**：对词项进行加工（如转小写、去停用词、同义词扩展）

**常用分词器：**

- **Standard Analyzer**：ES 默认分词器，按 Unicode 文本分割算法分词，转小写，去停用词。对英文效果好，对中文会逐字拆分（不实用）。
- **IK Analyzer**：中文分词器（第三方插件），支持两种模式：
  - `ik_smart`：粗粒度分词，尽量少切分。如"中华人民共和国" → "中华人民共和国"
  - `ik_max_word`：细粒度分词，尽量多切分。如"中华人民共和国" → "中华人民共和国"、"中华人民"、"中华"、"人民共和国"、"人民"、"共和国"...
  - 支持自定义词典，可以添加业务专有名词。
- **自定义分词器**：通过组合 Character Filter + Tokenizer + Token Filter 来定义，例如：
  ```json
  {
    "analyzer": {
      "my_analyzer": {
        "type": "custom",
        "tokenizer": "ik_max_word",
        "filter": ["lowercase", "my_synonym_filter"]
      }
    }
  }
  ```

**索引时和搜索时可以使用不同的分词器**：索引时用 `ik_max_word`（多切分，提高召回率），搜索时用 `ik_smart`（少切分，提高精确度）。
4. 索引模板（Index Template）的使用？

**回答：**

**索引模板** 是预定义的索引配置，当创建新索引时，如果索引名匹配模板的模式，ES 会自动应用模板中的 Settings 和 Mappings。

**使用场景：**
- 日志类索引按日期滚动创建（如 `log-2024-01-01`、`log-2024-01-02`），每个索引需要相同的配置。
- 统一管理多个索引的分片数、副本数、映射规则。

**示例：**
```json
PUT _index_template/log_template
{
  "index_patterns": ["log-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "message": { "type": "text", "analyzer": "ik_max_word" },
        "level": { "type": "keyword" }
      }
    }
  }
}
```

**注意事项：**
- ES 7.8+ 推荐使用**可组合索引模板**（Composable Index Template），支持组件模板（Component Template）复用。
- 多个模板匹配同一索引时，通过 `priority` 决定优先级。
- 模板只在索引创建时生效，修改模板不会影响已有索引。
5. 索引别名（Alias）的作用？

**回答：**

**索引别名** 是指向一个或多个索引的虚拟名称，客户端通过别名访问索引，实现索引的透明切换。

**核心作用：**

1. **零停机重建索引（Reindex）**：
   - 应用程序始终访问别名 `my_index`，实际指向 `my_index_v1`。
   - 需要修改 Mapping 时，创建新索引 `my_index_v2`，迁移数据后，将别名切换到 `my_index_v2`。
   - 应用程序无感知，无需修改代码或重启。

2. **多索引聚合查询**：
   - 一个别名可以关联多个索引，查询别名等于同时查询所有关联索引。
   - 适用于按时间分割的日志索引：别名 `logs_current` 指向最近 7 天的索引。

3. **过滤别名（Filtered Alias）**：
   - 别名可以附带 filter 条件，实现数据视图的效果。
   - 例如：别名 `active_users` 只返回 `status=active` 的文档。

**操作示例：**
```json
POST _aliases
{
  "actions": [
    { "remove": { "index": "my_index_v1", "alias": "my_index" } },
    { "add": { "index": "my_index_v2", "alias": "my_index" } }
  ]
}
```

这个操作是**原子性**的，切换过程中不会出现别名不可用的情况。

## 三、查询与聚合

1. Query DSL 的基本结构？

**回答：**

Query DSL（Domain Specific Language）是 ES 提供的基于 JSON 的查询语言，基本结构如下：

```json
GET /index/_search
{
  "query": {          // 查询条件
    "bool": {
      "must": [...],
      "filter": [...]
    }
  },
  "from": 0,          // 分页起始位置
  "size": 10,         // 返回文档数量
  "sort": [...],      // 排序规则
  "_source": [...],   // 指定返回的字段
  "aggs": {...},      // 聚合查询
  "highlight": {...}  // 高亮设置
}
```

**两大类查询上下文：**
- **Query Context（查询上下文）**：计算相关性评分（`_score`），回答"这个文档有多匹配"。如 `match`、`bool` 中的 `must`/`should`。
- **Filter Context（过滤上下文）**：不计算评分，只判断是否匹配（yes/no），且结果可以被缓存。如 `term`、`range`、`bool` 中的 `filter`。

**最佳实践**：不需要评分的条件放在 `filter` 中，既能提高性能（跳过评分计算），又能利用缓存。
2. match、term、range、bool 查询的区别？

**回答：**

- **match 查询**：全文检索查询。会对查询文本进行分词，然后在倒排索引中匹配。适用于 `text` 类型字段。
  ```json
  { "match": { "content": "Java 面试" } }
  // "Java 面试" 会被分词为 "java" 和 "面试"，匹配包含任一词项的文档
  ```

- **term 查询**：精确匹配查询。不对查询文本分词，直接在倒排索引中查找完全匹配的词项。适用于 `keyword`、数值、日期等类型。
  ```json
  { "term": { "status": "published" } }
  // 注意：对 text 字段使用 term 查询通常不会得到预期结果，因为 text 字段的值已被分词存储
  ```

- **range 查询**：范围查询。支持 `gt`（大于）、`gte`（大于等于）、`lt`（小于）、`lte`（小于等于）。
  ```json
  { "range": { "price": { "gte": 100, "lte": 500 } } }
  ```

- **bool 查询**：组合查询，通过 `must`、`should`、`must_not`、`filter` 组合多个查询条件。
  ```json
  {
    "bool": {
      "must": [{ "match": { "title": "Java" } }],
      "filter": [{ "range": { "price": { "lte": 100 } } }],
      "must_not": [{ "term": { "status": "deleted" } }]
    }
  }
  ```
3. must、should、must_not、filter 的区别？filter 为什么更快？

**回答：**

| 子句 | 是否影响评分 | 逻辑含义 | 说明 |
|------|------------|---------|------|
| `must` | ✅ 参与评分 | AND | 文档必须匹配，贡献相关性评分 |
| `should` | ✅ 参与评分 | OR | 文档应该匹配（在没有 must 时至少匹配一个），匹配的越多分数越高 |
| `must_not` | ❌ 不参与评分 | NOT | 文档必须不匹配，在 Filter Context 中执行 |
| `filter` | ❌ 不参与评分 | AND | 文档必须匹配，但不计算评分 |

**filter 为什么更快？**

1. **跳过评分计算**：filter 不需要计算 `_score`，省去了 TF-IDF / BM25 评分的计算开销。
2. **结果可缓存**：filter 的结果会被 ES 缓存到 **Node Query Cache**（基于 Bitset），相同的 filter 条件再次查询时直接命中缓存，速度极快。
3. **利用 Bitset 运算**：多个 filter 条件可以通过位运算（AND、OR）快速合并结果集。

**最佳实践**：
- 需要评分排序的条件放 `must`/`should`（如全文检索关键词）
- 不需要评分的过滤条件放 `filter`（如状态过滤、时间范围、分类筛选）
4. 全文检索和精确查询的区别？

**回答：**

| 对比项 | 全文检索 | 精确查询 |
|--------|---------|---------|
| 是否分词 | ✅ 查询文本会被分词 | ❌ 查询文本不分词 |
| 适用字段类型 | `text` | `keyword`、数值、日期、布尔 |
| 典型查询 | `match`、`multi_match`、`match_phrase` | `term`、`terms`、`range` |
| 评分 | 计算相关性评分 | 通常放在 filter 中，不计算评分 |
| 使用场景 | 搜索文章内容、商品描述 | 过滤状态、匹配ID、范围筛选 |

**常见坑**：
- 对 `text` 字段使用 `term` 查询：由于 text 字段存储的是分词后的词项（如 "Hello World" 存储为 "hello" 和 "world"），用 `term` 查询 "Hello World" 不会匹配。应该用 `match` 查询，或者对 `keyword` 子字段使用 `term`（如 `field.keyword`）。
- 对 `keyword` 字段使用 `match` 查询：虽然能工作（因为 keyword 不分词，match 分词后也是原值），但语义上不正确，应该用 `term`。
5. 聚合查询？Bucket 聚合、Metric 聚合、Pipeline 聚合？

**回答：**

ES 的聚合（Aggregation）类似于 SQL 的 `GROUP BY` + 聚合函数，用于数据分析和统计。

**三大类聚合：**

1. **Bucket 聚合（桶聚合）**：将文档分组到不同的桶中，类似 `GROUP BY`。
   - `terms`：按字段值分组（如按品牌分组统计商品数量）
   - `date_histogram`：按时间间隔分组（如按天/月统计订单量）
   - `range`：按数值范围分组（如价格区间）
   - `filter` / `filters`：按过滤条件分组
   ```json
   { "aggs": { "by_brand": { "terms": { "field": "brand" } } } }
   ```

2. **Metric 聚合（指标聚合）**：对文档集合计算统计指标。
   - `avg`、`sum`、`min`、`max`：平均值、求和、最小值、最大值
   - `cardinality`：去重计数（类似 `COUNT(DISTINCT)`，基于 HyperLogLog 算法，有微小误差）
   - `stats`：一次返回 count、min、max、avg、sum
   - `percentiles`：百分位数统计
   ```json
   { "aggs": { "avg_price": { "avg": { "field": "price" } } } }
   ```

3. **Pipeline 聚合（管道聚合）**：基于其他聚合的结果进行二次计算。
   - `derivative`：求导（如计算环比增长）
   - `moving_avg`：移动平均
   - `bucket_sort`：对桶排序
   - `bucket_selector`：过滤桶

**聚合可以嵌套**：在 Bucket 聚合内嵌套 Metric 聚合，如"按品牌分组，计算每个品牌的平均价格"。
6. 高亮搜索的实现？

**回答：**

高亮搜索是在搜索结果中对匹配的关键词添加标签（如 `<em>`），方便前端展示。

**基本用法：**
```json
GET /index/_search
{
  "query": { "match": { "content": "Java" } },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "content": {}
    }
  }
}
```

**三种高亮器：**
1. **unified（默认）**：ES 5+ 默认高亮器，基于 BM25 算法，支持 `text` 和 `keyword` 字段，性能和效果均衡。
2. **plain**：基于标准 Lucene 高亮器，逐字段分析，适合小文本字段，大文本性能较差。
3. **fvh（Fast Vector Highlighter）**：需要字段开启 `term_vector=with_positions_offsets`，占用更多存储空间，但对大文本字段性能最好。

**注意事项：**
- 高亮结果在返回的 `highlight` 字段中，不在 `_source` 中，前端需要用 `highlight` 中的内容替换 `_source` 中的对应字段。
- 可以通过 `fragment_size` 控制高亮片段长度，`number_of_fragments` 控制返回片段数量。
7. 搜索建议（Suggest）的实现？

**回答：**

ES 提供了 Suggest API 用于实现搜索建议（自动补全、拼写纠错等）。

**四种 Suggester：**

1. **Term Suggester**：基于编辑距离（Levenshtein Distance）的拼写纠错。对单个词项提供纠错建议。
   ```json
   { "suggest": { "my_suggest": { "text": "jvaa", "term": { "field": "title" } } } }
   // 可能建议 "java"
   ```

2. **Phrase Suggester**：在 Term Suggester 基础上，考虑短语的整体合理性，提供更准确的短语级纠错。

3. **Completion Suggester**：自动补全，性能最好。基于 FST（有限状态转换器）数据结构，存储在内存中，查询速度极快。
   - 需要使用 `completion` 类型字段。
   - 支持前缀匹配，适合搜索框的实时补全。
   ```json
   // Mapping
   { "suggest_field": { "type": "completion" } }
   // 查询
   { "suggest": { "my_suggest": { "prefix": "jav", "completion": { "field": "suggest_field" } } } }
   ```

4. **Context Suggester**：在 Completion Suggester 基础上增加上下文过滤（如按分类、地理位置过滤补全结果）。

**实际应用**：搜索框自动补全用 Completion Suggester（性能最好），搜索纠错用 Phrase Suggester。

## 四、性能优化

1. 深度分页问题？from+size vs scroll vs search_after？

**回答：**

**深度分页问题**：当 `from` 值很大时（如 `from=10000, size=10`），ES 需要在每个分片上查询 `from + size = 10010` 条数据，协调节点汇总后排序取 top 10。如果有 5 个分片，就需要汇总 50050 条数据，内存和性能开销巨大。ES 默认限制 `from + size ≤ 10000`。

**三种分页方案对比：**

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| `from + size` | 跳过前 from 条，取 size 条 | 支持随机跳页 | 深度分页性能差，有 10000 条限制 | 浅分页（前几页） |
| `scroll` | 创建快照，通过 scroll_id 遍历 | 适合大量数据导出 | 占用服务端资源（维护搜索上下文），不适合实时查询，数据是快照不反映最新变化 | 数据导出、全量遍历 |
| `search_after` | 基于上一页最后一条的排序值作为游标 | 无深度分页问题，实时数据 | 只能向后翻页，不能跳页 | 实时深度分页（无限滚动） |

**search_after 示例：**
```json
// 第一页
GET /index/_search
{ "size": 10, "sort": [{ "timestamp": "desc" }, { "_id": "asc" }] }

// 下一页（使用上一页最后一条的排序值）
GET /index/_search
{ "size": 10, "sort": [{ "timestamp": "desc" }, { "_id": "asc" }], "search_after": [1640000000000, "doc_id_xxx"] }
```

**最佳实践**：前端分页用 `from + size`（限制最大页数），无限滚动用 `search_after`，数据导出用 `scroll` 或 `search_after` + PIT（Point In Time）。
2. 索引设计的最佳实践？分片数量如何确定？

**回答：**

**索引设计最佳实践：**

1. **合理设置分片数**：
   - 单个分片建议 10GB ~ 50GB（官方推荐不超过 50GB）。
   - 分片数 = 预估数据量 / 单分片大小。
   - 分片数不宜过多：每个分片是一个 Lucene 索引，占用文件句柄、内存和 CPU，过多分片会增加集群负担。
   - 分片数不宜过少：无法充分利用多节点并行能力。
   - 经验法则：每个节点的分片数不超过 20 个/GB 堆内存。

2. **使用显式 Mapping**：避免动态映射导致类型不合理。

3. **合理使用字段类型**：
   - 不需要检索的字段设置 `index: false`
   - 不需要返回原始值的字段设置 `_source.excludes` 或 `enabled: false`
   - 不需要聚合/排序的 text 字段不要开启 `fielddata`

4. **按时间滚动索引**：日志类数据按天/周/月创建索引，配合 ILM（Index Lifecycle Management）自动管理。

5. **使用索引别名**：方便索引切换和重建。

6. **副本数设置**：
   - 写入密集型：先设为 0，写入完成后再增加副本。
   - 读取密集型：增加副本数提高查询吞吐。

**分片数量确定公式**：
```
分片数 = ceil(预估数据量 / 单分片目标大小)
```
例如：预估 500GB 数据，单分片 30GB，则分片数 = ceil(500/30) = 17 个主分片。
3. 写入性能优化？（批量写入、调整 refresh_interval、关闭副本）

**回答：**

**写入性能优化策略：**

1. **使用 Bulk API 批量写入**：
   - 单次请求写入多条文档，减少网络往返开销。
   - 建议每批 1000~5000 条，或每批 5~15MB，根据实际测试调整。
   ```json
   POST _bulk
   { "index": { "_index": "my_index" } }
   { "field1": "value1" }
   { "index": { "_index": "my_index" } }
   { "field2": "value2" }
   ```

2. **增大 refresh_interval**：
   - 默认 1 秒 Refresh 一次，大批量写入时设为 `30s` 或 `-1`（禁用自动 Refresh）。
   - 减少 Segment 生成频率，降低 I/O 和 Merge 开销。
   - 写入完成后恢复为默认值。

3. **写入期间关闭副本**：
   - 设置 `number_of_replicas: 0`，写入完成后再恢复。
   - 避免主分片写入时同步复制到副本的开销。

4. **调整 Translog 策略**：
   - 将 `index.translog.durability` 从 `request`（每次请求 fsync）改为 `async`（异步 fsync），降低磁盘 I/O。
   - 代价是宕机时可能丢失少量数据。

5. **增大 Index Buffer**：
   - `indices.memory.index_buffer_size` 默认为 JVM 堆的 10%，大量写入时可适当增大。

6. **使用自动生成 ID**：
   - 不指定 `_id`，让 ES 自动生成，避免写入前检查 ID 是否存在的开销。

7. **合理的 Mapping 设计**：
   - 不需要索引的字段设置 `index: false`，减少倒排索引构建开销。
4. 查询性能优化？（路由、预热、缓存）

**回答：**

**查询性能优化策略：**

1. **自定义路由（Routing）**：
   - 默认路由：`hash(_id) % 分片数`，查询时需要广播到所有分片。
   - 自定义路由：写入时指定 `routing=user_id`，相同用户的数据落在同一分片。查询时指定路由，只查一个分片，避免广播。
   - 适用于按租户、用户等维度查询的场景。

2. **利用 Filter Cache**：
   - 不需要评分的条件放在 `filter` 中，ES 会自动缓存 filter 结果（Node Query Cache）。
   - 频繁使用的 filter 条件命中缓存后查询极快。

3. **预热（Warmer）**：
   - 新 Segment 生成后，第一次查询可能较慢（需要加载到 OS Cache）。
   - 可以通过 `_forcemerge` 减少 Segment 数量，或确保有足够的 OS Cache 容纳索引数据。
   - 生产环境建议预留至少 50% 的物理内存给 OS Cache。

4. **合理使用 _source**：
   - 只返回需要的字段：`"_source": ["title", "price"]`，减少网络传输。

5. **避免深度分页**：使用 `search_after` 替代大 `from` 值。

6. **使用 keyword 字段做精确查询**：避免对 text 字段做排序和聚合。

7. **合理设计索引粒度**：
   - 按时间分索引，查询时通过索引名或别名限定范围，避免全量扫描。

8. **Profile API 分析慢查询**：
   - 使用 `"profile": true` 分析查询各阶段耗时，定位瓶颈。

9. **Segment Merge 优化**：
   - 定期执行 `_forcemerge`（在低峰期），将小 Segment 合并为大 Segment，减少查询时需要搜索的 Segment 数量。
5. 冷热架构是什么？如何实现？

**回答：**

**冷热架构（Hot-Warm-Cold Architecture）** 是根据数据的访问频率，将数据存储在不同性能的节点上，实现成本和性能的平衡。

**三层架构：**
- **Hot 节点**：高性能 SSD 磁盘，存储最近的、频繁访问的数据（如最近 7 天的日志）。负责写入和高频查询。
- **Warm 节点**：普通 SSD 或 HDD，存储较旧的、偶尔访问的数据（如 7~30 天的日志）。只读，不再写入。
- **Cold 节点**：大容量 HDD，存储历史归档数据（如 30 天以上的日志）。极少访问，主要用于合规保留。

**实现方式：**

1. **节点标记**：通过 `node.attr.data_type` 标记节点角色。
   ```yaml
   # elasticsearch.yml
   node.attr.data_type: hot   # 或 warm、cold
   ```

2. **索引分配策略**：创建索引时指定分配到 Hot 节点。
   ```json
   PUT /log-2024-01-15
   { "settings": { "index.routing.allocation.require.data_type": "hot" } }
   ```

3. **ILM（Index Lifecycle Management）自动管理**：
   - 定义生命周期策略，自动将索引从 Hot → Warm → Cold → Delete。
   ```json
   PUT _ilm/policy/log_policy
   {
     "policy": {
       "phases": {
         "hot": { "actions": { "rollover": { "max_age": "7d", "max_size": "50gb" } } },
         "warm": { "min_age": "7d", "actions": { "allocate": { "require": { "data_type": "warm" } }, "forcemerge": { "max_num_segments": 1 } } },
         "cold": { "min_age": "30d", "actions": { "allocate": { "require": { "data_type": "cold" } } } },
         "delete": { "min_age": "90d", "actions": { "delete": {} } }
       }
     }
   }
   ```

**优势**：Hot 节点用高性能硬件保证实时性能，Warm/Cold 节点用低成本硬件存储历史数据，整体成本大幅降低。

## 五、数据同步

1. ES 和 MySQL 的数据同步方案？

**回答：**

ES 通常作为 MySQL 的查询加速层，需要将 MySQL 数据同步到 ES。常见方案：

1. **同步双写**：应用层同时写 MySQL 和 ES。
2. **异步双写（MQ）**：写 MySQL 后发消息到 MQ，消费者写 ES。
3. **Canal + MQ**：通过 Canal 监听 MySQL Binlog，将变更通过 MQ 同步到 ES。
4. **Logstash JDBC Input**：Logstash 定时轮询 MySQL，增量同步到 ES。
5. **Flink CDC**：通过 Flink CDC 连接器实时消费 Binlog，写入 ES。

| 方案 | 实时性 | 侵入性 | 一致性 | 复杂度 |
|------|--------|--------|--------|--------|
| 同步双写 | 高 | 高（代码侵入） | 弱（可能部分失败） | 低 |
| 异步双写（MQ） | 较高 | 中 | 最终一致 | 中 |
| Canal + MQ | 较高（秒级） | 低（无代码侵入） | 最终一致 | 中高 |
| Logstash | 低（分钟级） | 无 | 最终一致 | 低 |
| Flink CDC | 高 | 低 | 最终一致 | 高 |

**推荐方案**：生产环境推荐 **Canal + MQ** 方案，对业务代码无侵入，实时性好，且通过 MQ 解耦可以保证可靠性。
2. 同步双写的问题？

**回答：**

同步双写是指在应用代码中，写完 MySQL 后紧接着写 ES。虽然实现简单，但存在以下问题：

1. **数据不一致**：
   - MySQL 写入成功但 ES 写入失败（网络抖动、ES 不可用），导致两边数据不一致。
   - 即使加了重试，也可能出现顺序问题（如先更新后删除，重试时顺序颠倒）。

2. **代码侵入性高**：
   - 每个涉及数据变更的地方都要加 ES 写入逻辑，维护成本高。
   - 容易遗漏某些写入点，导致数据不同步。

3. **性能影响**：
   - ES 写入是同步的，增加了接口响应时间。
   - 如果 ES 响应慢或不可用，会拖慢甚至阻塞主业务流程。

4. **事务问题**：
   - MySQL 和 ES 无法在同一个事务中，无法保证原子性。
   - MySQL 事务回滚了，但 ES 已经写入，数据不一致。

5. **耦合度高**：
   - 业务代码和 ES 强耦合，ES 的变更（如索引结构调整）需要修改业务代码。

**改进方向**：使用异步方案（MQ 或 Canal）解耦，保证最终一致性。
3. Canal + MQ 的异步同步方案？

**回答：**

**架构流程**：MySQL → Canal → MQ（Kafka/RocketMQ）→ Consumer → ES

**各组件职责：**

1. **Canal**：
   - 伪装为 MySQL 的从库，订阅 MySQL 的 Binlog（Binary Log）。
   - 解析 Binlog 中的 INSERT、UPDATE、DELETE 事件，转换为结构化的数据变更消息。
   - 将变更消息发送到 MQ。

2. **MQ（消息队列）**：
   - 解耦 Canal 和消费者，提供缓冲和削峰能力。
   - 保证消息的可靠投递（持久化、重试）。
   - 保证同一主键的消息有序（通过 Partition Key = 主键ID）。

3. **Consumer（消费者）**：
   - 消费 MQ 中的变更消息，根据操作类型执行对应的 ES 操作。
   - INSERT/UPDATE → ES index 操作
   - DELETE → ES delete 操作

**优点：**
- 对业务代码零侵入，不需要修改任何业务逻辑。
- 通过 MQ 保证可靠性，消费失败可以重试。
- 解耦彻底，ES 不可用时消息在 MQ 中堆积，恢复后自动追赶。

**注意事项：**
- 需要 MySQL 开启 Binlog（`binlog_format=ROW`）。
- 需要保证消息的顺序性，避免先 UPDATE 后 DELETE 的消息被乱序消费。
- 消费者需要做幂等处理，防止重复消费导致数据异常。
- 存在秒级延迟，不适合对实时性要求极高的场景。
4. Logstash 同步方案？

**回答：**

Logstash 通过 **JDBC Input Plugin** 定时轮询 MySQL，将数据同步到 ES。

**基本配置：**
```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "root"
    jdbc_password => "password"
    schedule => "*/5 * * * *"          # 每5分钟执行一次
    statement => "SELECT * FROM products WHERE update_time > :sql_last_value"
    use_column_value => true
    tracking_column => "update_time"    # 增量同步的标记列
    tracking_column_type => "timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "products"
    document_id => "%{id}"              # 用MySQL主键作为ES文档ID
  }
}
```

**工作原理：**
- 通过 `sql_last_value` 记录上次同步的位置（时间戳或自增ID）。
- 每次执行只查询增量数据，写入 ES。

**优点：**
- 配置简单，无需编写代码。
- 适合数据量不大、实时性要求不高的场景。

**缺点：**
- **无法感知删除操作**：Logstash 只能查询到存在的数据，MySQL 中删除的记录无法同步删除 ES 中的文档（需要软删除 + 标记位）。
- **实时性差**：依赖轮询间隔，通常分钟级延迟。
- **对 MySQL 有查询压力**：频繁轮询大表可能影响 MySQL 性能。
- **不支持复杂变更**：如多表关联变更难以处理。
5. 数据一致性如何保证？

**回答：**

MySQL 和 ES 之间是**最终一致性**，无法做到强一致。保证数据一致性的策略：

1. **消息可靠投递**：
   - MQ 开启消息持久化，确保消息不丢失。
   - 生产者使用同步发送 + 确认机制。
   - 消费者手动 ACK，处理成功后再确认。

2. **消费幂等性**：
   - 使用 MySQL 主键作为 ES 文档 `_id`，重复写入会覆盖（天然幂等）。
   - 或者通过版本号/时间戳判断，只处理更新的数据。

3. **消息顺序性**：
   - 同一条数据的变更消息发送到同一个 MQ Partition（通过主键 Hash）。
   - 保证同一条数据的 INSERT → UPDATE → DELETE 按顺序消费。

4. **补偿机制**：
   - 定时全量/增量对账：定期比对 MySQL 和 ES 的数据，修复不一致。
   - 可以通过对比 count、抽样校验 checksum 等方式发现差异。

5. **失败重试**：
   - 消费失败的消息进入重试队列，多次重试仍失败则进入死信队列，人工介入处理。

6. **监控告警**：
   - 监控 MQ 消费延迟（Consumer Lag），延迟过大时告警。
   - 监控 ES 写入失败率。

**核心思路**：接受最终一致性，通过可靠消息 + 幂等消费 + 对账补偿来保证数据最终一致。

## 六、实战场景

1. ES 在你的 AI 检索项目中是怎么用的？

**回答：**

在 AI 智能识别与检索项目中，ES（8.x+）作为核心检索引擎，实现了**混合检索（Hybrid Search）**架构，融合 BM25 文本检索和 kNN 向量检索。

**索引设计：**
- `entities_text`（text 类型）：将图片中识别出的人、车、物等实体属性拍平为一段英文文本，使用自定义英文分词器（含词根还原 Stemmer + 安防领域同义词字典），用于 BM25 精确词频匹配。
- `caption_vector`（dense_vector 类型，768 维）：将 AI 生成的场景描述（scene_caption）通过 Embedding 模型转为向量，用于语义相似度检索。
- `camera_id`（keyword）、`timestamp`（date）：用于前置硬过滤。

**混合检索流程：**
1. **Pre-filtering（前置过滤）**：通过 filter 子句按摄像头 ID 和时间范围截断无关数据，大幅缩小搜索范围。
2. **BM25 词频检索**：用分词后的搜索关键词匹配 `entities_text`，保证颜色、物种等属性的精确命中（如搜"红色"必须真实存在"Red"）。
3. **kNN 向量检索**：用搜索意图的向量匹配 `caption_vector`，捕捉复杂行为语义（如"拿着箱子跑入车库"的动作关系）。
4. **RRF 融合打分**：BM25 权重 0.6 + 向量权重 0.4，安防场景下文本匹配略占上风（用户对颜色和物种极其敏感）。

**重排机制：**
- 召回 Top 100 后，使用 Cross-Encoder Reranker 模型进行二阶段重排，解决"属性错位幻觉"问题（如搜"黄衣黑车"不会返回"黑衣黄车"）。
- 设置硬性相似度阈值（0.85），宁缺毋滥，不给用户推荐无关结果。
2. 向量检索（kNN）在 ES 中的支持？

**回答：**

ES 从 8.0 开始原生支持 kNN（k-Nearest Neighbors）向量检索。

**字段类型：**
- `dense_vector`：稠密向量字段，用于存储 Embedding 向量。
  ```json
  {
    "caption_vector": {
      "type": "dense_vector",
      "dims": 768,
      "index": true,
      "similarity": "cosine"    // 支持 cosine、dot_product、l2_norm
    }
  }
  ```

**两种检索方式：**

1. **Exact kNN（精确 kNN）**：通过 `script_score` 查询，遍历所有文档计算相似度，精确但慢。适合小数据量。

2. **Approximate kNN（近似 kNN）**：通过 `knn` 查询，使用 HNSW（Hierarchical Navigable Small World）算法，在精度和速度之间取平衡。适合大规模数据。
   ```json
   GET /index/_search
   {
     "knn": {
       "field": "caption_vector",
       "query_vector": [0.1, 0.2, ...],
       "k": 10,
       "num_candidates": 100
     }
   }
   ```

**混合检索（Hybrid Search）**：
- ES 8.x 支持在同一查询中组合 kNN 和传统 BM25 查询。
- 通过 RRF（Reciprocal Rank Fusion）或线性加权融合两种检索的分数。
- 在我们的项目中，BM25 保证属性精确匹配，kNN 捕捉语义相似度，两者互补。

**HNSW 参数调优：**
- `m`：每个节点的连接数，越大精度越高但索引越慢，默认 16。
- `ef_construction`：构建索引时的候选集大小，越大精度越高，默认 100。
- `num_candidates`：查询时的候选集大小，越大精度越高但查询越慢。
3. ES 集群的运维经验？节点角色划分？

**回答：**

**节点角色划分：**

| 角色 | 职责 | 配置建议 |
|------|------|---------|
| Master Node | 管理集群状态、索引创建/删除、分片分配 | 至少 3 个（奇数，防脑裂），低配即可（不需要大磁盘） |
| Data Node | 存储数据，执行搜索和聚合 | 高配（大内存、大磁盘、SSD） |
| Coordinating Node | 接收请求，转发和汇总结果 | 中等配置，主要消耗 CPU 和内存 |
| Ingest Node | 数据预处理（Pipeline） | 按需配置 |
| ML Node | 机器学习任务 | 按需配置 |

**生产环境建议：**
- Master 和 Data 角色分离，避免 Data Node 负载过高影响集群管理。
- 设置 `cluster.initial_master_nodes` 和 `discovery.seed_hosts` 防止脑裂。

**运维要点：**

1. **JVM 堆内存**：设为物理内存的 50%，但不超过 32GB（超过 32GB 会失去指针压缩优势）。剩余内存留给 OS Cache。

2. **监控指标**：
   - 集群健康状态（Green/Yellow/Red）
   - 节点 CPU、内存、磁盘使用率
   - JVM GC 频率和耗时
   - 搜索和索引延迟（Search Latency、Indexing Latency）
   - 未分配分片数

3. **常见问题处理**：
   - Yellow 状态：通常是副本分片未分配（如单节点集群），增加节点或调整副本数。
   - Red 状态：主分片丢失，需要紧急排查节点状态。
   - 磁盘水位线：ES 在磁盘使用超过 85% 时停止分配分片，超过 95% 时设为只读。

4. **滚动重启**：升级或配置变更时，先禁用分片分配（`cluster.routing.allocation.enable: none`），重启节点后再恢复，避免不必要的分片迁移。
4. ES 的安全机制？认证和授权？

**回答：**

ES 从 8.0 开始默认启用安全功能（之前需要 X-Pack 付费插件）。

**认证（Authentication）**：
1. **内置用户认证**：ES 内置 `elastic`（超级管理员）、`kibana_system` 等用户，通过用户名密码认证。
2. **Native Realm**：ES 内部存储用户信息，通过 API 管理用户。
3. **LDAP/Active Directory**：集成企业目录服务。
4. **SAML/OpenID Connect**：支持单点登录（SSO）。
5. **API Key**：为应用程序生成 API Key，适合服务间调用。
6. **PKI 证书认证**：基于 TLS 客户端证书认证。

**授权（Authorization）**：
1. **基于角色的访问控制（RBAC）**：
   - 定义角色（Role），指定可以访问的索引、字段和操作。
   - 将角色分配给用户。
   ```json
   POST _security/role/read_only_role
   {
     "indices": [
       {
         "names": ["log-*"],
         "privileges": ["read"],
         "field_security": { "grant": ["timestamp", "message"] }
       }
     ]
   }
   ```

2. **字段级安全（Field Level Security）**：控制用户能看到哪些字段。
3. **文档级安全（Document Level Security）**：通过查询条件限制用户能看到哪些文档。

**传输加密：**
- 节点间通信：TLS 加密（Transport Layer）。
- 客户端通信：HTTPS（HTTP Layer）。
- ES 8.0+ 首次启动时自动生成 CA 证书和节点证书。

**审计日志**：记录所有认证和授权事件，用于安全审计和合规。
