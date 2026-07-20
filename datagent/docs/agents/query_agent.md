# Query Agent 查数 Agent

Query Agent 是 `AGENT=query-agent` 时的 master agent，代码入口是 `modules/agents/query_agents/main.py`，核心 workflow 在 `modules/agents/query_agents/query_agent.py`。它不负责底层实体归一，也不直接查 OneService；它的主要职责是复用 Search Agent 的 DSL，选择核心指标和聚合维度，组装 Query DS Agent 能执行的数据查询 DSL。

## 1. Agent 组成

| 名称 | 类型 | 功能 | Prompt / 依赖 |
| -- | -- | -- | -- |
| `query_agent` | `WorkflowAgent` | master，查数主流程 | 无直接 prompt |
| `search_agent` | `SSEOxyGent` | 远程找数服务，强制走结构化找数路径 | URL 来自 `config.agents.query_agent.search_agent_url` |
| `core_ind_agent` | `ReActAgent` | 从 Search DSL 候选中选择核心指标、排序和 topN | `get_core_ind_agent_prompt()` |
| `core_dim_agent` | `ReActAgent` | 判断是否需要聚合，并选择 group 字段 | `get_core_dim_agent_prompt()` |
| `query_ds_agent` | `SSEOxyGent` | 远程数据查询服务 | URL 来自 `config.agents.query_agent.query_ds_agent_url` |
| `query_recommended_agent` | `ReActAgent` | 基于下钻配置生成推荐问题 | `get_query_recommended_agent_prompt()` |
| `word_segmentation_agent` | `ReActAgent` | 分词能力 | `get_word_segmentation_agent_prompt()` |
| `word_rewrite_agent` | `ReActAgent` | 词汇重写能力 | `get_word_rewrite_agent_prompt()` |

## 2. 主流程

`query_agent_func_workflow(req)` 第一步调用远程 `search_agent`，并传 `search_intent=search_workflow_agent`。这样 Search Agent 一定进入结构化找数，而不是 RAG 知识问答。Search Agent 返回后，Query Agent 把候选结果分成 `valid` 和 `invalid`。如果没有任何有效候选，直接把错误列表返回给用户。

如果存在有效候选，workflow 会从候选里取一个改写后的 query，然后调用 `core_ind_agent`。这个 Agent 的输入包括用户 query 和候选指标集合，输出核心指标名、排序方式和限制条数。Query Agent 根据核心指标找到对应 Search DSL，并调用 `set_ind_with_sub_ind()` 自动补充同比、环比、农历同比等派生指标。

随后 workflow 调用 `core_dim_agent`，输入是用户 query 和可能字段列表。可能字段由 Search DSL 识别到的维度加上 `dt` 日期字段组成。`core_dim_agent` 输出是否 group，以及 group 字段。Query Agent 根据日期粒度、group 字段、过滤维度、指标列表、排序和分页组装数据查询 DSL，再调用远程 `query_ds_agent`。

```text
query_agent
    ↓
search_agent(search_workflow_agent)
    ↓
valid / invalid 候选拆分
    ↓
core_ind_agent
    ↓
补充派生指标
    ↓
core_dim_agent
    ↓
组装 Query DS DSL
    ↓
query_ds_agent
    ↓
结果包装和推荐问题生成
```

## 3. 输出结构

Query Agent 在 Query DS 返回成功后，会把结果包装成面向前端的结构。`data` 来自 Query DS 的数据明细，`filters` 包含日期、分组字段和过滤维度，`fields` 是表头字段并补充 formatter、direction 等指标元数据，`errors` 合并数据服务错误和 Search DSL 中的错误，`recommendations` 是推荐追问。

推荐追问不是每次都有。workflow 会从 `get_ind_recommended()` 和 `get_dim_recommended()` 读取下钻配置，若存在推荐指标或推荐维度，才调用 `query_recommended_agent` 生成自然语言问题。

## 4. 示例

用户问“昨天成交金额 TOP10 店铺”。Search Agent 会输出包含时间、成交金额指标、店铺维度等信息的 DSL。`core_ind_agent` 选择“成交金额”为核心指标，并可能识别排序方向为降序、limit 为 10。`core_dim_agent` 判断需要按店铺聚合。Query Agent 组装出的 DSL 会包含 `date`、`groups`、`attributes`、`dims`、`indicators`、`sorts`、`page` 和 `common.pin`，然后交给 Query DS Agent 查询。

用户问“昨天成交金额是多少”。如果没有“按店铺”“按天”这类聚合意图，`core_dim_agent` 可能输出不 group。最终 DSL 就更像单指标聚合查询，返回一个总数或少量行。
