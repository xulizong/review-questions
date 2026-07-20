# chat summarize agent

`summary_agent_flow` 定义在 `applications/master_agent/run_summary_agent.py`，由 `data_oxy_space` 合入主 MAS。`webs/chat_router.py` 的 `/chat/summarize` 会显式设置 `callee = "summary_agent_flow"`，用于独立 AI 总结块场景。它不是 `master_agent` 普通常规主流程末尾的最终总结；主流程最终总结见 [global-summary-agent.md](./global-summary-agent.md)。

## 子 Agent 与流程

| Agent | 类型 | 功能 | Prompt |
| --- | --- | --- | --- |
| `summary_agent_flow` | `WorkflowAgent` | 读取用户总结要求和 `shared_data.tasks`，规范化成 JSON 字符串后调用总结 Agent。 | 无直接 prompt。 |
| `summary_agent` | `BackupChatAgent` | 严格基于 `tasks.sections.content` 输出 Markdown 总结，不编造数据。 | `SUMMARY_AGENT_PROMPT`。 |

`summary_flow` 会把用户 query 作为 `summary_instruction`，把 `shared_data.tasks` 作为 `tasks`，序列化成一个规范化 JSON，然后调用 `summary_agent`。`SUMMARY_AGENT_PROMPT` 要求输出 Markdown，标题从二级标题开始，且不得编造未出现的数据、指标或结论。

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | 无业务工具。 |
| Prompt | `SUMMARY_AGENT_PROMPT` 直接定义在 `applications/master_agent/run_summary_agent.py`。 |
| LLM | `summary_llm` 与 `summary_llm_backup`，由环境变量配置。 |
