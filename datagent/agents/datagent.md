# Datagent 总控 Agent

Datagent 是 `AGENT=datagent` 时的 master agent，代码入口是 `modules/agents/datagents/main.py`，核心 workflow 在 `modules/agents/datagents/datagent.py`。它不是一个直接完成所有任务的单体 Agent，而是数据场景的总控路由层：先通过意图 Agent 判断用户要“查数、找指标解释、查业务知识、做页面总结、猜你想问、店铺诊断”中的哪一类，再把请求交给远程 Agent 或本地 FunctionTool。

## 1. Oxy Space 注册结构

`modules/agents/datagents/main.py` 中的 `oxy_space` 注册了三个 LLM、一个 master workflow、一个意图识别 ReActAgent、三个远程 SSE Agent、一个店铺诊断 workflow，以及两个 FunctionHub。`datagent` 自身标记 `is_master=True`，所以外部请求进入该服务时默认先执行 `datagent_func_workflow()`。

| 注册对象 | 类型 | 作用 | 配置来源 |
| -- | -- | -- | -- |
| `datagent_model` | `HttpLLM` | master workflow 和店铺诊断 workflow 绑定的模型名，实际 workflow 主要是 Python 编排 | `config.agents.datagent.model` |
| `datagent_joy_builder_model` | `HttpLLM` | 预留或扩展模型，当前 datagent 主流程未直接使用 | `config.agents.datagent.joy_builder_model` |
| `datagent_gateway_model` | `HttpLLM` | `datagent_intent_agent` 使用的意图识别模型 | `config.agents.datagent.gateway_model` |
| `datagent` | `WorkflowAgent` | master，总控路由 | `datagent.datagent_func_workflow` |
| `datagent_intent_agent` | `ReActAgent` | 用户意图识别和查数 query 改写 | `get_datagent_intent_agent_prompt()` |
| `search_agent` | `SSEOxyGent` | 远程找数/指标知识 Agent | `config.agents.datagent.search_agent_url` |
| `query_agent` | `SSEOxyGent` | 远程查数 Agent | `config.agents.datagent.query_agent_url` |
| `ge_ai_page_summary_agent` | `SSEOxyGent` | 远程页面总结 Agent | `config.agents.datagent.ge_ai_page_summary_agent_url` |
| `shop_diagnostic_agent` | `WorkflowAgent` | 店铺诊断转发和熔断处理 | `shop_diagnostic_func_workflow` |
| `autobots_knowledge_tools` | `FunctionHub` | 展开成本地工具 `get_business_knowledge` | `modules/agents/datagents/tools/autobots_knowledge_tools.py` |
| `guess_query_tools` | `FunctionHub` | 展开成本地工具 `get_guess_query` | `modules/agents/datagents/tools/guess_query_tools.py` |

`datagent` 的 `sub_agents` 配置为 `["datagent_intent_agent", "search_agent", "query_agent", "ge_ai_page_summary_agent", "get_business_knowledge", "get_guess_query", "shop_diagnostic_agent"]`。这里既有真正的 Agent，也有 FunctionHub 展开后的函数名。OxyGent 初始化时会把这些名字加入 datagent 可调用列表，后续 workflow 通过 `await req.call(callee=...)` 调用。

## 2. Master Workflow: `datagent`

`datagent_func_workflow(req)` 的第一步是判断调用方是否已经在 `arguments.datagent_intent` 中直传意图。若直传存在，它会跳过 LLM 意图识别，直接进入对应分支；若不存在，它会复制 `req.get_short_memory()`，把短期记忆里的值全部转成字符串，然后调用 `datagent_intent_agent`，入参是当前 query 和处理后的 short memory。

意图识别返回后，workflow 用 `check_support_intent_and_rewrite()` 做二次校验。当前代码中的支持列表是 `datagent-search-knowledge`、`datagent-query`、`datagent-business-knowledge`、`datagent-page-summary`、`datagent-guess-query`、`datagent-shop-diagnostic`。对于 `datagent-query`，代码还要求 `query_operator` 必须是 `direct`，否则会视为不支持；如果意图结果中有 `rewritten_query`，后续调用 `query_agent` 时会优先使用改写后的问题。

```text
外部请求
    ↓
datagent_func_workflow
    ↓
是否直传 arguments.datagent_intent
    ├─ 是：直接使用该意图
    └─ 否：调用 datagent_intent_agent
          ↓
       check_support_intent_and_rewrite
    ↓
按 intent 调用子能力
```

每个有效分支都会先调用 `agent_scene(req, scene)`。这个函数会把 `req.shared_data["scene"]` 设置为当前场景，并发送一条 `{"type": "scene", "content": scene}` 消息。这个 scene 后续会被历史记录、前端展示和 trace 使用。

| 意图 | 分支行为 | 调用对象 | 入参 | 返回 |
| -- | -- | -- | -- | -- |
| `datagent-search-knowledge` | 指标知识解释 | `search_agent` | `{"query": req.get_query(), "search_intent": "search_rag_agent"}` | 远程 Search Agent 的 RAG 输出 |
| `datagent-query` | 查数 | `query_agent` | `{"query": rewritten_query 或原 query}` | 远程 Query Agent 的查数结果 |
| `datagent-business-knowledge` | 业务知识、文档、负责人、使用路径 | `get_business_knowledge` | `{"query": req.get_query(), "app": req.shared_data.get("app")}` | 文本，且 datagent 会按 2 字符分片流式发送 |
| `datagent-page-summary` | 页面解读 | `ge_ai_page_summary_agent` | `{"query": req.get_query()}` | 远程页面总结输出 |
| `datagent-guess-query` | 猜你想问 | `get_guess_query` | `{}` | `{"data": rsp.output}` |
| `datagent-shop-diagnostic` | 店铺诊断 | `shop_diagnostic_agent` | `{"query": req.get_query()}` | 外部店铺诊断 SSE 的 answer |

代码里还保留了 `datagent-qa` 分支，会返回 `{"data": get_qa()}`，但当前 `support_intents` 列表没有包含 `datagent-qa`。因此在没有直传 `arguments.datagent_intent="datagent-qa"` 的情况下，普通 LLM 意图识别结果即使命中 `datagent-qa`，也会被 `check_support_intent_and_rewrite()` 判为不支持。这是当前实现需要注意的一个不一致点。

`datagent_func_process_output()` 是 master Agent 的输出兜底处理。如果 OxyResponse 状态是 `FAILED`，输出会被替换为“抱歉，系统开了个小差，请稍后重试。”；如果状态是 `CANCELED`，输出会被替换为“抱歉，结果被手动中断，可点击重新生成。”。

## 3. 子 Agent: `datagent_intent_agent`

`datagent_intent_agent` 是 Datagent 最关键的子 Agent，它决定请求路由。它是一个 `ReActAgent`，使用 `datagent_gateway_model`，`max_react_rounds=3`，没有显式 tools。它的 prompt 来自 `modules/agents/datagents/laf_clients.py` 中的 `get_datagent_intent_agent_prompt()`，该函数绑定 LAF key `datagent_intent_agent_prompt`。本地 `modules/agents/datagents/prompts.py` 中有一个较旧的 fallback prompt，但当前 `main.py` 实际绑定的是 LAF getter，不是直接绑定本地 lambda。

当前线上 prompt 已单独整理到 [datagent_intent_prompt.md](datagent_intent_prompt.md)，完整原文见 [datagent_intent_prompt_raw.md](datagent_intent_prompt_raw.md)。这份 prompt 比本地 fallback 更复杂：它新增了 `datagent-ge-analyze`、`datagent-chat-clarification`，并为 `datagent-query` 细分了 `direct`、`calculation`、`filter`、`analysis`、`dimension_value` 等二级算子，还定义了 `need_rewrite` 和 `rewritten_query` 的上下文改写规则。

解析函数是 `datagent_intent_agent_func_parse_llm_response()`。它用 `parse_json()` 解析模型原始输出，并要求 JSON 中至少包含 `has_intent` 和 `intent_type`。只要这两个字段存在，就返回 `LLMState.ANSWER`；否则返回 `LLMState.ERROR_PARSE`，让 ReActAgent 进入重试。这个解析函数没有强制校验 `query_operator` 和 `rewritten_query`，这两个字段是在后续 `check_support_intent_and_rewrite()` 中读取。

| 字段 | 是否由解析函数强制要求 | 后续用途 |
| -- | -- | -- |
| `has_intent` | 是 | 表示是否识别到意图，但当前二次校验主要看 `intent_type` |
| `intent_type` | 是 | 决定 datagent 分支 |
| `query_operator` | 否 | 若 `intent_type=datagent-query`，当前代码必须为 `direct` 才支持 |
| `need_rewrite` | 否 | prompt 要求必填，但当前代码不直接消费，只通过是否有 `rewritten_query` 间接生效 |
| `rewritten_query` | 否 | 若是查数意图，会作为 query_agent 的优先 query |
| `reasoning` 等其他字段 | 否 | 当前 datagent workflow 不消费 |

需要特别注意 prompt 和代码之间的能力差异。Prompt 会把分析洞察类问题归到 `datagent-ge-analyze`，也会把闲聊和非数据问题归到 `datagent-chat-clarification`，但当前 `support_intents` 不包含这两个意图。Prompt 还会为查数输出 `calculation`、`filter`、`dimension_value` 等算子，但当前 `check_support_intent_and_rewrite()` 只允许 `query_operator == "direct"`。因此，这份 prompt 已经具备更细的意图识别能力，而当前 Datagent workflow 只接通了其中一部分。

## 4. 远程子 Agent

Datagent 中的 `search_agent`、`query_agent` 和 `ge_ai_page_summary_agent` 都是 `SSEOxyGent`。它们通过 `modules/utils/remote_agent_func_execute.py` 中的 `remote_agent_func_execute()` 调用远程服务的 `/sse/chat`。调用时会把当前 `OxyRequest` 转成 payload，合并 `arguments`，补充 `caller_category=user`，携带 `LoginContext` 中的登录 headers，并监听远程 SSE 返回。

远程 SSE 消息中，`type=answer` 会更新最终 answer，`type=extra` 会更新 extra，其他普通消息会继续转发给当前请求。对于 `tool_call` 和 `observation`，如果 caller 或 callee 是 user 会过滤，否则会根据是否共享 call stack 决定如何改写调用栈后再转发。异常时会返回 `OxyState.FAILED`，取消时返回 `OxyState.CANCELED`。

| 远程 Agent | Datagent 中的使用场景 | 特殊参数 |
| -- | -- | -- |
| `search_agent` | `datagent-search-knowledge` | 强制传 `search_intent=search_rag_agent`，让 Search Agent 走指标知识 RAG |
| `query_agent` | `datagent-query` | 使用 `rewritten_query` 或原 query |
| `ge_ai_page_summary_agent` | `datagent-page-summary` | 只传当前 query，页面上下文依赖请求 shared_data/group_data 透传 |

## 5. 本地 Tool: `get_business_knowledge`

`get_business_knowledge` 来自 `autobots_knowledge_tools = FunctionHub(name="autobots_knowledge_tools")`，注册函数描述是“autobots业务知识库”。它的函数签名是 `get_business_knowledge(query: str, app: str)`，其中 `query` 是用户问题，`app` 是端标识。workflow 调用时传的是当前 query 和 `req.shared_data.get("app")`。

工具内部先调用 `get_autobots_knowledge(query, LoginContext.erp(), app)`。请求体是一个数组，元素形如 `{"query": query, "erp": erp, "source": source}`，其中 `source` 来自 `config.autobots_knowledge.source_map.get(app, "")`。HTTP 目标是 `config.autobots_knowledge.server_url`。如果 HTTP 状态为 200，工具取 `response_json["data"]["output"]` 作为结果；如果非 200，会抛 `SystemException`。

异常处理是工具内部完成的。`BusinessException` 和 `SystemException` 会转成对应 code 的格式化文案；其他异常会记录日志并返回 `CodeEnum.SYS_5001.get_format_msg()`。因此 Datagent 分支里拿到的一般是字符串，无论成功还是失败都会被按 2 个字符分片发送 stream。

## 6. 本地 Tool: `get_guess_query`

`get_guess_query` 来自 `guess_query_tools = FunctionHub(name="guess_query_tools")`，注册函数描述是“猜你想问”。它没有入参，直接使用 `LoginContext.erp()` 获取当前用户。

工具会先调用 `get_guess_query_from_api(erp)`。该函数请求 `http://virgo.fds.jd.com/query/api-list/`，请求体包含 `requestId`、固定 `appToken`、`apiGroupName=chatbi-llm-index`、`apiName=pre_indicator` 和 `stringSubs.erp`。如果返回 `status=200` 且 `result` 非空，会读取第一条结果中的 `question1`、`question2`、`question3` 组成问题列表。请求失败、返回为空或解析失败时返回 None。

如果 API 没拿到推荐问题，`get_guess_query()` 会回退到 LAF 公共配置 `get_guess_default()`。最后它会过滤掉包含 `{dept}` 的问题，避免返回需要动态部门替换但当前没有替换上下文的问题。Datagent 对该工具的最终包装是 `{"data": rsp.output}`。

## 7. Tool 和子 Agent 的 Curl 调试命令

下面的 curl 用于你在本机或内网环境中快速观察返回结构。涉及登录态的请求需要补充有效 Cookie 或 SSO headers；涉及 ERP 的地方请替换为自己的 ERP。为了避免把密钥固化到文档中，示例里的 token、Cookie、appToken 都用环境变量占位。

### 7.0 直接调试意图识别 Agent

如果想直接查看 `datagent_intent_agent` 的 JSON 分类结果，可以在本地服务里显式指定 `callee=datagent_intent_agent`。`short_memory` 用字符串传入，形态与 master workflow 调用意图 Agent 时保持一致；如果要模拟多轮上下文，可以把最近历史问题按字符串形式放进去。

```bash
curl 'http://local.jd.com:8085/api/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "callee": "datagent_intent_agent",
    "query": "成交金额",
    "short_memory": "[]",
    "shared_data": {
      "app": "ge"
    }
  }'
```

如果不指定 `callee` 且不直传 `arguments.datagent_intent`，请求会走真实 Datagent master 链路：先由 `datagent_intent_agent` 识别，再由 workflow 路由到对应子 Agent 或 tool。这个命令适合验证 prompt 改写和路由整体是否符合预期，但返回内容是最终业务结果，不一定直接包含意图 JSON。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "成交金额",
    "shared_data": {
      "app": "ge"
    }
  }'
```

### 7.1 通过 Datagent 强制调用业务知识 Tool

这个命令不直接打 Autobots 底层服务，而是调用本地 Datagent 服务，并通过 `datagent_intent` 跳过意图识别，强制进入 `get_business_knowledge` 分支。它最接近真实线上链路，因为会走 LoginContext、scene 消息和 Datagent 的流式输出逻辑。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "交易概况负责人是谁",
    "arguments": {
      "datagent_intent": "datagent-business-knowledge"
    },
    "shared_data": {
      "app": "ge"
    }
  }'
```

如果你想直接看 Autobots 知识库底层接口的原始返回，可以用下面的命令。`source` 需要按 `config.autobots_knowledge.source_map` 映射，例如 `ge` 和 `ge_me` 当前映射到 `chatbi_ge`。

```bash
curl 'http://g.jsf.jd.local/com.jd.bpp.chatbi.index.api.service.IChatbiKnowledgeService/online/autoBotsService/1377191/jsf/120000' \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "query": "交易概况负责人是谁",
      "erp": "'"${ERP}"'",
      "source": "chatbi_ge"
    }
  ]'
```

### 7.2 通过 Datagent 强制调用猜你想问 Tool

这个命令强制进入 `get_guess_query` 分支。Datagent 返回的最终结构是 `{"data": [...]}`，中间是否命中个性化 API 或 LAF 默认配置，需要看服务日志或直接调用底层 API。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "猜你想问",
    "arguments": {
      "datagent_intent": "datagent-guess-query"
    },
    "shared_data": {
      "app": "ge"
    }
  }'
```

底层个性化推荐 API 的请求示例如下。代码中 `apiGroupName` 固定为 `chatbi-llm-index`，`apiName` 固定为 `pre_indicator`。

```bash
curl 'http://virgo.fds.jd.com/query/api-list/' \
  -H 'Content-Type: application/json' \
  -d '{
    "requestId": "'"$(uuidgen)"'",
    "appToken": "'"${VIRGO_APP_TOKEN}"'",
    "apiGroupName": "chatbi-llm-index",
    "apiName": "pre_indicator",
    "stringSubs": {
      "erp": "'"${ERP}"'"
    }
  }'
```

### 7.3 强制调用指标知识远程 Agent

这个命令通过 Datagent 进入 `datagent-search-knowledge` 分支，Datagent 会调用远程 `search_agent`，并自动补 `search_intent=search_rag_agent`。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "成交金额的口径是什么",
    "arguments": {
      "datagent_intent": "datagent-search-knowledge"
    },
    "shared_data": {
      "app": "ge"
    }
  }'
```

如果要绕过 Datagent 直接看 Search Agent 的 SSE 返回，可以调用远程 Search 服务。这个请求需要 Search Agent 服务可访问，并且 headers 中有有效登录态。

```bash
curl -N 'http://datagent-search.jd.com/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "成交金额的口径是什么",
    "search_intent": "search_rag_agent",
    "shared_data": {
      "app": "ge"
    }
  }'
```

### 7.4 强制调用查数远程 Agent

这个命令通过 Datagent 进入 `datagent-query` 分支。若你要测试 prompt 改写后的效果，可以把 query 直接写成完整问题；若要测试 `datagent_intent_agent` 的改写逻辑，则不要传 `datagent_intent`，让它先过意图识别。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "昨天成交金额是多少",
    "arguments": {
      "datagent_intent": "datagent-query"
    },
    "shared_data": {
      "app": "ge"
    }
  }'
```

直接调用远程 Query Agent 的示例：

```bash
curl -N 'http://datagent-query.jd.com/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "昨天成交金额是多少",
    "shared_data": {
      "app": "ge"
    }
  }'
```

### 7.5 强制调用页面总结远程 Agent

页面总结依赖页面侧传入的 `shared_data`，只传 query 通常不足以得到真实页面解读。下面的命令展示请求形态，实际调试时需要把 `configs.menu_id`、`date`、`filters`、`modules` 替换为页面真实数据。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "解读一下这个页面",
    "arguments": {
      "datagent_intent": "datagent-page-summary"
    },
    "shared_data": {
      "app": "ge",
      "scene": "your_scene",
      "configs": {
        "menu_id": "your_menu_id"
      },
      "date": {
        "startDate": "2026-06-01",
        "endDate": "2026-06-30"
      },
      "filters": [],
      "modules": []
    }
  }'
```

### 7.6 强制调用店铺诊断 Agent

这个命令通过 Datagent 进入店铺诊断分支。若 LAF 熔断开启，会直接返回熔断文案；否则会转发到 `config.agents.shop_diagnostic_agent.server_url`。

```bash
curl -N 'http://local.jd.com:8085/sse/chat' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "店铺ID888888店铺星级：为我诊断下店铺星级",
    "arguments": {
      "datagent_intent": "datagent-shop-diagnostic"
    },
    "shared_data": {
      "app": "ge"
    }
  }'
```

底层店铺诊断服务的直接调用形态如下。实际 payload 字段以 Datagent 转发的 `OxyRequest` 展开结构为准。

```bash
curl -N 'http://diagnostic-assistant.jd.com/api/star/standard_query' \
  -H 'Accept: text/event-stream' \
  -H 'Content-Type: application/json' \
  -H "Cookie: ${JD_COOKIE}" \
  -d '{
    "query": "店铺ID888888店铺星级：为我诊断下店铺星级",
    "caller_category": "user",
    "shared_data": {
      "app": "ge"
    }
  }'
```

## 8. 子 Agent: `shop_diagnostic_agent`

`shop_diagnostic_agent` 是本地 `WorkflowAgent`，但它并不自己做诊断推理，而是把请求转发到 `config.agents.shop_diagnostic_agent.server_url`。workflow 会把当前 `OxyRequest` dump 成 payload，排除 `mas`、`parallel_id`、`latest_node_ids`，再把 `arguments` 平铺到 payload 顶层，并设置 `caller_category=user`。

转发前会先检查 LAF 开关 `get_datagent_shop_diagnostic_fuse()`。如果开关开启，workflow 读取 `get_datagent_shop_diagnostic_fuse_msg()`，发送一条 stream 消息并直接返回该文案。若未熔断，它会携带 `Accept: text/event-stream`、`Content-Type: application/json` 和登录态 headers，用 aiohttp POST 到外部 SSE 服务。SSE 中 `data: done` 表示结束；消息类型为 `answer` 时更新最终 answer，其他消息会原样通过 `oxy_request.send_message(data)` 转发给前端。

## 9. LAF 与配置来源

Datagent 的动态配置分成 common 和 datagent 两个 LAF client。`common_client` 使用 `config.agents.datagent.common_laf`，`datagent_client` 使用 `config.agents.datagent.laf`。公共配置目前包含常见问题和默认猜你想问，Datagent 专属配置包含意图 prompt、店铺诊断熔断、熔断文案和不支持意图回复。

| Getter | LAF key | 用途 | 返回处理 |
| -- | -- | -- | -- |
| `get_qa` | `qa` | 常见问题数据 | `json.loads(val)` |
| `get_guess_default` | `guess_default` | 猜你想问兜底问题 | `json.loads(val)` |
| `get_datagent_intent_agent_prompt` | `datagent_intent_agent_prompt` | 意图识别 prompt | 原始字符串 |
| `get_datagent_shop_diagnostic_fuse` | `datagent_shop_diagnostic_fuse` | 店铺诊断熔断开关 | 字符串 `"true"` 转 True |
| `get_datagent_shop_diagnostic_fuse_msg` | `datagent_shop_diagnostic_fuse_msg` | 店铺诊断熔断文案 | 空值时回退限流文案 |
| `get_datagent_intent_reply` | `datagent_intent_reply` | 不支持意图的兜底回复 | 原始字符串 |

配置文件中，`application-datagent-*.yml` 负责把 `agents.agent_path` 指到 `modules.agents.datagents.main`，并配置历史回显范围。`application.yml` 中的 `agents.datagent` 配置远程 Agent URL、模型、common_laf、laf、joy_builder_model 和 gateway_model；`autobots_knowledge` 配置业务知识库服务地址和 app 到 source 的映射。

## 10. 端到端示例

用户问“昨天家电成交金额是多少”。请求进入 Datagent 后，没有直传意图时会先调用 `datagent_intent_agent`。理想情况下模型输出 `intent_type=datagent-query`，`query_operator=direct`，并可能给出 `rewritten_query`。二次校验通过后，Datagent 发送 scene `datagent-query`，调用远程 `query_agent`。如果有改写 query，就传改写 query；否则传原问题。最终 Query Agent 的输出直接作为 Datagent 输出。

用户问“成交金额这个指标是什么意思”。意图应落到 `datagent-search-knowledge`。Datagent 会发送 scene `datagent-search-knowledge`，调用远程 `search_agent`，并传 `search_intent=search_rag_agent`。这个参数很关键，它让 Search Agent 不走结构化找数，而是走指标知识召回和总结。

用户问“交易概况负责人是谁”或“这个功能怎么用”。意图应落到 `datagent-business-knowledge`。Datagent 调用本地 `get_business_knowledge`，工具按 app 映射 source，并带当前 ERP 请求 Autobots 业务知识库。拿到文本后，Datagent 会每 2 个字符发送一次 stream 消息，最后返回完整文本。

用户在页面中触发“页面 AI 智能解读”。意图落到 `datagent-page-summary` 后，Datagent 调用远程 `ge_ai_page_summary_agent`。Datagent 自己只传 query，页面总结真正需要的 app、scene、menu_id、日期、筛选器、模块数据等上下文依赖请求的 `shared_data` 和 `group_data` 透传给远程服务。

用户触发“猜你想问”。如果意图来自直传或生产 prompt 支持 `datagent-guess-query`，Datagent 调用 `get_guess_query`。工具优先取个性化 API 的 question1 到 question3，失败时回退 LAF 默认配置，最终返回 `{"data": [...]}`。

用户请求“店铺诊断”。Datagent 调用 `shop_diagnostic_agent`。如果熔断开关开启，立即返回熔断文案；否则把当前请求转发到外部店铺诊断 SSE 服务，并把中间消息继续转发给前端，最终返回 SSE 中最后的 answer。

## 11. 当前实现需要注意的点

`datagent_intent_agent` 的本地 fallback prompt 只详细描述了查数、找数、知识和页面解读四类主管，但 workflow 的支持列表还包括猜你想问和店铺诊断。因此实际线上分类能力依赖 LAF 中的 `datagent_intent_agent_prompt` 是否覆盖这些意图。

`datagent-qa` 分支存在于 workflow 中，但不在 `support_intents` 中。除非调用方直传 `arguments.datagent_intent=datagent-qa`，否则普通意图识别路径不会进入该分支。

`datagent-query` 对 `query_operator` 有额外限制，只有 `direct` 被接受。如果 prompt 没有稳定输出 `query_operator`，或者输出空值，查数意图会被判为不支持并返回 `get_datagent_intent_reply()`。这意味着 prompt 和代码字段约定必须保持一致。

三个远程 Agent 都依赖对应服务的 `/sse/chat` 可用性和登录态 headers。Datagent 本地只做 SSE 转发和消息透传，不保证远程服务内部成功。远程失败时，`remote_agent_func_execute()` 会返回 `OxyState.FAILED`，随后 master 的 `datagent_func_process_output()` 会把失败输出统一替换为系统兜底文案。
