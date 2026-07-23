# master_agent

`master_agent` 是普通数据分析请求的核心入口，定义在 `applications/master_agent/run_master_0130.py`，主 workflow 是 `applications/master_agent/agent/master_graph_flow.py` 中的 `super_master_workflow`。本文只整理常规多步骤 chat/report 主链路；查数快捷流、单步快捷流、报告解读、探索、QA、占位符和 direct 执行等特殊分支不放在本文件中。

## 主执行流

普通分析请求由 `data_agent` 进入 `master_agent` 后，先通过 `add_data_desc` 补齐数据描述，再通过 `pre_process_clarifier` 生成 `all_data_summary`、`day_info`、默认知识和搜索信息，并通过 `get_input_mode` 判断 chat/report/HITL/direct 等入口模式。这三步的实现细节和可用于 Postman 查询的外部 curl 已拆到 [master-preprocess-flow.md](./master-preprocess-flow.md)。常规多步骤 chat/report 主链路会调用 `data_planner` 生成图计划，再由 `data_executer` 按计划调度执行步骤，每个步骤落到 `process_and_summary_agent`，最后由 `global_summary_agent` 生成聊天结论或 HTML 报告。

如果要从 Agent 类型角度理解这些节点底层“收到请求后默认怎么处理”，见 [main-flow-agent-request-logic.md](./main-flow-agent-request-logic.md)。该文档横向说明 `WorkflowAgent`、`DataGraphPlanner`、`DataExecuter`、`ProgressiveReActAgent`、总结类和判断类 Agent 的预设处理逻辑。

```text
master_agent
  ↓
add_data_desc / pre_process_clarifier / get_input_mode
  ↓
data_planner
  ↓
data_executer
  ↓
process_and_summary_agent
  ├─ process_expert → Text取数Agent → 业务工具
  └─ summary_expert → node_summary_agent / visual_agent
  ↓
global_summary_agent
  ├─ global_summary_agent_chat
  └─ global_summary_agent_report
```

## 子 Agent 结构

| 子 Agent | 类型/位置 | 功能 | Tool / Prompt |
| --- | --- | --- | --- |
| `data_planner` | `DataGraphPlanner`，`applications/master_agent/agent/graph_planner.py` | 根据用户问题、数据概况、历史和知识生成图结构计划。 | Tool：`planner_tools`；Prompt：`get_planner_prompt`、`get_research_mode_planner_prompt`、`get_report_mode_planner_prompt`。 |
| `data_executer` | `DataExecuter`，`applications/master_agent/agent/graph_executor.py` | 按已有 `GraphPlan.dependencies` 调度步骤，支持并发、重跑和状态更新；它不生成图结构，也不直接做业务取数/加工。 | Tool：`executor_tools.update_plan`；Prompt：`get_executor_prompt`。 |
| `process_and_summary_agent` | `WorkflowAgent` | 单步骤执行编排，先处理数据，再生成节点结论和可视化。 | 子 Agent：`process_expert`、`summary_expert`；无直接 prompt。 |
| `process_expert` | `WorkflowAgent` | 构建数据源、知识、workspace 和工具上下文，调用 `Text取数Agent`。 | 子 Agent：`Text取数Agent`、`metric_route_judge_agent`；动态 prompt 由 `data_process_workflow` 构造。 |
| `Text取数Agent` | `ProgressiveReActAgent` | ReAct 取数与加工核心，按动态 prompt 调用业务工具并输出总结协议。 | Tools：`text2data_tool`、`py_coder_tool`、`web_search_tool`、`metric_analyze_tool`、`judge_tool`；Prompt：`build_multi_data_prompt`、`memory_prompt` 和工具描述 provider。 |
| `metric_route_judge_agent` | `ChatProAgent` | 判断 query 是否优先走指标结构化分析。 | Prompt：`get_metric_route_judge_prompt`。 |
| `node_judge_agent` | `ChatAgent` | 节点级合规判断，服务 `Text取数Agent` 的自评链路。 | Tool：`judge_tool`；Prompt：`build_node_judge_prompt_from_laf()`。 |
| `summary_expert` | `WorkflowAgent` | 节点总结与可视化编排。 | 子 Agent：`node_summary_agent`、`visual_agent`；无直接 prompt。 |
| `node_summary_agent` | `WorkflowAgent` | 当前实现从步骤附件读取 `process_expert` 结论并修复 Markdown 表格。 | 当前不使用 `NODE_SUMMARY_PROMPT` 做 LLM 总结。 |
| `visual_agent` | `WorkflowAgent` | 节点可视化工作流入口。 | 逻辑在 `node_visual_workflow`。 |
| `global_summary_agent` | `WorkflowAgent` | 根据 chat/report 模式选择全局总结 Agent。 | 子 Agent：`global_summary_agent_chat`、`global_summary_agent_report`。 |
| `global_summary_agent_chat` | `BackupChatAgent` | 聊天/Markdown 风格全局总结。 | Prompt：`get_global_summary_prompt_md`。 |
| `global_summary_agent_report` | `GlobalSummaryAgent` | HTML 报告风格全局总结，报告模式会追加数据来源。 | Prompt：`get_global_summary_prompt_html`。 |

## Planner / Executer / 执行链路边界

`data_planner` 负责把用户问题转成图结构计划，图的形状来自 `make_graph_plan` 参数中的 `steps[*].dependencies`，并被保存为 `GraphPlan.id`、`title`、`step_command`、`dependencies` 等字段。它如何接收预处理上下文、调用知识工具、产出 `make_graph_plan`，详见 [data-planner-flow.md](./data-planner-flow.md)。`data_executer` 不负责“把节点编排成图”，它拿到的是已经成形的 `GraphPlan`，主要工作是根据依赖关系判断哪些节点可以启动、控制最大并发、为依赖节点注入上游 memory、调用 `process_and_summary_agent` 执行当前节点，并通过 `update_plan` 更新步骤状态。它如何处理上游 planner 内容、如何把当前步骤传给下游，详见 [data-executer-flow.md](./data-executer-flow.md)。真正的数据取数、加工、代码执行和结论产出不在 `data_executer` 内完成，而是在 `process_and_summary_agent → process_expert → Text取数Agent / tools → summary_expert` 这条链路里完成。

## 计划确认与自动执行

`/chat` 请求里如果有前端语义上的 `autoExecute`，在当前项目后端实际对应的是 `shared_data.report_pref.outline`，schema 定义在 `schemas/chat_schema.py::ReportPref`，取值为 `auto` 或 `self`。代码没有直接读取名为 `autoExecute` 的字段；后续流程只通过 `master_tool.get_outline_mode(oxy_request)` 读取 `report_pref.outline`，默认值是 `auto`。

在 `report_workflow` 中，`data_planner` 先生成 `GraphPlan`，系统总会把计划以 `todolist` 卡片发给前端，卡片内容是 `{"title": [...], "command": [...]}`，`message_id` 固定为 `{plan_id}_todolist`。如果 `outline=auto`，后端不会等待用户修改，发送计划卡片后立即进入 `data_executer` 执行。也就是说，`auto` 模式下用户能看到计划，但这只是展示，不是确认点。

如果 `outline=self`，后端会在发送 `todolist` 卡片后调用 `reflexion_plan_when_report`，再通过 `get_checked_todolist → poll_get_chat_append_message` 每 3 秒轮询一次 `chat_append_message_{message_id}`。前端需要调用 `/chat/append`，把用户修改后的计划作为 JSON 数组写入 `message_content`，格式类似：

```json
[
  {"title": "查询销售趋势", "command": "查询近30天销售额，按日期聚合"},
  {"title": "分析异常日期", "command": "找出销售额明显波动的日期并说明原因"}
]
```

后端拿到这份内容后，不会再调用 planner 重写计划，而是直接用用户传回的 `title` 和 `command` 重建 `GraphPlan`，并覆盖 `shared_data["plan"]` 后继续执行。等待时长来自 DUCC 配置 `outline_confirm_timeout`；如果超时没有收到 `/chat/append` 内容，流程返回“用户超时未确认大纲，任务提前终止”。

这个人工确认分支只在报告和 HITL 占位符计划这类主流程里生效：`type=2` 的 `report_workflow` 会检查 `outline=self`，`type=4 && hitl_subtype=1` 的 `plan_placeholder_flow` 也会检查。`type=1` 的普通 `chat_workflow` 虽然也会发送计划卡片，但代码没有等待用户修改；`outline_launchable` 和 `direct_execute` 这类已有可执行计划的分支也会直接执行，不走这段人工确认逻辑。

## 关键编排 Agent 文档

`process_and_summary_agent` 和 `global_summary_agent` 是 `master_agent` 子树里的两个关键边界：前者负责单步骤执行与节点结论沉淀，后者负责跨步骤聚合和最终交付格式。为了避免 `master_agent` 文档继续内聚过重，它们的详细执行流、状态读写、子 Agent、tool 和 prompt 对应关系拆到独立文档中。

| Agent | 独立文档 | 说明 |
| --- | --- | --- |
| `add_data_desc` / `pre_process_clarifier` / `get_input_mode` | [master-preprocess-flow.md](./master-preprocess-flow.md) | 解释 planner 前的预处理、知识召回、数据源元信息补全和模式分流，并给出实际外部请求的 curl 模板。 |
| 主流程 Agent 类型逻辑 | [main-flow-agent-request-logic.md](./main-flow-agent-request-logic.md) | 从 Agent 类型维度解释主流程中各类 Agent 的底层请求处理方式。 |
| `data_planner` | [data-planner-flow.md](./data-planner-flow.md) | 解释预处理上下文如何进入 planner，planner 如何使用 `search_knowledge` 和 `make_graph_plan` 协议生成图计划，以及下游如何解析。 |
| `data_executer` | [data-executer-flow.md](./data-executer-flow.md) | 解释 planner 输出如何变成 `GraphPlan`，`data_executer` 如何按依赖调度，并如何把当前步骤传给 `process_and_summary_agent`。 |
| `process_and_summary_agent` | [process-and-summary-agent.md](./process-and-summary-agent.md) | 解释 `data_executer` 如何调用它执行单个图计划步骤，以及它如何串联 `process_expert`、`summary_expert`、`node_summary_agent` 和 `visual_agent`。 |
| `global_summary_agent` | [global-summary-agent.md](./global-summary-agent.md) | 解释它如何收集每个步骤的 `summary_expert` 结果，按 chat/report 模式分派到最终 Markdown 或 HTML 总结链路。 |

## Tool 详解

`master_agent` 直接声明了 `oss_tools` 和 `planner_tools`，但大多数工具并不是由它直接调用，而是在子 Agent 中使用。`planner_tools.make_graph_plan` 和 `planner_tools.search_knowledge` 服务 `data_planner`；`executor_tools.update_plan` 服务 `data_executer`；`text2data_tool`、`py_coder_tool`、`web_search_tool`、`metric_analyze_tool`、`judge_tool` 服务 `Text取数Agent`；`oss_tools.save_as_file_to_oss` 主要在报告输出阶段保存 HTML/TXT 到 OSS。

| Tool | 实现位置 | 使用 Agent | 说明 |
| --- | --- | --- | --- |
| `make_graph_plan` | `applications/master_agent/tools/new_master_tool.py` | `data_planner` | 提交图结构计划的协议工具，函数本身不执行计划。 |
| `search_knowledge` | 同上 | `data_planner` | 检索系统知识、用户知识和语义层知识。 |
| `update_plan` | `applications/master_agent/tools/new_master_tool.py` | `data_executer` | 更新当前步骤状态和执行摘要。 |
| `text2data_tool` | `applications/data_process/tools/text2data_tools.py` | `Text取数Agent` | 调用远端 text2data 服务取数，保存完整 CSV 到 workspace。 |
| `py_coder_tool` | `applications/data_process/tools/python_tools.py` | `Text取数Agent` | 执行 Python 做本地文件处理、临时数据加工、多源合并和复杂计算。 |
| `metric_analyze_tool` | `applications/data_process/tools/metric_analyze_tools.py` | `Text取数Agent` | 指标白名单结构化分析，失败后回退取数链路。 |
| `web_search_tool` | `applications/data_process/tools/web_search_tools.py` | `Text取数Agent` | master 链路联网搜索和摘要。 |
| `judge_tool` | `applications/data_process/tools/judge_tools.py` | `Text取数Agent`、`node_judge_agent` | 基于 rubric 评估步骤推理/结果。 |
| `save_as_file_to_oss` / `save_data_frame_to_oss` | `extends/oss/oss_function_hub.py` | `master_agent` 输出阶段 | 把报告文本或 DataFrame 上传 OSS。 |

## Prompt 来源

`master_agent` 子树的 prompt 大多通过 `extends/ducc/laf_instance.py` 读取，远端配置为空时回退本地常量。排查线上行为时，应先定位 provider 函数，再确认 LAF/DUCC 实际配置。

当前项目读取到的 DUCC/LAF 配置快照见 [../prompt文档.md](../prompt文档.md)，其中覆盖了 planner、executor、global summary、node judge、tool description 等与 `master_agent` 子树相关的 key。

| Prompt provider / 常量 | 对应 Agent | 默认文件 |
| --- | --- | --- |
| `get_planner_prompt` | `data_planner` | `applications/master_agent/prompt/graph_plan_prompt.py` |
| `get_research_mode_planner_prompt` | `data_planner` | `applications/master_agent/prompt/graph_plan_prompt.py` |
| `get_report_mode_planner_prompt` | `data_planner` | `applications/master_agent/prompt/graph_plan_prompt.py` |
| `get_executor_prompt` | `data_executer` | `applications/master_agent/prompt/base_prompt.py` |
| `build_multi_data_prompt` | `Text取数Agent` | `applications/data_process/prompts/multi_data_prompt_builder.py` |
| `memory_prompt` | `Text取数Agent` | `applications/data_process/prompts/memory_prompts.py` |
| `get_text2data_tool_description`、`get_py_coder_tool_description`、`get_metric_analyze_tool_description`、`get_web_search_tool_description`、`get_summary_tool_description` | `Text取数Agent` | `applications/data_process/tools_description/*.py` |
| `get_metric_route_judge_prompt` | `metric_route_judge_agent` | `applications/master_agent/agent/intention/metric_route_prompt.py` |
| `build_node_judge_prompt_from_laf` | `node_judge_agent` | `applications/master_agent/agent/ai_hosting_add/node_judge_prompt.py` |
| `get_global_summary_prompt_md` / `get_global_summary_prompt_html` | `global_summary_agent_chat` / `global_summary_agent_report` | `applications/summary_agent/prompt/*` |

## 注意事项

`Text取数Agent` 在 `run_master_0130.py` 中的 `prompt` 字段为空，真实 system prompt 是 `process_expert` 在运行时通过 `data_process_workflow` 传入的。这一点容易误导维护者：排查它的行为时，不要只看 Agent 注册代码，还要看 `shared_data["tool_description_dict"]`、`current_tool_description`、`build_multi_data_prompt` 和工具描述 provider。

`summary_tool`、`get_tool_documentation`、`retrieve_memory_detail` 是 prompt 协议工具，不是普通 `FunctionHub`。它们服务 `Text取数Agent` 的输出协议和工具切换协议，不能在 MAS 工具组织图里按普通工具查找。
