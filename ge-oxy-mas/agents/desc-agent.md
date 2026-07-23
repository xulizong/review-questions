# desc_oxy_space

`desc_oxy_space` 定义在 `applications/desc_agent/desc_oxy_space.py`，由 `main.py` 创建为 `app.state.desc_mas`。它不是普通 `/chat` 的默认入口，而是由 `webs/data_router.py` 与 `services/data_service.py` 在数据描述、推荐、摘要、标题和网页能力接口中直接调用。

## Agent 组

| Agent | 类型 | 功能 | Tool / Prompt |
| --- | --- | --- | --- |
| `desc_agent` | `ChatAgent` | 对文件或数据集字段、描述和样例生成 100-200 字概述。 | Prompt：`get_desc_agent_prompt`。 |
| `report_recommend_agent` | `ReActAgent` | 根据报告或当前数据上下文推荐后续问题。 | Prompt：`get_report_recommend_agent_prompt`。 |
| `bi_recommend_agent` | `ReActAgent` | 根据 BI 场景推荐后续问题。 | Prompt：`get_bi_recommend_agent_prompt`。 |
| `chat_summary_agent` | `ReActAgent` | 生成会话整体摘要和推荐问题，并校验为 `ChatRecommend` 结构。 | Prompt：`get_chat_summary_agent_prompt`。 |
| `chat_title_agent` | `ChatAgent` | 根据会话内容生成标题。 | Prompt：`get_chat_title_agent_prompt`。 |
| `web_search_agent` | `WorkflowAgent` | 调用描述侧联网搜索工具。 | Tool：`web_search`；无直接 prompt。 |
| `web_content_summary_agent` | `WorkflowAgent` | 调用网页内容总结工具。 | Tool：`web_content_summary`；工具内部有 `summary_prompt`。 |

## API 入口

`/chat/summary` 调用 `chat_summary_agent`，`/chat/title` 调用 `chat_title_agent`，`/report/recommend` 根据 recommend type 调用 `report_recommend_agent` 或 `bi_recommend_agent`，`/data/web_search` 调用 `web_search_agent`，`/data/web_content_summary` 调用 `web_content_summary_agent`。文件和数据集的描述生成逻辑在 `services/data_service.py` 中会先提取字段、样例和基础质量信息，再把拼好的 query 交给 `desc_agent`。

## Tool 详解

| Tool | 实现位置 | 使用 Agent | 说明 |
| --- | --- | --- | --- |
| `web_search` | `extends/web_search/web_search_client.py` | `web_search_agent` | 调用配置中的搜索模型接口，解析 `tool_calls.search_result` 返回搜索结果列表。 |
| `web_content_summary` | `extends/web_search/web_content_summary_client.py` | `web_content_summary_agent` | 调用外部文档理解模型读取 URL，并按工具内部 `summary_prompt` 输出标题、正文、摘要和 chunk。 |

## Prompt 来源

描述侧 prompt 同样通过 `extends/ducc/laf_instance.py` 读取，默认回退到 `applications/desc_agent/prompts.py`。网页内容总结的 prompt 不走 `laf_instance`，而是定义在 `extends/web_search/web_content_summary_client.py` 的 `summary_prompt` 变量里。

当前 DUCC/LAF 快照未覆盖描述侧 prompt provider，因此这些 provider 会回退本地默认值；具体 fallback 列表见 [../prompt文档.md](../prompt文档.md)。
