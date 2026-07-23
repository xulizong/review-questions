# Page Summary Agent 页面总结 Agent

Page Summary Agent 是 `AGENT=page-summary-agent` 时的 master agent，代码入口和主 workflow 都在 `modules/agents/page_summary_agents/main.py`。它的职责是把页面上下文、页面知识、分析思路和用户问题组合起来，调用 ChatAgent 生成页面总结，并支持灰度、缓存和多轮追问。

## 1. Agent 组成

| 名称 | 类型 | 功能 | Prompt / 配置 |
| -- | -- | -- | -- |
| `ge_ai_page_summary_agent` | `WorkflowAgent` | master，整理页面上下文并调用总结 Agent | 无直接 prompt |
| `ge_ai_page_summary_chat_agent` | `ChatAgent` | 基于页面 prompt 生成总结 | `get_prompt()` |

额外配置来自 `modules/agents/page_summary_agents/laf_clients.py`。`get_open_percent()` 控制灰度比例，`get_open_whitelist()` 控制白名单，`get_open_reject_msg()` 控制降级文案，`get_prompt()` 是 ChatAgent 的基础 prompt。页面级知识和分析思路由 `config_respository.py` 从 ES 中读取，支持 default 和 custom prompt。

## 2. 主流程

`ge_ai_page_summary_agent_func_workflow(req)` 首先读取当前 ERP，并根据 hash、灰度比例和白名单判断是否开放。如果未开放，它会发送 stream 消息并返回降级文案。

通过灰度后，workflow 区分首次请求和多轮请求。首次请求从 `shared_data` 里读取 `app`、`scene`、`configs.menu_id`、日期、筛选器和页面模块数据，并把模块数据转换成 Markdown 表格。多轮请求从 `group_data` 恢复 app、scene、menu_id、date_desc、filter_desc 和 module_desc，同时写回 `shared_data`，确保追问仍基于原页面。

随后 workflow 校验必要参数。如果 `scene`、`menu_id`、`date_desc`、`filter_desc` 或 `module_desc` 缺失，会返回“页面解读不支持从对话触发，请从页面点击智能解读按钮”。参数完整时，它调用 `get_config(app, scene, menu_id, erp)` 获取 `knowledge`、`prompt_type`、`prompt_detail` 和 `menu_name`，再用这些上下文生成 MD5 cache key，优先读取 JimDB 缓存。没有缓存时，首次请求会先发送 `basic` 消息，然后调用 `ge_ai_page_summary_chat_agent`。

```text
ge_ai_page_summary_agent
    ↓
灰度 / 白名单校验
    ↓
首次请求构造页面上下文，或多轮请求恢复 group_data
    ↓
get_config 获取知识和 prompt_detail
    ↓
JimDB 缓存检查
    ↓
ge_ai_page_summary_chat_agent
    ↓
页面总结
```

## 3. 输入上下文

首次请求依赖页面传入完整 `shared_data`。日期会被整理成 `startDate ~ endDate`，筛选器会被整理成 Markdown 列表，页面模块会被整理成 Markdown 表格。这些内容作为 `date_desc`、`filter_desc`、`module_desc` 传给 ChatAgent。workflow 还会把 app、scene、menu_id 和整理后的上下文写入 `group_data`，用于多轮。

## 4. 示例

页面点击“智能解读”时传入菜单 ID、日期范围“2026-06-01 ~ 2026-06-30”、筛选器“事业部=3C 数码事业部”和多个模块表格。Page Summary Agent 会把这些数据转换成 prompt 参数，读取该菜单的知识和分析思路，然后调用 ChatAgent 输出页面总结。

如果用户在同一 trace 里继续追问“为什么成交金额下降”，多轮请求会从 `group_data` 恢复上一次页面上下文。用户不需要重新传页面模块数据，ChatAgent 仍然能基于同一页面信息回答。
