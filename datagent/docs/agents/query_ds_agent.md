# Query DS Agent 数据查询 Agent

Query DS Agent 是 `AGENT=query-ds-agent` 时的 master agent，代码入口是 `modules/agents/query_ds_agents/main.py`，核心 workflow 在 `modules/agents/query_ds_agents/query_ds_agent.py`。它的职责是执行上游传入的数据查询 DSL，并根据 `out_type` 返回 JSON、写入 ClickHouse 或执行其他输出方式。

## 1. Agent 与工具组成

| 名称 | 类型 | 功能 | 来源 |
| -- | -- | -- | -- |
| `query_ds_agent` | `WorkflowAgent` | master，解析 DSL 并按输出类型执行 | `query_ds_agent_func_workflow` |
| `query_ds_one_service` | `FunctionTool` | 将 DSL 转成 OneService 请求并查询指标数据 | `query_ds_one_service_tools` |
| `clickhouse_tools` | `SSEMCPClient` | 提供 `select`、`insert`、`execute` 等 ClickHouse 工具 | `config.agents.query_ds_agent.store_ck_sse_url` |

## 2. 主流程

`query_ds_agent_func_workflow(req)` 会先调用 `get_dsl(req)`，将 `arguments.dsl` 转为 `Dsl` 对象。如果调用方传的是字符串，会先 JSON 解析；如果字段不符合模型，会返回参数错误。解析成功后，workflow 会把 `dsl.common.query` 设置为 `origin_query`，并把 `dsl.common.pin` 重置为当前登录 ERP，避免上游伪造用户身份。

核心分支由 `dsl.common.out_type` 控制。默认或空值走 JSON 输出，调用 `process_json()`；`ck` 会申请 RedisTaskLock，创建 ClickHouse 本地表和分布式表，分页查询并写入；`ck_union` 会按指标和 suffix 拆分查询后写入同一张宽表；`oss` 当前直接返回暂未开放。

```text
query_ds_agent
    ↓
get_dsl
    ↓
重置 query 和 pin
    ↓
按 out_type 分支
    ├─ json → process_json
    ├─ ck → process_ck
    ├─ ck_union → process_ck_union
    └─ oss → 暂未开放
```

## 3. JSON 查询路径

默认 JSON 路径会调用 `get_data(req, dsl)`。`get_data()` 根据 `dsl.common.ds` 调用具体数据工具，典型值是 `query_ds_one_service`。返回后它会检查 `errors`，如果有错误则转成 `SystemException`。如果 DSL 的 attributes 或 groups 配置了 `dim_trans`，`get_data()` 还会调用维度中心 `get_dim_name()`，把维度 ID 翻译成名称字段并合并到结果行中。

`query_ds_one_service` 会把 DSL 里的维度、指标、排序、分页和时间条件转换成 OneService 请求。时间条件会被补成 `time_interval`、`dt >= start`、`dt <= end` 三类 criterion；限流错误会触发 tenacity 重试，最多重试 3 次。

## 4. ClickHouse 输出路径

`ck` 路径用于把查询结果落到 ClickHouse。workflow 会先用 ERP、source、table 名申请 RedisTaskLock，避免同一用户同一场景并发写同一任务。`process_ck()` 会根据维度和指标生成字段列表，创建本地表和分布式表，必要时按日期拆分单天查询。由于单次最多处理 2000 条，`process_ck_query_and_insert()` 会按 page 分批查询，并通过 MCP tool `insert` 写入 ClickHouse。

`ck_union` 路径类似，但会按指标和 suffix 拆分查询，再把不同指标列写入同一张表，适合同比、环比或多指标宽表场景。

## 5. 示例

上游传入一个 DSL，要求查询“昨日成交金额按天”。如果 `out_type` 为空，Query DS Agent 会调用 `query_ds_one_service`，返回 JSON 数据和字段信息。若某个 group 是事业部 ID，并配置了 `dim_trans`，返回行中会额外补充事业部名称字段。

如果同样的 DSL 设置 `out_type=ck`，Agent 会创建 ClickHouse 表，分页查询 OneService 数据并写入表，最终返回 `local_table_name`、`distributed_table_name` 和字段列表。上游拿到的是表信息，而不是完整数据明细。
