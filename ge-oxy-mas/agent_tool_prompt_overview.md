# Agent / Tool / Prompt 文档索引

## 0. 普通 `/chat` 主请求：并发许可 + 超限排队

普通 `/chat` 主请求不是一进来就直接调用 Agent，也不是所有请求都先进入等待队列。当前设计是“先尝试获取并发许可，未超限就立即执行，超限才写入 Redis/JimDB 队列”。也就是说，队列在这里承担的是并发保护和削峰作用，而不是所有请求的统一入口。代码入口在 `webs/chat_router.py` 的 `/chat` 路由，路由层先做请求校验、文件下载、用户信息补充，然后创建后台任务调用 `services/chat_service.py::chat_with_agent`；真正决定“立即执行还是入队”的逻辑在 `QueueLimiter.submit_task(...)`。

```text
POST /chat
  ↓
webs/chat_router.py::chat
  - 校验 query / shared_data / group_data
  - 下载 file 类型数据到 workspace
  - 补充用户部门信息
  - 创建后台任务 chat_with_agent(...)
  - 立即返回 ChatResponse(trace_id)
  ↓
services/chat_service.py::chat_with_agent
  - 根据 app / scene / type 计算 queue_name
  - QueueLimiter.submit_task(trace_id, payload)
      ├─ running_count < limit → 获得执行许可，立即 mas.chat_with_agent(...)
      └─ running_count >= limit → 写入 queue，等待 /chat/trigger 调度
```

队列名由请求里的 `shared_data.app`、`shared_data.scene`、`shared_data.type` 拼出来，格式是 `q_{env}_{app}_{scene}_{type}`。例如当前环境是 `pre`，请求是 `app=ge`、`scene=c`、`type=2`，队列名就是 `q_pre_ge_c_2`。每个队列的并发上限来自 DUCC/LAF 的 `chat_queue_limit`；如果某个队列没有单独配置，就使用 `chat_queue_default_limit`。运行中的任务和等待中的任务存在 Redis/JimDB 的两个 key 里：`limit:{queue_name}` 记录正在运行的 trace，`queue:{queue_name}` 记录排队等待的 trace，排队顺序按入队时间排序。

`QueueLimiter.submit_task(...)` 是一个 Redis Lua 原子操作。它会先清理超时的 running 任务，再数 `limit:{queue_name}` 里当前运行任务数。如果当前运行数小于 limit，就把当前 `trace_id` 加入 running 集合并返回 `acq=True`，随后代码会进入 `mas.chat_with_agent(...)`，也就是正式调用 `data_agent` 以及后续 `master_agent` 等 Agent。如果当前运行数已经达到 limit，就把当前 `trace_id` 加入 `queue:{queue_name}`，并把完整 payload 存到 `queue:{queue_name}:task_data:{trace_id}`，返回 `acq=False`。此时本次后台任务不会调用 Agent，只会写一条 trace 占位记录，等待后续调度。

举一个具体例子：假设 `q_pre_ge_c_2` 的并发上限是 2。A 请求进来时 running=0，A 获得许可并立即执行；B 请求进来时 running=1，B 也立即执行；C 请求进来时 running=2，已经达到上限，所以 C 被写入 `queue:{q_pre_ge_c_2}`，前端拿到的仍然是 C 的 `trace_id`，但 C 的 Agent 流程还没有开始。之后某个运行任务结束时，代码会释放 running 许可；真正把 C 从队列里取出来执行的是 `/chat/trigger`，它会调用 `chat_with_agent_from_queue(...)`，内部执行 `QueueLimiter.schedule_next()`，从队列头取出 C 的 payload，再调用 `mas.chat_with_agent(...)`。

```text
limit = 2

T1: A 进入 /chat → running 0/2 → A 直接执行
T2: B 进入 /chat → running 1/2 → B 直接执行
T3: C 进入 /chat → running 2/2 → C 入队，等待 trigger
T4: A 完成 → release A，running 1/2
T5: /chat/trigger → schedule_next 取出 C → C 加入 running 并执行 Agent
```

调度触发有两个入口：一是外部或调度服务请求 `/chat/trigger`，二是 `main.py` 中的 scheduler 在 `config.web.scheduler` 开启时定期请求配置里的 trigger server。这个设计意味着任务完成时只释放许可，不在同一个函数里递归拉起下一个任务；拉起下一个任务依赖 trigger。队列状态和清理接口也在 `webs/chat_router.py` 里，包括 `/chat/queue/status/{queue_name}`、`/chat/queue/clear_all/...` 和 `/chat/queue/clear_top/...`。

需要注意两个边界：`restart_node_id` 的重启请求会走 `handle_restart_chat(...)`，直接调用 `mas.chat_with_agent(...)`，不进入这个队列；`/chat/summary`、`/chat/title`、`/data/query_router`、`/knowledge/upload` 等描述侧或数据侧接口也不是这个 `/chat` 主队列链路。本文档后面的 Agent 说明默认讨论的是普通 `/chat` 主请求在获得执行许可之后进入 `data_agent` 的流程。

本文档只作为入口索引，避免把所有 Agent、Tool、Prompt 的说明耦合在一个长文档里。当前项目的默认服务入口是 `main.py`，启动后会创建两个 MAS：主对话链路 `data_oxy_space` 和描述/推荐链路 `desc_oxy_space`。普通 `/chat` 请求默认进入 `data_agent`，由它按请求类型分流到 `sql_proxy_agent`、`bi_proxy_agent` 或 `master_agent`；描述、摘要、标题、推荐和网页搜索类接口则通过 `desc_mas` 直接调用描述侧 Agent。

如果只想理解入口关系，先看下面的“主 Agent 文档”。如果要看工具参数和调用样例，再看 [agent_tool_prompt_examples.md](./agent_tool_prompt_examples.md)。DUCC/LAF 当前实际读取到的运行时配置单独整理在 [prompt文档.md](./prompt文档.md)，其中包含当前 LAF profile、被代码 provider 读取的 key、各 key 的实际内容、远端存在但未发现直接使用的 key，以及代码有 provider 但当前 DUCC 未覆盖而回退本地默认值的配置。

## 主 Agent 文档

| 主 Agent / 服务 | 文档 | 覆盖内容 |
| --- | --- | --- |
| `data_agent` | [agents/data-agent.md](./agents/data-agent.md) | 主 MAS 第一层入口，说明 `/chat` 如何按 quick data、BI、普通分析分流，并列出它直接下挂的子 Agent。 |
| `master_agent` | [agents/master-agent.md](./agents/master-agent.md) | 常规多步骤 chat/report 主链路，覆盖 `data_planner`、`data_executer`、`process_expert`、`Text取数Agent`、节点总结和全局总结；planner 前预处理见 [agents/master-preprocess-flow.md](./agents/master-preprocess-flow.md)，关键编排 Agent 另有独立文档。 |
| `desc_oxy_space` | [agents/desc-agent.md](./agents/desc-agent.md) | 描述侧 Agent 组，覆盖 `desc_agent`、推荐 Agent、会话摘要、标题、网页搜索和网页内容总结。 |
| `sql_proxy_agent` | [agents/sql-proxy-agent.md](./agents/sql-proxy-agent.md) | 快速查数代理，说明 SQLAgent 提交接口、SSE trace 消费和 prompt/tool 关系。 |
| `bi_proxy_agent` | [agents/bi-proxy-agent.md](./agents/bi-proxy-agent.md) | BI 看板代理，说明大纲确认、`bi_agent` 远端调用和输出 dashboard 的流程。 |
| `notice_agent` | [agents/notice-agent.md](./agents/notice-agent.md) | 京Me 通知链路，覆盖预通知、结果通知和 `msg_summary_agent`。 |
| `enhance_agent_flow` | [agents/enhance-agent.md](./agents/enhance-agent.md) | 提示词增强链路，覆盖 `enhance_agent` 与占位符安全约束。 |
| `summary_agent_flow` | [agents/chat-summarize-agent.md](./agents/chat-summarize-agent.md) | 独立 AI 总结块链路，覆盖 `summary_agent` 的输入结构与 prompt；不是主流程最终全局总结。 |
| `/data/query_router` | [agents/query-router.md](./agents/query-router.md) | 查询路由服务；它不是 Oxy Agent，但使用 `desc_mas.default_llm` 和独立 prompt，需要单独理解。 |

## 关键编排 Agent 文档

这些 Agent 仍然属于 `master_agent` 子树，但它们承担跨步骤或步骤内的关键边界，单独成文档更便于阅读和维护。

| Agent | 文档 | 主要关注点 |
| --- | --- | --- |
| `master_agent` 预处理 | [agents/master-preprocess-flow.md](./agents/master-preprocess-flow.md) | `add_data_desc`、`pre_process_clarifier`、`get_input_mode` 的实现、示例和外部 curl 查询模板。 |
| 主流程 Agent 类型逻辑 | [agents/main-flow-agent-request-logic.md](./agents/main-flow-agent-request-logic.md) | 横向说明主流程中 `WorkflowAgent`、`DataGraphPlanner`、`DataExecuter`、`ProgressiveReActAgent`、总结类和判断类 Agent 的预设请求处理逻辑。 |
| `data_planner` | [agents/data-planner-flow.md](./agents/data-planner-flow.md) | 解释它如何接收预处理后的 query、数据概况、知识和历史，如何通过 `search_knowledge` / `make_graph_plan` 协议生成图计划，并如何交给下游解析成 `GraphPlan`。 |
| `data_executer` | [agents/data-executer-flow.md](./agents/data-executer-flow.md) | 解释它如何接收 planner 生成的 `GraphPlan`，按依赖调度步骤，并把 `current_step_index`、附件和状态传给下游/合并回来。 |
| `process_and_summary_agent` | [agents/process-and-summary-agent.md](./agents/process-and-summary-agent.md) | 单个计划步骤如何从 `process_expert` 执行到 `summary_expert` 输出，并写入步骤附件。 |
| `global_summary_agent` | [agents/global-summary-agent.md](./agents/global-summary-agent.md) | 多步骤 `summary_expert` 结果如何聚合为最终 Markdown 或 HTML 输出。 |
| 主链路面试题 | [agents/main-flow-interview-questions.md](./agents/main-flow-interview-questions.md) | 从主链路结构、Agent 特点、planner/executer、成功失败处理、工具 prompt、总结输出和工程化可靠性整理问题与参考答案。 |

## 顶层执行关系

主链路可以理解为一棵入口树：`data_agent` 是默认入口，`master_agent` 是普通数据分析的核心树，`desc_oxy_space` 是一组由数据接口直接调用的辅助 Agent。`sql_proxy_agent`、`bi_proxy_agent`、`notice_agent`、`enhance_agent_flow`、`summary_agent_flow` 虽然也注册在主 MAS 中，但它们各自服务特定接口或特定场景，不应全部塞进 `master_agent` 的说明里。

```text
main.py
  ├─ MAS(data_oxy_space)
  │   ├─ data_agent
  │   │   ├─ sql_proxy_agent
  │   │   ├─ bi_proxy_agent
  │   │   └─ master_agent
  │   ├─ notice_agent
  │   ├─ enhance_agent_flow
  │   └─ summary_agent_flow
  └─ MAS(desc_oxy_space)
      ├─ desc_agent
      ├─ report_recommend_agent / bi_recommend_agent
      ├─ chat_summary_agent / chat_title_agent
      └─ web_search_agent / web_content_summary_agent
```

## 维护约定

新增或修改 Agent 文档时，优先按“主要 Agent 和自己的子 Agent 放在同一文档”的规则整理。某个子 Agent 如果只是服务一个主 Agent，通常放到主 Agent 文档里；但如果它承担步骤执行、跨步骤聚合、输出交付这类关键边界，或者说明内容会让主文档过重，就应单独拆文档，并在主文档里保留引用入口。Tool 和 Prompt 不再集中列成长表，而是写到它实际服务的 Agent 文档中，这样从索引进入时能直接看到该 Agent 的子树、工具和 prompt 来源。
