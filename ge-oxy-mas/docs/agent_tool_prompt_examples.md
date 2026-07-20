# Agent / Tool / Prompt 示例

本文档只放示例，主索引见 [agent_tool_prompt_overview.md](./agent_tool_prompt_overview.md)，各主要 Agent 的详解见 `docs/agents/`。

## Agent 调用链示例

用户发起普通数据分析请求时，`data_agent` 会把请求交给 `master_agent`。`master_agent` 先做输入预处理，再由 `data_planner` 生成图结构计划；`data_executer` 按计划调用 `process_and_summary_agent`；后者先通过 `process_expert` 和 `Text取数Agent` 完成取数或加工，再交给 `summary_expert`、`node_summary_agent`、`visual_agent` 输出节点结论，最后需要时由 `global_summary_agent` 汇总成完整报告。

快速查数模式下，`data_agent` 会直接调用 `sql_proxy_agent`，由后者提交 SQLAgent 任务并消费 SSE 返回结果。BI 类型请求会调用 `bi_proxy_agent`，先生成待确认的大纲，再二次调用 `bi_agent` 生成 dashboard。

## Tool 调用示例

`text2data_tool` 适合线上数据集的单一取数或 SQL 可表达的聚合计算。

```json
{
  "tool_name": "text2data_tool",
  "arguments": {
    "query": "查询昨日手机类目按品牌聚合的成交金额，并按成交金额降序排序",
    "dataset_id": "123456",
    "title": "昨日手机品牌成交金额"
  }
}
```

`py_coder_tool` 适合读取临时文件或本地文件做二次加工，代码必须独立运行，并通过一个 DataFrame 变量输出结果。

```json
{
  "tool_name": "py_coder_tool",
  "arguments": {
    "think": "读取 text2data_tool 生成的完整临时 CSV，按品牌计算成交金额占比",
    "code": "import pandas as pd\nurl = 'https://example.com/temp.csv?Expires=xxx'\ndf = pd.read_csv(url)\ntotal = df['成交金额'].sum()\nresult = df.groupby('品牌', as_index=False)['成交金额'].sum()\nresult['成交金额占比'] = result['成交金额'] / total\noutput_var = result",
    "title": "品牌成交金额占比",
    "output_var": "output_var"
  }
}
```

`metric_analyze_tool` 适合白名单指标的表现、归因、对比和定性评价。

```json
{
  "tool_name": "metric_analyze_tool",
  "arguments": {
    "query": "分析昨日成交金额表现及主要变化原因"
  }
}
```

`summary_tool` 只负责最终表达，不允许在总结阶段新增、估算或二次计算数字。

```json
{
  "tool_name": "summary_tool",
  "arguments": {
    "summary": "昨日手机类目成交金额为 123,456,789 元，品牌 A 成交金额为 45,678,901 元。以上数字均来自已执行工具结果。"
  }
}
```

`make_graph_plan` 是 planner 提交图计划的协议工具，不直接执行数据任务。

```json
{
  "tool_name": "make_graph_plan",
  "arguments": {
    "scene": "确定性分析",
    "global_config": "商品范围为手机类目，时间范围为昨日。",
    "steps": [
      {
        "id": "step_1",
        "title": "分析整体成交表现",
        "detail": "查询昨日手机类目的成交金额、成交订单量和用户转化率，并判断整体表现。",
        "output": "输出核心指标表格和简短结论。",
        "dependencies": []
      }
    ],
    "layout_config": "按分析步骤组织报告。"
  }
}
```

## Prompt 使用示例

当用户输入“分析昨日手机类目成交金额为什么下降”时，`metric_route_judge_agent` 会用 `METRIC_ROUTE_JUDGE_PROMPT` 判断该问题既包含归因意图，又命中“成交金额”指标白名单，因此 `multi_data` prompt 会优先提示 `Text取数Agent` 尝试 `metric_analyze_tool`。如果远端返回不支持或失败，再回退到 `text2data_tool` 取数和 `summary_tool` 总结。

当用户输入“帮我把这个很长的分析需求整理一下，但不要新增分析目标”时，`enhance_agent_flow` 会更适合使用 `INTENTION_ENHANCE_PROMPT_FORMAT` 的长文本规范化逻辑；如果输入只是“帮我分析手机类目最近表现”，则更适合 `INTENTION_ENHANCE_PROMPT`，将模糊问题扩写成清晰的取数和分析任务。

当请求需要占位符复用时，`extract_placeholder_agent` 会把关键字映射和待替换文本交给 `EXTRACT_PLACEHOLDER_PROMPT`。例如关键字映射为 `{"品牌": "苹果", "时间": "月至今"}` 时，文本中的“Apple”“苹果”“这个月”等等价表达会被替换为 `{{品牌}}` 或 `{{时间}}`，同时保持原 JSON 或自然语言结构不变。
