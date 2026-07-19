# SQL 数据库区别

## 1. ClickHouse、Doris、MySQL 有什么区别？查询如何解析和执行？

### 问题

ClickHouse、Doris、MySQL 三者之间有什么区别？一条 SQL 查询进入数据库后，分别会如何解析、优化和执行？

### 核心结论

- **MySQL**：面向 OLTP，擅长精确找到少量数据，并在事务中可靠地增删改。
- **ClickHouse**：面向实时 OLAP，擅长从海量列式数据中少读数据、多线程扫描和聚合。
- **Doris**：面向实时数仓和复杂分析，通过 FE 统一规划，将一条 SQL 拆成多个 Fragment，在多个 BE 上并行执行。

### 核心区别

| 维度 | MySQL | ClickHouse | Doris |
|---|---|---|---|
| 定位 | OLTP 事务数据库 | 实时 OLAP、日志和指标分析 | 实时数仓、MPP 分析数据库 |
| 存储 | 行式，聚簇 B+Tree | 列式，Part + Granule | 列式，Tablet + Rowset + Segment |
| 典型查询 | 主键点查、短范围查询 | 大规模扫描、过滤、聚合 | 多表 Join、BI、复杂报表 |
| 主键 | 唯一，通常是聚簇索引 | 稀疏索引，不保证唯一 | 由 Duplicate、Unique、Aggregate 模型决定 |
| 优化器 | 单机 CBO | Analyzer + 查询计划优化 | Nereids：Cascades RBO + CBO |
| 执行模型 | 逐行迭代器为主 | 列式 Block + Pipeline | MPP + 向量化 Block + Pipeline |
| 分布式 | 复制分散不同请求，不拆分单条 SQL | Distributed 表将查询发送到各分片 | FE 统一规划，多个 BE 执行同一条 SQL |
| 更新 | 原生行级增删改 | 追加 Part、后台 Merge，随机更新成本较高 | Unique Key 支持 UPSERT，物理上仍偏追加合并 |
| 事务 | 完整多语句、多表 ACID | 单次 INSERT 较强，通用事务受限 | 支持 DML 事务，但限制多于 MySQL |

### SQL 的通用执行流程

```text
SQL 文本
→ 词法、语法解析
→ AST 或逻辑树
→ 语义分析：绑定表、列、函数和类型
→ 逻辑改写：谓词下推、列裁剪、子查询改写
→ 优化器选择物理计划
→ 扫描存储并执行 Join、聚合、排序
→ 返回结果
```

三者的 Parser 都能把 SQL 转换为语法树。真正造成性能差异的主要是：存储布局、索引和数据裁剪方式、执行模型以及是否能进行分布式并行。

### MySQL 查询过程

```text
客户端
→ Server 层：Parser → Resolver → Optimizer → Executor
→ handler 存储引擎接口
→ InnoDB
```

1. Parser 将 SQL 转换为内部查询树。
2. Resolver 绑定表、列和别名，展开 `SELECT *`，检查权限和类型，并处理视图、派生表及子查询。
3. CBO 根据表统计信息、索引基数和直方图选择访问路径、索引、Join 顺序及 Join 算法。
4. Executor 通过迭代器逐行拉取数据，执行过滤、Join、聚合、排序和 `LIMIT`。
5. 底层通过 `handler` 调用 InnoDB，使用聚簇 B+Tree、Buffer Pool 和 MVCC 读取数据。

InnoDB 的主键索引叶子节点保存完整行；二级索引保存“二级索引列 + 主键”。如果查询需要索引之外的字段，需要根据主键再次查询聚簇索引，即“回表”。

标准 MySQL 复制主要将不同读请求分配给 Replica，不会自动把一条聚合 SQL 拆到多个 Replica 执行。

### ClickHouse 查询过程

```text
SQL
→ Parser / AST
→ Analyzer / Query Tree
→ Planner / Query Plan
→ Query Pipeline
→ Executor
```

1. Parser 把 SQL 转换为 AST。
2. Analyzer 解析表、列、别名、函数和类型，通过 Pass 进行表达式改写与优化。
3. Planner 生成 `ReadFromMergeTree → Filter → Aggregating → Sorting → Limit` 等查询计划。
4. MergeTree 依次进行分区、稀疏主键、跳数索引和 Projection 裁剪。
5. 只读取查询涉及的列和无法排除的 Granule，解压后以列式 Block 处理。
6. 多个线程先做局部聚合，再合并聚合状态。

MergeTree 的主键索引通常每个 Granule 保存一个 Mark，默认 Granule 大约为 8192 行，因此它用于跳过数据块，而不是像 MySQL B+Tree 那样精确指向每一行；ClickHouse 主键也不要求唯一。

分布式查询时，发起节点将查询发送到各 Shard，各 Shard 尽量完成过滤和局部聚合，最后由发起节点合并结果。

### Doris 查询过程

```text
客户端 MySQL 协议
→ FE：Parser → Logical Plan → Nereids RBO/CBO
→ Physical Plan → PlanFragment DAG
→ Coordinator 下发
→ 多个 BE 使用 Pipeline 执行
→ Root Fragment → FE → 客户端
```

1. FE 负责 SQL 解析、语义分析、权限检查和查询优化。
2. Nereids 的 RBO 完成列裁剪、谓词下推、分区裁剪及子查询改写。
3. Cascades CBO 根据统计信息选择 Join 顺序，以及 Broadcast、Shuffle、Bucket Shuffle 或 Colocate Join。
4. 物理计划在数据重分布边界被拆成多个 `PlanFragment`。
5. Coordinator 将 Tablet、Fragment Instance 分配给对应 BE。
6. BE 将 Fragment 拆成多个 PipelineTask，以列式 Block 执行扫描、过滤、Hash Join 和聚合。
7. 数据需要跨节点重新分布时，通过 `DataStreamSink → ExchangeNode` 进行 Shuffle。

Doris 扫描时可依次利用分区、分桶、Prefix Index、ZoneMap、Bloom Filter 和倒排索引。Join 的 Build 端还可以生成 Runtime Filter，下推到 Probe 端 Scan，提前减少数据量。

### 同一条聚合 SQL 的处理差异

```sql
SELECT region, SUM(amount)
FROM orders
WHERE order_date >= '2026-07-01'
GROUP BY region
ORDER BY SUM(amount) DESC
LIMIT 10;
```

**MySQL：**

```text
选择索引或全表扫描
→ 逐行读取，必要时回表
→ 过滤和聚合
→ 临时表 / filesort
→ LIMIT 10
```

通常在一个 MySQL Server 内完成。

**ClickHouse：**

```text
分区和 Granule 裁剪
→ 只读取 order_date、region、amount 三列
→ 多线程局部聚合
→ 合并聚合状态
→ Top-N
```

如果是分布式表，各 Shard 先局部聚合，再由发起节点合并。

**Doris：**

```text
FE 裁剪分区和 Tablet
→ 多个 BE 扫描本地数据并局部聚合
→ 按 region Hash Shuffle
→ Final Aggregate
→ Top-N
```

Doris 天然会将一条分析查询拆到多个节点执行。

### 选型建议

- 订单、支付、库存、账户余额等强事务业务：选择 **MySQL**。
- 日志、埋点、监控指标、海量明细扫描聚合：优先 **ClickHouse**。
- 实时数仓、BI、星型模型、多表 Join、CDC UPSERT：优先 **Doris**。
- 常见架构是 MySQL 保存交易数据，再通过 Binlog/CDC 同步到 ClickHouse 或 Doris 做分析。

ClickHouse 与 Doris 的选择可以概括为：

- 表比较扁平、以追加写和扫描聚合为主：更偏 ClickHouse。
- Join 较多、需要 CBO、UPSERT 和标准数仓建模：更偏 Doris。

### 面试总结

> MySQL 的核心是行存、B+Tree、MVCC 和事务；ClickHouse 的核心是列存、稀疏索引、数据跳读与 Pipeline；Doris 的核心是 FE 集中优化、BE 分布式列式执行和 MPP Shuffle。

## 2. 关键概念补充

### 2.1 AST：抽象语法树

AST 是 `Abstract Syntax Tree`，即抽象语法树，是 Parser 对 SQL 进行语法解析后生成的结构化表示。

例如：

```sql
SELECT name, age + 1 AS next_age
FROM users
WHERE id = 10;
```

可以抽象为：

```text
Select
├── Projection
│   ├── Column: name
│   └── Alias: next_age
│       └── Add
│           ├── Column: age
│           └── Literal: 1
├── From
│   └── Table: users
└── Where
    └── Equal
        ├── Column: id
        └── Literal: 10
```

AST 会省略逗号、括号和换行等不影响语义的细节，只保留查询字段、表、表达式和条件之间的关系。

AST 只描述“要做什么”，还没有决定使用哪个索引、采用什么 Join 算法或是否分布式执行。后续还需要经过语义分析、逻辑改写和物理计划优化。

```text
SQL → Token → AST → 语义分析 → 逻辑计划 → 物理计划 → 执行
```

语法正确但字段不存在的 SQL 可以成功生成 AST，随后在语义分析阶段报错。

### 2.2 FE 与 BE：前端节点和后端节点

FE、BE 是 Doris 分布式架构中的两类组件。

**FE（Frontend）是大脑和调度中心，主要负责：**

- 接收 MySQL 协议连接，完成认证和权限检查。
- 管理库表、分区、Tablet、副本和集群拓扑等元数据。
- 解析 SQL，生成 AST、逻辑计划和物理计划。
- 使用 Nereids 优化器选择执行计划。
- 将计划拆成多个 Fragment，并调度到不同 BE。
- 汇总执行状态，并将结果返回客户端。

**BE（Backend）是存储与计算节点，主要负责：**

- 存储 Tablet 数据和副本。
- 扫描列式文件，并利用索引排除无关数据。
- 执行过滤、Join、聚合和排序。
- 在不同 BE 之间传输和 Shuffle 数据。
- 执行 FE 下发的 Fragment 和 PipelineTask。

```text
客户端 SQL
   ↓
FE：解析、优化、拆分、调度
   ↓
BE-1 / BE-2 / BE-3：扫描与计算
   ↓
Root Fragment 汇总
   ↓
FE 返回结果
```

FE 通常不负责扫描海量业务数据，真正的数据读取和计算主要发生在 BE。

### 2.3 CBO：基于代价的优化器

CBO 是 `Cost-Based Optimizer`，即基于代价的优化器。它会生成多个可行执行方案，估算每种方案的成本，然后选择预计成本最低的方案。

CBO 通常考虑：

- 表和列的统计信息、行数及基数。
- 预计扫描的数据量。
- CPU、内存和磁盘 I/O 成本。
- Join 后预计产生的行数。
- 网络传输和 Shuffle 成本。

例如：

```sql
SELECT *
FROM orders o
JOIN users u ON o.user_id = u.id;
```

如果 `orders` 有 10 亿行，而 `users` 只有 10 万行，CBO 可能选择把小表 `users` 广播到所有 BE，让每个 BE 在本地完成 Join；如果两张表都很大，则可能选择按照 `user_id` 进行 Shuffle Join。

CBO 与 RBO 的区别：

```text
RBO：按照固定规则进行谓词下推、列裁剪、常量折叠、分区裁剪
CBO：比较 Join 顺序、Join 算法和数据分布方式的预计成本
```

CBO 强依赖统计信息。统计信息过期或数据分布估算不准时，可能选择错误的 Join 顺序或 Shuffle 策略。

### 2.4 Shuffle：分布式数据重分布

Shuffle 是按照某个 Key，把不同节点上的数据重新发送到指定节点，使相同 Key 的数据最终进入同一个计算节点。

例如执行：

```sql
SELECT region, SUM(amount)
FROM orders
GROUP BY region;
```

原始数据可能分散在不同 BE：

```text
BE-1：北京 100、上海 200
BE-2：北京 300、广州 400
BE-3：上海 500、广州 600
```

按照 `hash(region) % BE数量` 重新分布后：

```text
BE-1：北京 100、北京 300
BE-2：上海 200、上海 500
BE-3：广州 400、广州 600
```

每个 BE 就可以完成对应地区的最终聚合。Join 也类似：将两张表按照 Join Key 重新分布，使能够匹配的记录落到同一个 BE。

Doris 中的网络 Shuffle 通常表现为：

```text
上游 Fragment
→ DataStreamSink
→ 网络传输
→ ExchangeNode
→ 下游 Fragment
```

常见数据分布方式包括：

- `HASH_PARTITIONED`：按照 Key 的 Hash 分发，常用于 Group By 和 Join。
- `BROADCAST`：把小表完整复制到所有节点。
- `RANDOM`：随机或轮询分发，用于均衡负载。
- `UNPARTITIONED`：把数据集中到一个节点，常用于最终汇总。

Shuffle 会产生序列化、网络传输、缓冲和可能的磁盘落盘成本，也是分布式查询中最昂贵的环节之一。通常可以通过局部预聚合、Broadcast 小表、Colocate Join、Bucket Shuffle、谓词下推和 Runtime Filter 减少 Shuffle 数据量。

### 概念关系总结

```text
SQL
→ FE 中的 Parser 生成 AST
→ RBO 进行规则改写
→ CBO 选择成本较低的物理计划
→ FE 将计划拆成 Fragment 并下发
→ BE 执行扫描、Join 和聚合
→ 必要时通过 Shuffle 在 BE 之间重新分布数据
→ FE 返回结果
```

> AST 描述“用户要做什么”；FE 负责解析、优化和调度；CBO 决定“怎样执行成本更低”；BE 负责真正读取和计算；Shuffle 负责让跨节点的数据按照计算需要重新聚集。

### 官方资料

- [MySQL 存储引擎架构](https://dev.mysql.com/doc/refman/8.4/en/pluggable-storage-overview.html)
- [MySQL InnoDB 聚簇与二级索引](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [ClickHouse 查询执行过程](https://clickhouse.com/docs/guides/developer/understanding-query-execution-with-the-analyzer)
- [ClickHouse MergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [Doris Nereids 优化器](https://doris.apache.org/docs/4.x/query-acceleration/optimization-technology-principle/query-optimizer/)
- [Doris 系统架构](https://doris.apache.org/docs/4.x/features-architecture/system-architecture/)
- [Doris MPP 架构](https://doris.apache.org/docs/4.x/key-features/mpp/)
