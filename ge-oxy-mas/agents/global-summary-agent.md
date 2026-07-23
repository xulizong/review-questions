# global_summary_agent

`global_summary_agent` 是 `master_agent` 常规主流程的最终全局总结编排 Agent。它不取数、不写代码、不重新执行任何图节点；它只读取已经执行完的步骤结果，把每个步骤的 `summary_expert` 节点结论和 planner 生成的步骤大纲合并起来，再按当前请求模式输出最终 Markdown 答案或 HTML 报告。

它和 `summary_expert` 的边界要分清：`summary_expert` 是单个步骤结束后的局部总结，产物保存到 `GraphPlan.plan_attachment[*]["summary_expert"]`；`global_summary_agent` 是所有步骤结束后的跨步骤汇总，输入正是这些已经保存好的局部总结。

## 1. 位置与上下游

| 项 | 内容 |
| --- | --- |
| 注册位置 | `applications/master_agent/run_master_0130.py` |
| Agent 类型 | `WorkflowAgent` |
| Workflow | `applications/summary_agent/workflow/global_summary_workflow.py::global_summary_workflow` |
| 上游准备函数 | `applications/master_agent/agent/master_graph_flow.py::pre_process_report_expert` |
| 上游主要数据 | `shared_data["plan"].plan_attachment[*]["summary_expert"]`、`GraphPlan.title`、用户原始 query |
| 下游子 Agent | `global_summary_agent_chat`、`global_summary_agent_report` |
| 直接 Prompt | 无；prompt 在 chat/report 子 Agent 上 |
| 直接 Tool | 无；report 模式有 HTML 清洗、样式注入、数据来源拼接等后处理函数 |

主流程里常见调用点是：`chat_workflow` 多步骤执行完成后调用 `pre_process_report_expert(...)`，再 `call(callee="global_summary_agent", arguments=arguments)`；`report_workflow` 也会走同样的 `global_summary_agent`，只是输出后还会 `flow_output(...)` 保存 HTML/TXT 到 OSS。

## 2. 调用流程

```text
data_executer 执行所有图节点
  ↓
process_and_summary_agent 按步骤保存附件
  ├─ plan_attachment[0]["summary_expert"]
  ├─ plan_attachment[1]["summary_expert"]
  └─ ...
  ↓
pre_process_report_expert
  ├─ 遍历 plan_attachment
  ├─ 收集 summary_expert → report_segment
  ├─ 收集 GraphPlan.title → outline_full
  └─ 生成 query="用户的输入为：..."
  ↓
global_summary_agent
  ├─ get_input_mode: type=1 → chat，type=2 → report
  ├─ _build_merged_query: 把大纲和每步总结按 index 对齐
  ├─ chat  → global_summary_agent_chat   → Markdown
  └─ report → global_summary_agent_report → HTML + 后处理 + 数据来源
```

`global_summary_agent` 自己不直接请求 LLM。它先在 `global_summary_workflow` 里判断模式，然后把整理好的 `merged_query`、`now_time`、`master_query` 和 `outline_full` 传给真正请求模型的子 Agent。chat 模式调用 `global_summary_agent_chat`，底层是 `BackupChatAgent`；report 模式调用 `global_summary_agent_report`，底层是 `GlobalSummaryAgent`。

## 3. 输入整理例子

假设 planner 生成了两个步骤，并且每个步骤已经由 `process_and_summary_agent` 写入了 `summary_expert` 附件：

```json
{
  "title": ["销售变化概览", "渠道拆解"],
  "plan_attachment": [
    {
      "summary_expert": [
        "近7日GMV为1200万，较上周下降8.5%，主要下降来自移动端。<datasource>1</datasource>"
      ]
    },
    {
      "summary_expert": [
        "POP渠道GMV下降12.3%，自营渠道下降3.1%，POP是主要拖累项。<datasource>2</datasource>"
      ]
    }
  ]
}
```

`pre_process_report_expert` 会把它整理成传给 `global_summary_agent` 的 arguments：

```json
{
  "report_segment": [
    ["近7日GMV为1200万，较上周下降8.5%，主要下降来自移动端。<datasource>1</datasource>"],
    ["POP渠道GMV下降12.3%，自营渠道下降3.1%，POP是主要拖累项。<datasource>2</datasource>"]
  ],
  "outline_full": ["销售变化概览", "渠道拆解"],
  "query": "用户的输入为：分析近7日GMV下降原因"
}
```

进入 `global_summary_workflow` 后，`_build_merged_query` 不会把两段总结简单拼接，而是按步骤保留结构，整理成模型最终看到的主体输入：

```text
===== 报告大纲与分段报告内容（按子步骤分段对应给出） =====
【子步骤 1 大纲】
销售变化概览

【经过agent分析得到这段报告】：
近7日GMV为1200万，较上周下降8.5%，主要下降来自移动端。<datasource>1</datasource>

========== 分段分隔符 ==========

【子步骤 2 大纲】
渠道拆解

【经过agent分析得到这段报告】：
POP渠道GMV下降12.3%，自营渠道下降3.1%，POP是主要拖累项。<datasource>2</datasource>
===== END =====
```

这个 `merged_query` 会作为 `query` 传给 `global_summary_agent_chat` 或 `global_summary_agent_report`。同时，system prompt 里还会通过 `${now_time}` 和 `${master_query}` 填入当前时间和用户原始总诉求，让模型知道报告是在什么时间、围绕哪个总问题生成的。

## 4. chat 和 report 的差异

chat/report 由 `shared_data["type"]` 决定：`type=1` 是 chat，`type=2` 是 report。chat 模式面向对话窗口，调用 `global_summary_agent_chat`，使用 `get_global_summary_prompt_md`，输出 Markdown，并且 workflow 里不做 `clean_model_output`、样式注入和数据来源拼接。

report 模式面向正式报告，调用 `global_summary_agent_report`，使用 `get_global_summary_prompt_html`，输出 HTML。模型返回后会继续执行 `clean_model_output`、`inject_style_to_html`，再从 `shared_data["workspace_file_dict"]` 读取 `process_data_id`、`oss_url`、`file_name`，整理成数据来源附加到 HTML。如果 HTML 输出长度小于原始 Markdown 分段内容的 10%，会认为模型输出异常，直接用 `get_html_content(markdown_report)` 把原始 Markdown 转成 HTML 兜底。

| 模式 | 子 Agent | Prompt | 输出 | 后处理 |
| --- | --- | --- | --- | --- |
| chat | `global_summary_agent_chat` | `get_global_summary_prompt_md` | Markdown | 无额外清洗，流式返回 |
| report | `global_summary_agent_report` | `get_global_summary_prompt_html` | HTML | 清洗输出、注入样式、追加数据来源、长度异常兜底 |

## 5. 子 Agent 与 Prompt

| 对象 | 功能 | Prompt / 模型 |
| --- | --- | --- |
| `global_summary_agent` | 判断 chat/report，构造 `merged_query`，分派到具体总结子 Agent。 | 无直接 prompt，注册时 `llm_model="workflow_llm"` 只是 WorkflowAgent 必填配置。 |
| `global_summary_agent_chat` | 生成聊天场景 Markdown 全局总结。 | Prompt：`get_global_summary_prompt_md`，本地回退文件是 `applications/summary_agent/prompt/global_summary_prompt_md.py`；LLM 是 `global_summary_llm_stream`，有 backup。 |
| `global_summary_agent_report` | 生成报告场景 HTML 全局总结。 | Prompt：`get_global_summary_prompt_html`，本地回退文件是 `applications/summary_agent/prompt/global_summary_prompt_html.py`；LLM 是 `global_summary_llm`，有 backup。 |

这两个 prompt 的共同约束是：只能基于“待总结的分段报告内容”提炼，不允许重新计算或编造数据。report 版本额外强调 HTML 输出、Markdown 表格完整转换、`<datasource>` 转 `<sup>`、`<jmtCharts>` 图表保留等格式要求。

## 6. 排查边界

最终报告缺内容时，优先查 `plan_attachment[*]["summary_expert"]`，而不是先改 `global_summary_agent` 的 prompt。因为 `global_summary_agent` 不会重新取数，也不会重新读 `process_expert.output_data` 做二次分析；如果某个步骤的 `summary_expert` 为空、没有保存成功，或者只保存了错误提示，全局总结只能基于这些已有片段继续汇总。

如果 report 模式没有数据来源，检查 `shared_data["workspace_file_dict"]` 是否存在，以及里面是否有 `process_data_id`、`oss_url`、`file_name`。这些字段来自前面数据处理工具写入 workspace 的结果，全局总结只是读取并追加展示，不负责创建这些文件。

`global_summary_agent` 属于 [master-agent.md](./master-agent.md) 的最终输出子树；单步骤执行和 `summary_expert` 附件写入过程见 [process-and-summary-agent.md](./process-and-summary-agent.md)。独立 `/chat/summarize` 总结块不是这条主流程，已单独放到 [chat-summarize-agent.md](./chat-summarize-agent.md)。
