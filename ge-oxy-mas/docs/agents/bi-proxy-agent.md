# bi_proxy_agent

`bi_proxy_agent` 定义在 `applications/bi_proxy_agent/bi_proxy_oxy_space.py`，由 `data_oxy_space` 合入主 MAS。当 `data_agent` 发现 `str(shared_data.type) == "3"` 时，会把请求转给 `bi_proxy_agent`。

## 子 Agent 与流程

| Agent | 类型 | 功能 |
| --- | --- | --- |
| `bi_proxy_agent` | `WorkflowAgent` | BI 主代理，控制大纲生成、用户确认和二次生成 dashboard。 |
| `bi_agent` | `WorkflowAgent` | 远端 BI 服务代理，向 `config.bi_proxy_agent.server_url` 发送请求并校验返回。 |

`bi_proxy_agent_workflow` 会先把 `shared_data.report_pref.outline` 设置为 `self`，调用 `bi_agent` 获取待确认大纲，并以 todolist 卡片发给前端。如果原始请求要求手动确认大纲，就通过 `poll_get_chat_append_message` 等待用户修改或确认；确认后再把 `outline` 改为 `auto`，把大纲写入 `outline_template`，第二次调用 `bi_agent` 生成 dashboard。

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | 无本地 FunctionHub 工具；`bi_agent` 封装远端 BI HTTP 服务。 |
| Prompt | 本仓库无 BI 生成 prompt；大纲和 dashboard 生成由远端 BI 服务处理。 |
| 子 Agent | `bi_proxy_agent → bi_agent`。 |
| 人机交互 | 手动大纲确认通过 `poll_get_chat_append_message` 等待前端 append 消息。 |

手动大纲确认的等待时间来自 DUCC key `outline_confirm_timeout`，当前实际值见 [../prompt文档.md](../prompt文档.md)。
