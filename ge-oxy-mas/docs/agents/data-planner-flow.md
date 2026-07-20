# data_planner 上下游处理流程

`data_planner` 是 `master_agent` 常规主流程里的计划生成 Agent，注册在 `applications/master_agent/run_master_0130.py`，实现类是 `applications/master_agent/agent/graph_planner.py::DataGraphPlanner`。它的职责是把用户问题、数据源概况、知识和历史上下文整理成图结构计划；它不执行取数、不加工数据，也不直接调用 `process_expert`。真正执行计划的是下游 `data_executer`。

## 上游传入的内容

`data_planner` 由 `chat_workflow` 或 `report_workflow` 调用。进入它之前，`master_agent` 已经完成 `add_data_desc` 和 `pre_process_clarifier`，因此 planner 看到的不是原始用户输入，而是补齐后的分析上下文。核心入参如下：

```json
{
  "query": "分析近30天销售额下降原因",
  "all_data_summary": "第1个数据源: 品策数据洞察SPU,类型为在线数据集,描述: ... 包含的列信息:\n| column name | value type | ...",
  "user_level_short_memory": ["<历史对话>"],
  "day_info": "当前日期、昨日、近7天、上月等时间信息",
  "default_knowledge": "默认召回知识，包括系统知识、用户知识、语义层知识",
  "recall_few_shot": "从 few-shot 库召回的 planner 示例",
  "planner_web_search_info": "是否允许 planner 规划 web_search_tool 的提示"
}
```

如果是报告模式，`shared_data.type=2` 时 `DataGraphPlanner` 会把模式从 `AUTO` 切成 `report`，使用 `get_report_mode_planner_prompt`。如果 `shared_data.mode="DEEP_RESEARCH"`，则使用 deep research prompt。普通 chat 默认使用 `get_planner_prompt`。这些 prompt provider 通过 DUCC/LAF 读取，远端没有覆盖时回退 `applications/master_agent/prompt/graph_plan_prompt.py`。

## planner 如何组织 LLM 请求

`DataGraphPlanner` 本质是一个定制 ReAct Agent。每一轮请求 LLM 前，它会先构造 system prompt，并把 `tools_description` 注入 prompt 变量。planner 可见的工具只有两个：`search_knowledge` 和 `make_graph_plan`。其中 `search_knowledge` 是真实工具调用，`make_graph_plan` 是提交计划的协议。

消息顺序可以简化为：

```text
system: planner prompt + tools_description + 数据概况/知识/few-shot 等变量
history: master 短期记忆 + 用户级历史计划记忆
user: 当前 query
react_memory: 上轮工具调用和 observation，如果有
```

用户级历史计划记忆会被转换成类似下面的形式再混入上下文，这样模型能参考历史问题对应的计划结构：

```text
user: Tool [make graph plan] execution result: <历史 assistant 里生成过的计划>
assistant: 执行完成。
```

## 工具协议如何处理

LLM 返回后，`DataGraphPlanner` 会先去掉 `</think>` 之前的内容，再抽取第一个 JSON。如果 JSON 里没有 `tool_name`，或者 JSON 不合法，会把错误提示写回 ReAct memory，让模型重试。

当模型输出 `search_knowledge` 时，planner 会真实调用工具，并把知识召回结果作为 observation 写回上下文。例如：

```json
{
  "tool_name": "search_knowledge",
  "arguments": {
    "query": "近30天 销售额下降 渠道 SPU"
  }
}
```

返回结果会进入下一轮：

```text
Tool [search_knowledge] execution result: 系统知识库：...
语义层知识库：...
```

当模型输出 `make_graph_plan` 时，planner 不会真的执行 `make_graph_plan` 函数。代码里 `trans_to_answer_tools = ["make_graph_plan"]`，因此这个 JSON 会被直接当作最终答案返回。`make_graph_plan` 函数本身如果被真实调用，只会返回“此时使用make_graph_plan是错误的。”，这也说明它在 planner 里只是计划提交协议。

## planner 输出示例

以下是 planner 最终返回给 `master_graph_flow` 的典型结构：

```json
{
  "tool_name": "make_graph_plan",
  "arguments": {
    "scene": "确定性分析",
    "global_config": "分析近30天，商品范围限定为重点SPU",
    "layout_config": "",
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

这里的 `scene` 决定后续主流程如何分流，常见值包括 `查数`、`确定性分析`、`探索性分析`、`报告解读`、`无关问题`。`global_config` 是对所有步骤都生效的全局限制，会在后续解析时加到每个步骤指令前面。`steps[*].dependencies` 是图结构的核心，它声明当前步骤依赖哪些前置步骤 id；这一步只定义图，不执行图。

## 下游如何接收 planner 结果

`master_graph_flow.parse_graph_planner_result` 会把 `make_graph_plan.arguments` 转成 `plan_type` 和 `plan_dict`。它会抽取 `scene` 作为 `plan_type`，把每个 step 的 `id/title/dependencies` 保留下来，并把 `title/detail/output` 合并为 `command`。

上面的 planner 输出会被整理成：

```json
{
  "id": ["1", "2", "3"],
  "title": ["销售变化概览", "渠道拆解", "结论汇总"],
  "command": [
    "【全局配置须知】：分析近30天，商品范围限定为重点SPU\n\n【销售变化概览】\n# 任务和计算方法：按日期统计销售额和销量，判断下降发生在哪些时间段。\n\n # 输出格式：输出趋势表和关键变化结论。",
    "【全局配置须知】：分析近30天，商品范围限定为重点SPU\n\n【渠道拆解】\n# 任务和计算方法：基于步骤1结果，按渠道拆解销售额下降贡献。\n\n # 输出格式：输出渠道贡献排序和主要异常渠道。",
    "【全局配置须知】：分析近30天，商品范围限定为重点SPU\n\n【结论汇总】\n# 任务和计算方法：结合前两步结果，给出下降原因和建议。\n\n # 输出格式：输出最终结论和建议。"
  ],
  "dependencies": [[], ["1"], ["1", "2"]],
  "layout_config": ""
}
```

随后 `chat_workflow` / `report_workflow` 构造 `GraphPlan`，写入 `shared_data["plan"]`：

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

这个 `GraphPlan` 才是 `data_executer` 的直接输入。也就是说，`data_planner` 的下游边界不是“执行某一步”，而是“生成可被 `GraphPlan` 表达的步骤列表和依赖关系”。

## 主流程分流

planner 输出后，主流程会先看 `plan_type`。如果是 `无关问题`，会直接把 `title` 拼成回答，不进入执行器；如果是 `报告解读`，会转给 `read_report_agent`；如果是 `查数` 且只有一步，会进入 `query_data_flow` 快捷查数；如果只有一个普通分析步骤，会进入 `easy_analysis_flow`。只有常规多步骤计划才会进入 `data_executer`。

chat 模式里还有一个保护逻辑：如果 `plan_type == "查数"` 但 planner 给出了多步计划，代码会把它改成 `确定性分析`，避免多步查数绕过执行器。

## 异常与兜底

planner 的 LLM 输出为空时，会再请求一次主模型；如果仍为空，会请求 `plan_backup_llm`。如果输出不是合法 JSON，或者缺少 `tool_name`，`DataGraphPlanner` 会把格式错误写回 ReAct memory，让模型按正确格式重试。`parse_graph_planner_result` 自身解析失败时会返回 `("报错", {})`，但下游 `chat_workflow` / `report_workflow` 后续仍会读取 `plan_dict["id"]`、`plan_dict["title"]` 等字段，因此严重格式错误仍可能中断主流程。

## 核心结论

`data_planner` 的上下游边界可以简化为一句话：上游给它用户问题、数据源摘要、知识和历史，它通过 planner prompt 产出 `make_graph_plan` 协议 JSON；下游把这个 JSON 压平成 `GraphPlan`，再按场景决定是快捷查数、单步分析、报告解读、无关问题回复，还是进入 `data_executer` 执行多步骤图计划。
