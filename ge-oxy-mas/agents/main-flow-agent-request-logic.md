# 主流程 Agent 类型与请求处理逻辑

本文从“Agent 类型”的角度解释 `master_agent` 常规主流程里预设好的请求处理逻辑。范围限定为普通 chat/report 数据分析链路：`master_agent → data_planner → data_executer → process_and_summary_agent → process_expert/Text取数Agent → summary_expert → global_summary_agent`。查数快捷流、单步快捷流虽然会复用部分子 Agent，但不属于多步骤主执行流；探索、QA、占位符、全托管、报告解读、direct 执行等特殊分支不在本文展开。

## 类型总览

| Agent 类型 | 主流程实例 | 底层预设逻辑 |
| --- | --- | --- |
| `WorkflowAgent` | `master_agent`、`process_and_summary_agent`、`process_expert`、`summary_expert`、`node_summary_agent`、`visual_agent`、`global_summary_agent` | 不默认让模型自主推理，核心是执行注册时绑定的 `func_workflow`。workflow 内部显式决定读写哪些 `shared_data`、调用哪些子 Agent、如何组织返回。 |
| `DataGraphPlanner` | `data_planner` | 自定义 ReAct planner。它把数据概况、知识、历史和时间信息放进 planner prompt，让模型输出 `make_graph_plan` 或 `search_knowledge` JSON；其中 `make_graph_plan` 被当作最终答案，不真的执行工具。 |
| `DataExecuter` | `data_executer` | 自定义执行器。它读取 planner 已经生成的 `GraphPlan`，按 `dependencies` 调度可运行节点，调用 `process_and_summary_agent` 执行节点，再用 `update_plan` 更新状态。它不生成图，也不直接取数。 |
| `ProgressiveReActAgent` | `Text取数Agent` | 工具循环执行器。它基于 `process_expert` 动态拼好的 prompt 和当前工具描述，让模型输出工具调用 JSON；工具结果写回 ReAct memory，直到调用 `summary_tool` 形成最终业务结论。 |
| `ChatAgent` / `ChatProAgent` / `BackupChatAgent` | `metric_route_judge_agent`、`node_judge_agent`、`global_summary_agent_chat` | 单轮或带备份的 LLM 调用。通常只做判断、裁判或最终 Markdown 总结，不承担步骤编排。 |
| `GlobalSummaryAgent` | `global_summary_agent_report` | 报告模式专用的 HTML 总结 Agent。它把已合并的 Markdown/分段报告转成 HTML，输出过短或异常时走备份模型。 |

## 主流程顺序

普通请求进入 `master_agent` 后，`master_agent` 先运行预处理逻辑补齐数据源描述、知识召回和输入模式，再进入 chat/report 主链路。主链路先调用 `data_planner` 产出图计划；如果不是查数快捷流或单步快捷流，就交给 `data_executer` 根据图依赖执行每个步骤。每个步骤实际落到 `process_and_summary_agent`，它先调用 `process_expert` 取数和加工，再调用 `summary_expert` 生成节点结论。所有步骤完成后，`global_summary_agent` 收集每步 `summary_expert` 的结果，按 chat/report 模式输出最终答案。

```text
master_agent
  ↓
data_planner
  ↓
data_executer
  ↓
process_and_summary_agent
  ├─ process_expert → Text取数Agent → text2data_tool / py_coder_tool / metric_analyze_tool / web_search_tool
  └─ summary_expert → node_summary_agent / visual_agent
  ↓
global_summary_agent → global_summary_agent_chat / global_summary_agent_report
```

## LLM 消息处理细节

所有 `LocalAgent` 子类都会复用一层基础能力：`_build_instruction(arguments)` 会把 prompt 里的 `${变量名}` 替换为 `oxy_request.arguments` 中的值；如果请求还没有短期记忆，`_pre_process` 会从历史中加载 `short_memory`。真正进入 LLM 前的 `messages` 怎么拼、LLM 返回后怎么处理，则由具体 Agent 类型决定。

| Agent 类型 | 请求 LLM 前的 messages | LLM 返回后的处理 |
| --- | --- | --- |
| `WorkflowAgent` | 自身不组 LLM messages，直接执行 `func_workflow(oxy_request)`。如果 workflow 内部需要 LLM，会显式调用其他 Agent 或 LLM。 | 自身不解析 LLM 返回，只把 workflow 返回值包装成 `OxyResponse(COMPLETED)`。 |
| `ChatAgent` | `system` 为 prompt 变量替换后的文本，然后追加 `short_memory`，最后追加当前用户 query。 | 直接返回 LLM 响应，不做 JSON 解析、工具调用或业务校验。`node_judge_agent` 虽注册了 `judge_tool`，但它本身仍只返回文本/JSON，后续由调用方解析并决定是否调用 `judge_tool`。 |
| `ChatProAgent` | 与 `ChatAgent` 基本一致，但 query 长度超过 `switch_threshold` 时会切到 `prompt_backup[0]`。 | 主模型输出为空、异常或过短时最多重试 2 次；仍失败且配置了 backup model 时调用备份模型。返回内容不做业务解析。 |
| `BackupChatAgent` | 与 `ChatAgent` 一致，`global_summary_agent_chat` 使用这一类生成 Markdown 总结。 | 主模型输出异常或少于 3 个字符时重试，失败后走备份模型；返回前会去掉 `</think>` 之前的思考内容。 |
| `GlobalSummaryAgent` | `system` 为 HTML 总结 prompt，追加短期记忆；然后把待转换的 Markdown 报告作为一条 `assistant` 消息，再追加固定 `user` 消息：“好的，现在请把这个markdown格式的报告转换成HTML格式的报告。” | 如果输出等于模型友好错误，或长度小于输入 Markdown 的 10%，调用备份模型。它只做模型级兜底；HTML 清洗、样式注入和数据来源拼接在外层 `global_summary_workflow` 中完成。 |
| `DataGraphPlanner` | 每轮构造 `system` planner prompt；prompt 会按 `AUTO/REPORT/DEEP_RESEARCH` 模式选择，并注入 `make_graph_plan`、`search_knowledge` 的工具说明。随后混合当前 Agent 短期记忆和用户级历史计划记忆，再追加当前 query 和本轮 ReAct memory。 | 先去掉 `</think>`，再抽取第一个 JSON。如果 `tool_name=make_graph_plan`，直接当最终答案返回；如果是 `search_knowledge`，执行工具并把 observation 写回 ReAct memory；解析失败时把错误提示写回 ReAct memory 让模型重试。 |
| `DataExecuter` | 主路径通常不请求 LLM。它先构造 executor prompt、短期记忆、当前 query，再为每个步骤追加伪造的 `get_plan_status` assistant tool call 和对应 user observation，供可选 LLM 更新计划时使用。 | 默认不解析 LLM，而是执行完 `process_and_summary_agent` 后直接构造 `update_plan` 结果。只有 `get_master_config().use_llm=true` 时，才把步骤执行上下文发给 LLM，并用 `json_repair + extract_first_json` 解析 `update_plan`；解析失败回退默认 `update_plan`。 |
| `ProgressiveReActAgent` | 每轮都会刷新 `workspace`、`current_tool_name`、`current_tool_description`，必要时用 `arguments["prompt"]` 替换注册 prompt。messages 为：动态 system prompt、短期记忆、带“禁止自行计算，必须用工具”约束的当前 query、历史 ReAct memory。入模前还会按 `memory_max_tokens` 截断，并额外传 `input_md5_memory` 给 LLM。 | 通过 `func_parse_llm_response_for_tool` 解析。普通工具 JSON 进入 `TOOL_CALL`；`get_tool_documentation` 和 `retrieve_memory_detail` 是内部协议工具；业务工具会真实调用并把 observation 写回 ReAct memory。`summary_tool` 会进入后置裁判、数值溯源、数值标签化、dataset_id 修复和 Markdown 表格修复，最后作为 `ANSWER` 返回；解析失败则把错误提示写回 ReAct memory 让模型重试。 |

`DataGraphPlanner` 和 `ProgressiveReActAgent` 都是 ReAct 类 Agent，但语义不同。planner 的 `make_graph_plan` 是“提交计划”的协议，所以一旦模型输出它就结束；它的上游上下文、知识检索和下游解析细节见 [data-planner-flow.md](./data-planner-flow.md)。`Text取数Agent` 的业务工具是真执行，工具 observation 会不断回灌到上下文，直到模型通过 `summary_tool` 结束。`DataExecuter` 虽然继承 ReActAgent，但当前主路径更像 Python 调度器：它负责按图依赖启动步骤，不让模型自由决定执行哪个业务工具。它的上游计划转换、依赖调度和下游传参细节见 [data-executer-flow.md](./data-executer-flow.md)。

## WorkflowAgent 的处理逻辑

`WorkflowAgent` 的行为最固定：收到请求后直接执行注册时传入的 `func_workflow`，返回这个 Python workflow 的结果。它自身不负责自动构造 prompt，也不会自动选择工具；是否调用 LLM、调用哪个子 Agent、写入哪些状态，都由 workflow 代码显式控制。因此看这类 Agent 时，重点不是看 `llm_model` 字段，而是看 `func_workflow`。

主流程中 `master_agent` 的 workflow 是 `super_master_workflow`。它先执行 `add_data_desc`、`pre_process_clarifier`、`get_input_mode`，再根据模式进入 `chat_workflow` 或 `report_workflow`。`process_and_summary_agent` 的 workflow 是 `combine_data_expert_workflow`，固定执行“处理 → 总结”两段：先把当前步骤 command 传给 `process_expert`，保存 `process_expert` 附件，再调用 `summary_expert`，保存 `summary_expert` 附件。`process_expert` 的 workflow 是 `data_process_workflow`，它负责把数据源、知识、workspace、工具描述拼成 `Text取数Agent` 的运行上下文。`global_summary_agent` 的 workflow 是 `global_summary_workflow`，它负责把多步骤结果合并后分派给 chat 或 report 总结 Agent。

## data_planner 的处理逻辑

`data_planner` 是 `DataGraphPlanner`，本质是一个被改造过的 ReAct Agent。它每轮把 planner prompt、短期记忆、用户问题、数据源概况、默认知识、few-shot 和联网提示合成消息，再调用 `plan_llm`。模型输出必须是 JSON，并带 `tool_name`。如果 `tool_name=search_knowledge`，它会真实调用知识检索工具，把 observation 写回上下文继续规划；如果 `tool_name=make_graph_plan`，代码会把这段 JSON 直接视为最终规划结果返回。

这里最容易误解的是 `make_graph_plan`。它在 prompt 里看起来像工具，但 `DataGraphPlanner` 把它放进 `trans_to_answer_tools`，所以不会像普通工具一样被执行；它只是模型提交计划的协议格式。真正的图结构来自 `make_graph_plan.arguments.steps[*].dependencies`，后续被解析成 `GraphPlan.id/title/step_command/dependencies`。

## data_executer 的处理逻辑

`data_executer` 是 `DataExecuter`，它处理的输入不是原始用户问题，而是 `shared_data["plan"]` 里的 `GraphPlan`。它先按 `dependencies` 计算哪些步骤可以启动；有依赖的步骤必须等待上游完成，没有显式依赖时按串行逻辑降级。运行时它会复制当前 `OxyRequest`，写入 `shared_data["current_step_index"]`，再调用 `process_and_summary_agent` 执行这个节点。

执行器支持并发，最大并发来自 `get_max_concurrent()`。每个节点结束后，它合并该节点产生的 `plan_attachment`、`workspace_file_dict` 和节点 memory，并把 `step_status`、`step_result`、`completed_step` 写回 `GraphPlan`。默认配置下，它不依赖 LLM 判断下一步，而是直接构造 `update_plan` 调用；只有 `get_master_config().use_llm` 打开时，才会让模型参与生成 `update_plan`。因此它的核心职责是“按已有图调度执行”，不是“把节点编排成图形状”。

## Text取数Agent 的处理逻辑

`Text取数Agent` 是主流程里真正做工具循环的 Agent，但它的 prompt 不是注册时写死的。`process_expert` 会先根据数据源类型和场景推导可用工具：有在线 dataset 时启用 `text2data_tool`，有本地或临时文件时始终启用 `py_coder_tool`，`metric_analyze_tool` 和 `web_search_tool` 只在 `tool_list` 或路由条件命中时加入。随后它把 `dataset_markdown`、`file_markdown`、语义层知识、workspace 临时数据、当前工具描述等放进动态 prompt，再调用 `Text取数Agent`。

`Text取数Agent` 每轮都会刷新 workspace 和当前工具描述，然后要求模型输出一个 JSON 工具调用。普通工具调用会执行 `text2data_tool`、`py_coder_tool`、`metric_analyze_tool` 或 `web_search_tool`，工具结果作为 observation 写回 ReAct memory。内部协议工具 `get_tool_documentation` 用来切换当前工具描述，`retrieve_memory_detail` 用来取压缩记忆详情。最后模型必须通过 `summary_tool` 输出业务结论；解析器会对 summary 做数值来源校验、数值标签化、Markdown 表格修复和后置裁判，未通过时以 `ERROR_PARSE` 写回上下文要求重试。

## 总结类 Agent 的处理逻辑

`summary_expert` 本身是 WorkflowAgent，默认调用 `node_summary_agent` 和 `visual_agent`。当前 `node_summary_agent` 不再用 LLM 重写节点结论，而是从当前步骤附件的 `process_expert` 输出中抽取每个 item 的 `conclusion` 字段，并做 Markdown 表格修复。`visual_agent` 会从 `process_expert` 附件中抽取可视化所需字段，拼接 EasyBI visual prompt，再请求 `bi_proxy_agent.server_url` 下的远端可视化 Agent。

`global_summary_agent` 也是 WorkflowAgent。它不会重新取数，也不会重新执行任何步骤，只读取每个步骤已保存的 `summary_expert` 结果和 planner 生成的 `outline_full`。chat 模式调用 `global_summary_agent_chat`，用 `BackupChatAgent` 生成 Markdown；report 模式调用 `global_summary_agent_report`，用 `GlobalSummaryAgent` 生成 HTML，然后执行清洗、样式注入和数据来源拼接。最终报告缺内容时，应优先检查对应步骤的 `summary_expert` 附件，而不是先查全局总结 prompt。

## 判断类 Agent 的处理逻辑

主流程里还有少量判断类 Agent，它们不是执行主干。`metric_route_judge_agent` 是 `ChatProAgent`，只在 `process_expert` 判断是否优先使用 `metric_analyze_tool` 时被调用，输出倾向是“是/否”。`node_judge_agent` 是 `ChatAgent`，服务 `Text取数Agent` 的后置裁判链路，用 `judge_tool` 和节点裁判 prompt 检查结论是否合规。这类 Agent 的特点是输入窄、输出短，不负责调度步骤，也不保存图计划附件。

## 对应代码位置

| 逻辑 | 主要代码 |
| --- | --- |
| prompt 变量替换与短期记忆预处理 | `oxygent/oxy/agents/local_agent.py` |
| `WorkflowAgent` / `ChatAgent` / `ChatProAgent` / `BackupChatAgent` / `GlobalSummaryAgent` 消息处理 | `oxygent/oxy/agents/*.py` |
| `ProgressiveReActAgent` ReAct 循环 | `oxygent/oxy/agents/progressive_react_agent.py` |
| Agent 注册 | `applications/master_agent/run_master_0130.py` |
| master 主 workflow | `applications/master_agent/agent/master_graph_flow.py` |
| planner 类型实现 | `applications/master_agent/agent/graph_planner.py` |
| executer 类型实现 | `applications/master_agent/agent/graph_executor.py` |
| process_expert 动态 prompt 和工具选择 | `applications/data_process/agents/data_process_workflow.py` |
| Text取数Agent 输出解析 | `applications/data_process/util/llm_response_parser.py` |
| 节点总结与可视化 | `applications/summary_agent/workflow/node_summary_workflow.py`、`applications/summary_agent/workflow/node_visual_workflow.py` |
| 全局总结 | `applications/summary_agent/workflow/global_summary_workflow.py` |
