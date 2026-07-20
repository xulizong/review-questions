# sql_proxy_agent

`sql_proxy_agent` 定义在 `applications/sql_proxy_space/sql_proxy_oxy_space.py`，由 `applications/data_agent/data_oxy_space.py` 合入主 MAS。它服务 `data_agent` 的 quick data 分支：当 `shared_data.mode == ChatModelType.QUICK_DATA` 时，`data_agent` 会调用它完成快速查数。

## 子结构与流程

`sql_proxy_agent` 没有子 Agent，也没有 LLM prompt。它是一个 `WorkflowAgent`，核心逻辑在 `sql_proxy_agent_workflow`。流程先把当前请求转换成外部 SQLAgent 的 payload，调用 `config.sql_proxy_agent.chat_server_url` 提交任务，拿到新的 trace id 后再调用 `sql_sse_trace` 消费 `config.sql_proxy_agent.trace_server_url` 的 SSE 流。SSE 中的文本输出会通过项目自己的 `send_message` 转发给前端，最终返回最后一次 output 文本。

```text
data_agent
  ↓ QUICK_DATA
sql_proxy_agent_workflow
  ↓
外部 SQLAgent chat_server_url
  ↓
sql_sse_trace 消费 trace_server_url
  ↓
send_message 转发 SSE
```

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | 无本地 FunctionHub 工具；能力来自外部 SQLAgent HTTP/SSE 服务。 |
| Prompt | 本仓库无对应 prompt；SQL 生成和理解逻辑在外部 SQLAgent。 |
| 鉴权 | 使用 `get_datagent_sign` 生成签名头，同时带固定 Authorization/app_id。 |
| 错误处理 | 对业务异常、网络超时、RequestError 做错误消息发送和最多 3 次 SSE 重试。 |

