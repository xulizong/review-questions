# OxyGent 运行模型

本文说明所有业务 Agent 共享的运行机制。业务细节请进入对应 Agent 文档阅读；这里仅解释请求如何进入 MAS，`oxy_space` 如何注册能力，以及 `WorkflowAgent`、`ReActAgent`、`FunctionHub`、远程 Agent 和 MCP tool 如何协作。

## 1. 启动与注册

项目启动入口是 `main.py`。启动时 `configs/app.py` 会读取环境变量 `AGENT` 和 `ENV`，先加载公共配置 `configs/application.yml`，再加载 `configs/application-{agent}-{env}.yml` 并做深度合并。业务配置里的 `agents.agent_path` 指向一个 Python 模块，例如 `modules.agents.search_agents.main`，该模块必须暴露 `oxy_space`。

`main.py` 中的 `MasService.init_mas()` 会执行 `getattr(importlib.import_module(config.agents.agent_path), config.agents.oxy_space)`，拿到 `oxy_space` 后调用 `MAS.create(oxy_space=oxy_space)`。`MAS.init()` 会注册所有 Oxy 实例，初始化 ES/Redis，初始化 LLM、tool、agent，选择 `is_master=True` 的 master agent，并构造 Agent 组织树。

## 2. 统一请求路径

一次请求进入 FastAPI 后，会被 OxyGent 路由转换为 `OxyRequest`。`MAS.chat_with_agent()` 会补齐 `shared_data.query`、trace、附件和流式消息 key，再把请求交给当前服务的 master agent。不同 `AGENT` 启动的是不同 master，例如 `datagent`、`search_agent`、`query_agent`、`query_ds_agent`、`text_2_sql_agent` 或 `augment_agent`。

```text
HTTP / SSE request
    ↓
main.py / oxygent.routes
    ↓
MAS.chat_with_agent()
    ↓
master agent
    ↓
req.call(callee=...)
    ↓
子 Agent / FunctionTool / MCPTool / RemoteAgent / LLM
    ↓
OxyResponse
```

## 3. Agent 类型

| 类型 | 代码位置 | 执行方式 | 常见用途 |
| -- | -- | -- | -- |
| `WorkflowAgent` | `oxygent/oxy/agents/workflow_agent.py` | 直接执行 `func_workflow(req)` | 业务编排、路由、聚合 |
| `ReActAgent` | `oxygent/oxy/agents/react_agent.py` | 构造 prompt，调用 LLM，解析为答案或 tool-call，多轮循环 | 意图识别、NER、SQL 生成、总结 |
| `ChatAgent` | `oxygent/oxy/agents/chat_agent.py` | 单次聊天式 LLM 调用 | 页面总结、知识总结 |
| `SSEOxyGent` | `oxygent/oxy/agents/sse_oxy_agent.py` | 通过 SSE 调用另一个 OxyGent 服务 | 跨服务复用 Agent |
| `SSEMCPClient` / `StdioMCPClient` | `oxygent/oxy/mcp_tools/*` | 连接 MCP 服务并展开工具 | ClickHouse、外部工具服务 |
| `FunctionHub` | `oxygent/oxy/function_tools/function_hub.py` | 把 `@hub.tool` 函数展开成 `FunctionTool` | 本地 Python 工具 |

`WorkflowAgent` 适合确定性的业务编排，因为调用链完全由 Python 代码控制。`ReActAgent` 适合 LLM 参与判断或生成的节点，因为它会把 prompt、工具说明、短期记忆和用户问题组合给模型，并根据模型输出继续调用工具或返回答案。

## 4. Tool 展开规则

`LocalAgent._init_available_tool_name_list()` 决定了一个 Agent 能调用什么。`sub_agents` 会被当成可调用对象加入工具列表；`tools` 中如果写的是 `FunctionHub` 名称，会展开 hub 下所有 `@tool` 函数；如果写的是 MCP Client 名称，会展开 MCP 服务暴露的具体工具名；如果写的是具体 `FunctionTool` 或 `MCPTool` 名称，则直接加入。

这意味着新增工具时只写函数不够，还必须把对应 `FunctionHub` 放入 `oxy_space`。同样，如果 Agent 的 `tools` 引用了一个具体函数名，但该函数所属 hub 没有注册到 MAS，初始化阶段会报 tool 不存在。

## 5. Prompt 注入

Prompt 的绑定点通常在业务 `main.py` 中的 `prompt=...` 参数。运行时 `LocalAgent._before_execute()` 会把 `tools_description`、`additional_prompt`、query、附件和短期记忆写入 `OxyRequest.arguments`，`ReActAgent._execute()` 再调用 `_build_instruction()` 渲染系统提示。Text2SQL 这类复杂链路还会在各 Agent 的 `func_format_input` 中写入数据集 schema、业务知识、当前日期、候选 SQL、执行结果和错误信息。

生产 prompt 多来自 LAF/DUCC，例如各模块的 `laf_clients.py` 和 `modules/ducc/laf_instance.py`。本地 prompt 文件一般作为 fallback 或特定 Agent 的固定模板。

## 6. 状态与可观测性

`MAS.init_db()` 会创建 `{app}_trace`、`{app}_node`、`{app}_history` 和可选 `{app}_message`。`trace` 记录一次调用的整体输入输出，`node` 记录每个 Agent/tool 节点，`history` 记录短期记忆，`message` 用于持久化 SSE 消息。业务代码中普遍使用 `LoginContext` 获取 ERP 和请求 headers，使用 `ump.async_ump_log`、`CallerInfo`、`LogContext` 记录耗时和异常。
