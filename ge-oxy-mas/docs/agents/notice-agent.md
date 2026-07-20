# notice_agent

`notice_agent` 定义在 `applications/notice_agent/main.py`，由 `data_oxy_space` 合入主 MAS。它主要由 `applications/notice_agent/notice_flow.py` 的 `notice(...)` 函数触发，用于对特定 plan type 和 app 发送京Me预通知或结果通知。

## 子 Agent 与流程

| Agent | 类型 | 功能 | Prompt |
| --- | --- | --- | --- |
| `notice_agent` | `WorkflowAgent` | 京Me 通知入口，包装预通知和结果通知调用。 | 无直接 prompt。 |
| `msg_summary_agent` | `ReActAgent` | 将长分析结果压缩为通知标题和内容。 | `get_msg_summary_agent_prompt`，默认来自 `applications/notice_agent/prompt.py`。 |

预通知不需要模型总结，直接从 `get_long_query_jme_pre_notice_msg()` 读取配置并调用 `config.notice_agent.pre_notice_url`。结果通知会先调用 `msg_summary_agent`，要求输出 JSON 且包含 `title`、`content` 两个字段，然后调用 `config.notice_agent.result_notice_url`。

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | 无本地 FunctionHub 工具；外部能力是京Me通知 HTTP 接口。 |
| Prompt | 只有 `msg_summary_agent` 使用 `get_msg_summary_agent_prompt`。 |
| 配置 | `notice_scene`、`notice_app`、`notice_disabled`、`long_query_jme_pre_notice_msg` 都由 `extends/ducc/laf_instance.py` 提供。 |

通知链路当前实际读取到的 DUCC/LAF 值见 [../prompt文档.md](../prompt文档.md)，包括 `msg_summary_agent_prompt`、`notice_scene`、`notice_app`、`notice_disabled` 和 `long_query_jme_pre_notice_msg`。
