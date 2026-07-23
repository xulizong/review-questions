# process_and_summary_agent

`process_and_summary_agent` 是主执行流里的“单步骤执行编排 Agent”。它不负责拆计划，也不负责调度依赖；上游 `data_executer` 已经选定了当前可执行的 `GraphPlan` 节点，然后用 `{"command": "start"}` 触发它。它真正处理的内容不是这个 `command` 字段，而是从 `shared_data["current_step_index"]` 定位到当前步骤，再读取 `shared_data["plan"]["step_command"][current_step_index]`，把这一条步骤命令交给 `process_expert` 做数据处理，随后交给 `summary_expert` 做节点级结论整理。

它的实现入口是 `applications/master_agent/agent/master_graph_flow.py::combine_data_expert_workflow`，注册位置是 `applications/master_agent/run_master_0130.py`。注册时它是 `WorkflowAgent`，下挂 `process_expert` 和 `summary_expert`，本身没有独立 system prompt，也不直接调用业务工具。它更像一个固定 Python workflow：按固定顺序调用两个子 Agent，并把产物写回 `GraphPlan.plan_attachment`。

## 1. 入口契约

| 项 | 当前实现 |
| --- | --- |
| Agent 名称 | `process_and_summary_agent` |
| 类型 | `oxy.WorkflowAgent` |
| 注册位置 | `applications/master_agent/run_master_0130.py` |
| Workflow | `applications/master_agent/agent/master_graph_flow.py::combine_data_expert_workflow` |
| 上游 | `data_executer` |
| 下游 | `process_expert`、`summary_expert` |
| 直接 prompt | 无 |
| 直接业务工具 | 无 |
| 输入参数 | `{"command": "start"}`，用于触发流程 |
| 实际步骤输入 | `shared_data["current_step_index"]` + `shared_data["plan"]["step_command"]` |
| 返回值 | `summary_expert.output`，即当前节点总结文本 |
| 主要副作用 | 写入 `plan_attachment[current_step_index]["process_expert"]` 和 `plan_attachment[current_step_index]["summary_expert"]` |

这里要特别注意：`input_schema` 要求有 `command`，但 workflow 里没有读取 `oxy_request.arguments["command"]` 来决定执行内容。真正决定当前步骤做什么的是 `GraphPlan`。因此从外部看，它的入参是 `{"command": "start"}`；从业务看，它的入参是当前计划节点的 `step_command`。

## 2. 处理流程

```text
data_executer
  ↓  call process_and_summary_agent({"command": "start"})
process_and_summary_agent
  ↓  current_step_index = shared_data["current_step_index"]
  ↓  query = GraphPlan.step_command[current_step_index]
process_expert({"query": query})
  ↓  保存 plan_attachment[current_step_index]["process_expert"]
summary_expert({"query": "无"})
  ↓  读取当前步骤 process_expert 附件，生成节点总结
  ↓  保存 plan_attachment[current_step_index]["summary_expert"]
return summary_expert.output
```

`combine_data_expert_workflow` 第一段先读取当前步骤序号，再通过 `graph_plan_schema.get_current_command(...)` 取出 planner 生成的步骤命令。随后它向前端发送一次 `process_expert` 工具调用状态，并执行 `oxy_request.call(callee="process_expert", arguments={"query": query})`。如果 `process_expert.output` 是 JSON 字符串，会先尝试 `json.loads`；解析成功后按结构化对象写入当前步骤附件。

第二段固定调用 `summary_expert`，参数是 `{"query": "无"}`。这是因为 summary 阶段不再重新理解用户问题，而是读取刚写入的当前步骤 `process_expert` 附件。`summary_expert` 返回后，workflow 会把结果推送给前端，并继续保存到 `summary_expert` 附件字段，最后把这段 summary 文本作为 `process_and_summary_agent` 的输出返回给 `data_executer`。

## 3. process_expert 细节

`process_expert` 也是 `WorkflowAgent`，实现是 `applications/data_process/agents/data_process_workflow.py::data_process_workflow`。它的直接入参是 `{"query": 当前步骤命令}`，但运行时还依赖 `shared_data["user_data"]`、`shared_data["shortcut_mode"]`、`workspace_file_dict` 和当前计划上下文。它先把用户数据源整理成 Prompt 可读的 Markdown，再调用 `Text取数Agent` 完成取数、加工和结论生成。

`Text取数Agent` 的动态 Prompt、入参、四类知识、工具切换、`text2data_tool`、`py_coder_tool`、`summary_tool`、ReAct 消息处理和完整示例已拆分到 [text取数Agent分析.md](./text取数Agent分析.md)。本文只保留 `process_expert` 与上下游的边界。

在普通主流程中，有线上 dataset 时由 `text2data_tool` 开始，需要二次加工时切换到 `py_coder_tool`，最后通过 `summary_tool` 结束；只有本地 file 时由 `py_coder_tool` 开始，最后同样通过 `summary_tool` 结束。

### 3.1 process_expert 输出结构

`process_expert` 返回给 `process_and_summary_agent` 的典型结构如下：

```json
[
  {
    "process_data_ids": ["tmp_result_001"],
    "output_data": [
      {
        "data": "最后一次业务工具返回的数据或 stdout"
      }
    ],
    "conclusion": "基于工具结果生成的节点结论，由 summary_tool 输出。"
  }
]
```

如果 `Text取数Agent` 调用或后处理异常，`data_process_workflow` 会记录日志并返回空结果 `[]`。如果异常直接从 `process_expert` 调用处抛出，`process_and_summary_agent` 会捕获并返回“当前步骤无法顺利执行，请执行下一步骤。”，不会继续调用 `summary_expert`。

## 4. summary_expert 细节

`summary_expert` 的实现是 `applications/summary_agent/workflow/node_summary_and_visual_workflow.py::node_summary_and_visual_workflow`。它的入参固定为 `{"query": "无"}`，内部先调用 `node_summary_agent`，再根据结果判断是否需要调用 `visual_agent`。注册时虽然配置了 `llm_model="workflow_llm"`，但当前主流程里的 `node_summary_agent` 是 Python workflow，不会直接让 LLM 重新总结。

`node_summary_agent` 的实现是 `applications/summary_agent/workflow/node_summary_workflow.py::node_summary_workflow`。它读取当前步骤附件里的 `process_expert`，遍历其中每个 item 的 `conclusion` 字段，把这些 conclusion 拼接成节点总结，并调用 `fix_markdown_tables` 修复 markdown 表格。也就是说，当前节点总结的主要来源就是 `process_expert` 输出里的 `conclusion`，不是重新读取 `output_data` 做二次分析。

例如当前步骤附件是：

```json
{
  "process_expert": [
    {
      "output_data": [
        {
          "data": [
            {
              "SPU数量": 8642
            }
          ]
        }
      ],
      "conclusion": "当前筛选条件下共有 8,642 个 SPU。"
    }
  ]
}
```

`node_summary_agent` 不会再查数，也不会重新跑 Python，它只抽取 `conclusion`，返回：

```text
当前筛选条件下共有 8,642 个 SPU。
```

如果节点总结中命中 EasyBI 可视化协议，`node_summary_and_visual_workflow` 会调用 `visual_agent`。`visual_agent` 会从当前步骤的 `process_expert` 附件中提取可视化需要的 raw data，用 `applications/summary_agent/prompt/old/easybi_prompt.py::VISUAL_PROMPT` 拼出可视化请求，然后向 `config.bi_proxy_agent.server_url` 发起 HTTP POST：

```json
{
  "callee": "xy_atom_for_visualization_agent_for_c",
  "query": "VISUAL_PROMPT 拼出的可视化请求"
}
```

触发可视化的节点总结必须包含合法的 `<jmtCharts>...</jmtCharts>` 标签。例如：

```text
近 7 天成交金额整体上升。

<jmtCharts>折线图展示近 7 天成交金额变化，横轴为日期，纵轴为成交金额。</jmtCharts>
```

`visual_agent` 会把当前步骤的 `process_expert` 结果提取成 raw data，并把上面的节点总结一起塞进 `VISUAL_PROMPT`。远端可视化 Agent 返回后，`replace_echarts_segment_easybi(...)` 会把 `<jmtCharts>...</jmtCharts>` 中的占位描述替换成实际图表内容。未命中 `<jmtCharts>` 时，`summary_expert` 只返回 `node_summary_agent` 拼出的文本。

## 5. 附件写回格式

`GraphPlan.save_attachment(...)` 会把非 list 的值包装成 list，再追加到当前步骤附件里。因此 `process_expert` 保存的是数据处理结果列表，`summary_expert` 保存的是 `["总结文本"]`。

示例：第 0 步执行前，plan 附件为空。

```json
{
  "title": ["统计 054.csv 行数"],
  "step_command": ["【表格行数统计】读取 054.csv，计算不含表头的数据行数，并输出一句话结论。"],
  "plan_attachment": [{}],
  "step_status": [""],
  "step_result": [""]
}
```

`process_expert` 完成后，当前步骤附件会变成：

```json
{
  "process_expert": [
    {
      "process_data_ids": ["054_row_count"],
      "output_data": [
        {
          "data": "row_count = 12345"
        }
      ],
      "conclusion": "054.csv 共 12,345 行，统计不含表头。"
    }
  ]
}
```

`summary_expert` 会读取上面的 `conclusion`，输出并保存：

```json
{
  "process_expert": [
    {
      "process_data_ids": ["054_row_count"],
      "output_data": [
        {
          "data": "row_count = 12345"
        }
      ],
      "conclusion": "054.csv 共 12,345 行，统计不含表头。"
    }
  ],
  "summary_expert": [
    "054.csv 共 12,345 行，统计不含表头。"
  ]
}
```

`data_executer` 在单步结束后会把这个附件从子 request 合并回主 request 的 `shared_data["plan"]`。后续 `global_summary_agent` 不会重新执行步骤，而是从每个步骤的 `summary_expert` 附件中收集片段，组合成最终回答或报告。

## 6. 简化示例

假设 `data_planner` 生成了一个步骤：

```json
{
  "id": ["1"],
  "title": ["统计 SPU 数量"],
  "step_command": [
    "【统计 SPU 数量】# 任务和计算方法：基于“品策数据洞察SPU”数据源，统计满足当前筛选条件的 SPU 总数。\n\n# 输出格式：输出一句话，包含 SPU 总数。"
  ],
  "dependencies": [[]],
  "plan_attachment": [{}]
}
```

`data_executer` 启动第 0 步时，会复制 request，并写入：

```json
{
  "current_step_index": 0
}
```

然后调用：

```json
{
  "callee": "process_and_summary_agent",
  "arguments": {
    "command": "start"
  }
}
```

`process_and_summary_agent` 取出的真实 query 是 `step_command[0]`。它调用 `process_expert` 后，`process_expert` 会把数据源描述、语义知识、当前时间和工具描述拼入 `Text取数Agent` 的 prompt。在线上 dataset 场景下，初始工具是 `text2data_tool`，最终必须通过 `summary_tool` 输出结论。例如 `Text取数Agent` 的最终结论是：

```text
当前筛选条件下共有 8,642 个 SPU。
```

那么 `process_expert` 返回并保存的核心内容是：

```json
[
  {
    "process_data_ids": [],
    "output_data": [
      {
        "data": "spu_count = 8642"
      }
    ],
    "conclusion": "当前筛选条件下共有 8,642 个 SPU。"
  }
]
```

随后 `summary_expert` 不再重新查数，只从 `process_expert` 附件中抽取 `conclusion`，修复 markdown 后返回：

```text
当前筛选条件下共有 8,642 个 SPU。
```

最后当前步骤附件中会同时存在 `process_expert` 和 `summary_expert` 两份内容。前者偏过程产物和工具结果，后者偏最终可展示的节点结论。

## 7. 异常与边界

如果 `process_expert` 调用直接抛异常，`process_and_summary_agent` 会返回“当前步骤无法顺利执行，请执行下一步骤。”，并提前结束，不会保存 `process_expert` 附件，也不会调用 `summary_expert`。如果只是保存 `process_expert` 附件失败，workflow 只记录错误，仍会继续调用 `summary_expert`；这时 `summary_expert` 会读不到有效 `process_expert`，从而返回空总结。

如果 `summary_expert` 调用异常，当前 workflow 没有显式捕获，会向外抛给 `data_executer`，由 `data_executer._execute_single_step` 捕获后把该步骤标记为 `error`。如果 `summary_expert` 正常返回空字符串，workflow 仍会保存空结果并返回空结果，是否继续下游由 `data_executer` 的调度逻辑决定。

当前设计是“单步骤失败尽量不阻断整个计划”。这对弱依赖或并行步骤有容错价值，但对强依赖分析会带来结果传播风险：下游步骤能否继续，目前主要由 `data_executer` 的依赖完成状态控制，而不是由本 Agent 判断上游结论是否可靠。
