# Agent 执行流程文档入口

这份文档原本承载了所有主要 Agent 的执行流程和示例。为避免不同 Agent 的细节耦合在同一个长文档里，内容已拆分到 `docs/agents/` 目录。后续阅读和维护请以 [Datagent Agent 文档索引](./project_recon_agent_tool_prompt_mapping.md) 为主入口。

| Agent | 执行流程文档 |
| -- | -- |
| 通用 OxyGent 运行模型 | [OxyGent 运行模型](./agents/oxygent_runtime.md) |
| Datagent | [Datagent 总控 Agent](./agents/datagent.md) |
| Search Agent | [Search Agent 找数 Agent](./agents/search_agent.md) |
| Query Agent | [Query Agent 查数 Agent](./agents/query_agent.md) |
| Query DS Agent | [Query DS Agent 数据查询 Agent](./agents/query_ds_agent.md) |
| Page Summary Agent | [Page Summary Agent 页面总结 Agent](./agents/page_summary_agent.md) |
| Text2SQL Agent | [Text2SQL Agent](./agents/text_2_sql_agent.md) |
| Augment Agent | [Augment Agent 日报增强 Agent](./agents/augment_agent.md) |

新文档的组织原则是：一个主要 Agent 一份文档，该 Agent 自己的子 Agent、tool、prompt、执行流程、关键分支和示例都放在同一份文档中。主索引只负责导航，不再承载详细说明。
