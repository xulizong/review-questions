# data_executer 上下游处理流程

`data_executer` 是 `master_agent` 常规多步骤主流程里的图计划执行器，注册在 `applications/master_agent/run_master_0130.py`，实现类是 `applications/master_agent/agent/graph_executor.py::DataExecuter`。它不负责生成图，也不直接取数、写代码或总结；它拿到上游已经成形的 `GraphPlan`，按依赖关系决定哪些步骤可以执行，然后把当前步骤交给 `process_and_summary_agent`。

## 上游传入的内容

`data_planner` 原始输出是 `make_graph_plan` 协议 JSON。这个 JSON 不会原样交给 `data_executer`，而是先由 `master_graph_flow.parse_graph_planner_result` 压平，再构造成 `GraphPlan`，写入 `shared_data["plan"]`。`data_executer` 真正读取的是这个 `shared_data["plan"]`。

一个简化的 planner 输出如下：

```json
{
  "tool_name": "make_graph_plan",
  "arguments": {
    "scene": "确定性分析",
    "global_config": "分析近30天，商品范围限定为重点SPU",
    "steps": [
      {
        "id": "1",
        "title": "销售变化概览",
        "detail": "按日期统计销售额和销量，判断下降发生在哪些时间段。",
        "output": "输出趋势表和关键变化结论。",
        "dependencies": []
      },
      {
        "id": "2",
        "title": "渠道拆解",
        "detail": "基于步骤1结果，按渠道拆解销售额下降贡献。",
        "output": "输出渠道贡献排序和主要异常渠道。",
        "dependencies": ["1"]
      },
      {
        "id": "3",
        "title": "结论汇总",
        "detail": "结合前两步结果，给出下降原因和建议。",
        "output": "输出最终结论和建议。",
        "dependencies": ["1", "2"]
      }
    ]
  }
}
```

`parse_graph_planner_result` 会把每个 step 的 `title/detail/output` 合成一条可执行指令，并把 `dependencies` 保持为步骤 id 列表。上面例子会变成：

```json
{
  "id": ["1", "2", "3"],
  "title": ["销售变化概览", "渠道拆解", "结论汇总"],
  "command": [
    "【销售变化概览】\n# 任务和计算方法：按日期统计销售额和销量，判断下降发生在哪些时间段。\n\n # 输出格式：输出趋势表和关键变化结论。",
    "【渠道拆解】\n# 任务和计算方法：基于步骤1结果，按渠道拆解销售额下降贡献。\n\n # 输出格式：输出渠道贡献排序和主要异常渠道。",
    "【结论汇总】\n# 任务和计算方法：结合前两步结果，给出下降原因和建议。\n\n # 输出格式：输出最终结论和建议。"
  ],
  "dependencies": [[], ["1"], ["1", "2"]],
  "layout_config": ""
}
```

随后 `chat_workflow` / `report_workflow` 构造 `GraphPlan`。此时 `data_executer` 看到的核心结构是：

```json
{
  "plan_id": "<trace_id>",
  "id": ["1", "2", "3"],
  "title": ["销售变化概览", "渠道拆解", "结论汇总"],
  "step_command": ["<第1步指令>", "<第2步指令>", "<第3步指令>"],
  "dependencies": [[], ["1"], ["1", "2"]],
  "plan_attachment": [{}, {}, {}],
  "step_result": [],
  "step_status": [],
  "completed_step": 0
}
```

这里最关键的是两个映射：`id` 是 planner 生成的稳定步骤 id，`step_command` 是真正给下游执行的步骤指令。`dependencies` 里引用的是步骤 id，不是数组下标，所以 `data_executer` 会先建立 `id_to_index = {"1": 0, "2": 1, "3": 2}`，再用它判断依赖是否完成。

## data_executer 如何调度

`data_executer` 启动后先把 `shared_data["plan"]` 还原成 `GraphPlan`，并补齐 `step_result`、`step_status` 的长度。然后它维护三个运行状态：`completed_indices` 表示已完成的步骤下标，`running_tasks` 表示正在跑的异步任务，`task_memories` 保存每个步骤完成后给后续依赖节点使用的 memory。

对于上面的三步例子，调度过程是：

```text
初始 completed_indices = {}
  ↓
第1步 dependencies=[]，可以启动
第2步 dependencies=["1"]，等待第1步
第3步 dependencies=["1","2"]，等待第1步和第2步
  ↓
第1步完成后 completed_indices={0}
  ↓
第2步依赖满足，可以启动，并注入第1步产生的 memory
  ↓
第2步完成后 completed_indices={0,1}
  ↓
第3步依赖满足，可以启动，并注入第1步、第2步产生的 memory
```

如果有两个步骤都只依赖第1步，那么第1步完成后它们会同时进入候选队列，实际能并发启动几个由 `get_max_concurrent()` 返回值控制。没有 `dependencies` 字段时，代码会退化成串行执行：第 N 步默认等待第 N-1 步完成。

## 传给下游的内容

`data_executer` 传给下游时，不是把当前步骤指令直接放进 `arguments`。它会复制当前 `OxyRequest`，在副本里写入当前步骤下标：

```json
{
  "shared_data": {
    "current_step_index": 1,
    "plan": {
      "step_command": ["<第1步指令>", "<第2步指令>", "<第3步指令>"]
    }
  },
  "arguments": {
    "command": "start"
  }
}
```

然后它调用：

```text
process_and_summary_agent({"command": "start"})
```

`command=start` 只是启动动作标识。真正的业务指令由 `process_and_summary_agent` 自己读取：它先拿 `shared_data["current_step_index"]`，再调用 `graph_plan_schema.get_current_command(...)` 从 `shared_data["plan"].step_command[index]` 里取出当前步骤指令。以上面例子为例，当 `current_step_index=1` 时，下游拿到的是第二步：

```text
【渠道拆解】
# 任务和计算方法：基于步骤1结果，按渠道拆解销售额下降贡献。

 # 输出格式：输出渠道贡献排序和主要异常渠道。
```

随后 `process_and_summary_agent` 固定做两件事：先调用 `process_expert({"query": 当前步骤指令})`，把取数和加工结果保存到当前步骤附件的 `process_expert`；再调用 `summary_expert({"query": "无"})`，由 summary 侧读取附件生成节点结论，并保存到当前步骤附件的 `summary_expert`。

## 下游结果如何回到 plan

每个步骤执行完成后，`data_executer` 会从这个步骤的 request 副本里取回两类产物：一类是当前步骤写入的 `plan_attachment[step_index]`，一类是工具加工产生的 `workspace_file_dict`。然后它把这些产物合并回主 request 的 `shared_data`。

一个完成后的附件形态可以理解为：

```json
{
  "process_expert": [
    {
      "output_data": [{"data": "<工具输出或临时数据信息>"}],
      "conclusion": "渠道A贡献了主要销售额下降..."
    }
  ],
  "summary_expert": [
    "渠道拆解显示，渠道A是主要下降来源，贡献占比最高..."
  ]
}
```

同时，`data_executer` 会把当前步骤状态写回 `GraphPlan`：

```json
{
  "step_status": ["continue", "continue", ""],
  "step_result": [
    "步骤1(销售变化概览)已执行，结果：已执行了process_and_summary_agent，完成当前步骤分析",
    "步骤2(渠道拆解)已执行，结果：已执行了process_and_summary_agent，完成当前步骤分析",
    ""
  ],
  "completed_step": 2
}
```

这里的 `step_result` 是执行器层面的步骤状态摘要，不是最终业务结论全文。真正给全局总结使用的业务内容在 `plan_attachment[*]["summary_expert"]` 里。所有步骤结束后，`data_executer` 返回一段执行摘要；后续 `pre_process_report_expert` 会遍历 `GraphPlan.plan_attachment`，收集每个步骤的 `summary_expert`，再交给 `global_summary_agent` 做最终汇总。

## 失败与特殊处理

如果某个步骤执行异常，`_execute_single_step` 会返回 `command="error"` 和错误摘要；如果 `workflow_until_target` 没拿到结果，会使用备选结果 `command="skip"`。当所有已执行步骤都是 `skip` 时，`data_executer` 会额外调用 LLM 生成一句整体失败说明；否则它仍然返回已完成步骤摘要，让主流程继续进入全局总结。

重跑场景下，`data_executer` 会读取 `restart_node_input`，从指定 `restart_step_index` 之后覆盖 `step_command` 和 `title`，并在每个任务 request 里写入 `_is_read_cache_for_restart`，用于控制重跑缓存。这个逻辑只改变执行计划和缓存标识，不改变“按依赖启动步骤、通过 `current_step_index` 交给下游”的主流程。

## 核心结论

`data_executer` 的上下游边界可以简化为一句话：上游给它的是已经解析成 `GraphPlan` 的图计划，它用 `dependencies` 决定执行顺序，用 `current_step_index` 告诉下游当前该执行哪一步；下游把步骤附件写回 `plan_attachment`，它再合并状态、附件和临时文件信息，供后续 `global_summary_agent` 聚合。它不是 planner，也不是数据执行工具，而是图计划的调度器和状态合并器。
