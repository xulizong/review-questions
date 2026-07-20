# enhance_agent_flow

`enhance_agent_flow` 定义在 `applications/master_agent/agent/intention/intention_enhance.py`，由 `data_oxy_space` 合入主 MAS。`webs/chat_router.py` 的 `/chat/enhance` 会显式把 payload 的 `callee` 设置为 `enhance_agent_flow`，因此它是独立接口入口，不经过 `data_agent` 的普通分流。

## 子 Agent 与流程

| Agent | 类型 | 功能 | Prompt |
| --- | --- | --- | --- |
| `enhance_agent_flow` | `WorkflowAgent` | 提示词增强入口，补齐数据描述、知识和上下文后调用增强 Agent。 | 无直接 prompt。 |
| `enhance_agent` | `ChatProAgent` | 把模糊问题扩写成清晰、可执行的数据查询或分析任务。 | `get_enhance_agent_prompt`，backup 为 `get_enhance_agent_prompt_backup`。 |
| `extract_placeholder_agent` | `WorkflowAgent` | 占位符模式下复用 master 子树的占位符提取能力。 | 详见 [master-agent.md](./master-agent.md)。 |

`enhance_flow` 会先兼容 `@@admin@@提示词增强` 的旧触发方式，再调用 `pre_process_master_agent`、`add_data_desc` 和 `pre_process_clarifier` 补上下文。若 `master_tool.get_input_mode` 判断为 `intention_enhance_placeholder`，会拼入占位符安全约束，禁止从占位符值推导出额外时间或公式。增强完成后，如果仍是占位符模式，会再调用 `extract_placeholder_agent`。

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | `enhance_agent_flow` 声明 `planner_tools`，但核心流程主要通过 workflow 和子 Agent 完成。 |
| Prompt | `enhance_agent` 使用 `get_enhance_agent_prompt`；长文本/备份使用 `get_enhance_agent_prompt_backup`。 |
| 子 Agent | `enhance_agent`、`extract_placeholder_agent`。 |

当前 DUCC/LAF 快照未覆盖增强链路 prompt provider，因此 `enhance_agent` 会回退本地默认 prompt；具体 provider 和 fallback 位置见 [../prompt文档.md](../prompt文档.md)。
