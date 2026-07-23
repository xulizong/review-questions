# Datagent Agent 文档索引

本文是当前项目 Agent 文档的总入口，只保留项目启动方式、主要 Agent 清单和详细文档位置。每个主要 Agent 的子 Agent、tool、prompt、执行流程和示例都拆到了 `docs/agents/` 下，阅读时先从这里定位，再进入对应文档看细节。

## 1. 项目入口

项目是基于 FastAPI 和内置 `oxygent` 多 Agent 框架的 Python 服务。启动入口是 `main.py`，配置入口是 `configs/app.py`，运行时通过 `AGENT` 和 `ENV` 选择 `configs/application-{agent}-{env}.yml`。配置里的 `agents.agent_path` 指向具体业务模块，`main.py` 会动态 import 对应模块的 `oxy_space`，再通过 `MAS.create(oxy_space=oxy_space)` 注册 LLM、Agent、tool、MCP Client 和远程 Agent。

通用执行模型见 [OxyGent 运行模型](./agents/oxygent_runtime.md)。如果只关心某个业务 Agent，直接从下面的清单进入。

## 2. 主要 Agent 文档

| 主要 Agent | 代码入口 | 详细文档 | 说明 |
| -- | -- | -- | -- |
| Datagent | `modules/agents/datagents/main.py` | [Datagent 总控 Agent](./agents/datagent.md)，[Datagent Intent Prompt](./agents/datagent_intent_prompt.md) | 顶层意图识别与路由，连接找数、查数、业务知识、页面总结、猜你想问、店铺诊断 |
| Search Agent | `modules/agents/search_agents/main.py` | [Search Agent 找数 Agent](./agents/search_agent.md) | 找数意图、时间/指标/维度识别与归一、指标知识 RAG |
| Query Agent | `modules/agents/query_agents/main.py` | [Query Agent 查数 Agent](./agents/query_agent.md) | 复用 Search DSL，选择核心指标和聚合维度，组装查询 DSL |
| Query DS Agent | `modules/agents/query_ds_agents/main.py` | [Query DS Agent 数据查询 Agent](./agents/query_ds_agent.md) | 执行 OneService 查询、ClickHouse 写入、JSON/CK 输出 |
| Page Summary Agent | `modules/agents/page_summary_agents/main.py` | [Page Summary Agent 页面总结 Agent](./agents/page_summary_agent.md) | 页面上下文整理、知识和 prompt 获取、页面总结、多轮缓存 |
| Text2SQL Agent | `modules/agents/text_2_sql_agents/main.py` | [Text2SQL Agent](./agents/text_2_sql_agent.md)，[Text2SQL Prompt 分析](./agents/text_2_sql_prompt_analysis.md) | 自然语言转 SQL/DSL，数据集召回、SQL 生成、纠错、选举、可视化和总结 |
| Augment Agent | `modules/agents/augment_agents/main.py` | [Augment Agent 日报增强 Agent](./agents/augment_agent.md) | 采销日报任务编排、基础数据加载、字段/文件处理、EasyBI 组件生成 |

## 3. Agent 间关系

Datagent 是业务总控入口时，会把不同意图转发给 Search、Query、Page Summary 或本地工具。Query Agent 会调用 Search Agent 做结构化找数，再调用 Query DS Agent 查数。Text2SQL Agent 是另一套更完整的数据集查询链路，内部有自己的 SQL/DSL 生成、执行、选举、总结和可视化流程。Augment Agent 主要服务日报增强场景，会调用 Query DS Agent 和 ClickHouse MCP 工具，也会使用 EasyBI、文件和字段相关子 Agent。

```text
datagent
    ├─ search_agent
    ├─ query_agent
    │   ├─ search_agent
    │   └─ query_ds_agent
    ├─ ge_ai_page_summary_agent
    ├─ get_business_knowledge
    ├─ get_guess_query
    └─ shop_diagnostic_agent

text_2_sql_agent
    ├─ ge_c_query_analysis_agent
    ├─ ge_c_query_agent
    └─ query_with_summary_agent

augment_agent
    ├─ query_ds_agent
    ├─ add_fields_agent
    ├─ add_file_agent
    ├─ display_agent
    └─ easybi_agent
```

## 4. 维护方式

以后新增或修改某个主要 Agent 时，优先更新 `docs/agents/{agent_name}.md`，不要把细节继续堆到主索引里。主索引只需要补一行文档入口和一句定位说明。跨 Agent 的通用机制，例如 `oxy_space` 注册、`WorkflowAgent`、`ReActAgent`、`FunctionHub` 展开和 trace/history 记录，统一放在 [OxyGent 运行模型](./agents/oxygent_runtime.md)。
