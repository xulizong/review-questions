# Text取数Agent 分析

`Text取数Agent` 是主执行流中的“单步骤数据处理 Agent”。它不负责生成整份分析计划，而是接收 planner 已经写好的一个步骤，根据数据源选择取数或加工工具，把工具结果回灌给 LLM，直到通过 `summary_tool` 产出该步骤的结论。

它的核心定位是 ReAct 编排器，不是单纯的 Text2SQL Agent：

```text
Text取数Agent
  ├─ 理解当前计划步骤
  ├─ 选择当前业务工具
  ├─ 组织工具入参
  ├─ 读取工具返回的数据
  ├─ 必要时继续取数或进行 Python 加工
  └─ 通过 summary_tool 生成节点结论
```

`Text取数Agent` 的注册位置是 `applications/master_agent/run_master_0130.py`，Agent 实现是 `oxygent/oxy/agents/progressive_react_agent.py::ProgressiveReActAgent`，主流程调用入口是 `applications/data_process/agents/data_process_workflow.py::_get_data_result`，LLM 返回解析器是 `applications/data_process/util/llm_response_parser.py::func_parse_llm_response_for_tool`。

## 1. 在主流程中的位置

```text
data_planner
  ↓ 生成 GraphPlan 步骤
data_executer
  ↓ 选中当前可执行步骤
process_and_summary_agent
  ↓
process_expert
  ↓ 整理 query、数据源、知识和 workspace
Text取数Agent
  ↓ 多轮调用 text2data_tool / py_coder_tool
summary_tool
  ↓ 返回当前步骤 conclusion
summary_expert
```

`Text取数Agent` 每次只处理一个 GraphPlan 节点。多个步骤之间的依赖、并发和状态由 `data_executer` 处理，不是该 Agent 的职责。`process_and_summary_agent` 如何定位当前步骤并写回附件，见 [process-and-summary-agent.md](./process-and-summary-agent.md)。

## 2. 进入 Agent 前的准备

`process_expert` 先从当前 GraphPlan 节点取得标题和 `step_command`，拼成本次 query。例如：

```text
苹果成交金额趋势

查询 2026-07-01 至 2026-07-14 品牌为苹果的每日成交金额，
并计算 7 日移动平均。
```

随后会把 `shared_data["user_data"]` 整理成线上 dataset 和本地 file 两类上下文，再根据当前场景生成 Prompt。传给 `Text取数Agent` 的关键参数如下：

| 参数 | 作用 |
| --- | --- |
| `query` | 当前计划步骤的完整指令 |
| `dataset_list` | 本步骤可见的线上数据集名称 |
| `dataset_id_list` | `text2data_tool` 可使用的 dataset ID |
| `dataset_markdown` | 数据集说明、字段信息和业务描述 |
| `file_list` / `file_markdown` | 用户上传文件及其字段说明 |
| `knowledge` | 系统、关键词、用户和语义层知识的拼接结果 |
| `workspace` | 当前步骤可继续读取的临时数据文件 |
| `now_time` | 当前日期和时间背景 |
| `prompt` | `build_multi_data_prompt(...)` 动态生成的 Prompt 模板 |
| `tool_list` | 本次场景允许使用的业务工具 |
| `current_tool_name` | 当前这一轮允许调用的业务工具 |
| `current_tool_description` | 当前工具的完整参数和使用说明 |

例如线上 dataset 场景的入参形态是：

```json
{
  "query": "苹果成交金额趋势\n查询2026-07-01至2026-07-14的每日成交金额，并计算7日移动平均。",
  "dataset_list": ["订单明细"],
  "dataset_id_list": ["1001"],
  "dataset_markdown": "订单明细的描述和字段信息",
  "file_list": [],
  "file_markdown": "",
  "knowledge": "四类知识拼接结果",
  "workspace": "没有临时数据。",
  "prompt": "动态 Prompt 模板",
  "tool_list": ["text2data_tool", "py_coder_tool", "summary_tool"],
  "current_tool_name": "text2data_tool",
  "current_tool_description": "text2data_tool 的详细说明"
}
```

## 3. Prompt 如何组装

`Text取数Agent` 注册时的 `prompt` 是空字符串，真正使用的 Prompt 由 `applications/data_process/prompts/multi_data_prompt_builder.py::build_multi_data_prompt` 按当前可用工具动态生成。`ProgressiveReActAgent` 每一轮都会用入参填充 `${dataset_markdown}`、`${knowledge}`、`${workspace}`、`${current_tool_description}` 等 System Prompt 占位符；`query` 不是其中的占位符，而是另外组装成“我的最新问题是……”的 user 消息。

Prompt 的核心内容可以归纳为：

| Prompt 部分 | 对 LLM 的约束 |
| --- | --- |
| 角色与时间 | 作为数据分析程序员，严格完成当前步骤，不额外扩展分析 |
| 工具规则 | 每次只调用一个业务工具；切换前先调用 `get_tool_documentation` |
| 取数要素 | 向 `text2data_tool` 传递完整的时间、指标、维度、筛选、排序和计算要求 |
| 数据语境 | 注入 dataset、file、workspace 和知识，禁止编造字段和口径 |
| 工具选择 | 线上 dataset 先用 `text2data_tool`；本地文件、结果合并和复杂计算使用 `py_coder_tool` |
| 最终输出 | 所有数字必须来自工具结果，最后必须通过 `summary_tool` 结束 |

Prompt 中一个非常重要的边界是：LLM 可以组织业务查询和 Python 代码，但不能在没有工具返回的情况下自行编造数据。不同聚合粒度通常需要分开调用 `text2data_tool`，取得的完整结果会落到 workspace，而不是全部堆进 Prompt。

## 4. 四类知识如何进入 Prompt

`process_expert` 在调用 `Text取数Agent` 前执行 `search_knowledge(query, oxy_request)`，返回结果作为 `${knowledge}` 放入 Prompt。四类知识的责任不同：

| 知识类型 | 解决的问题 | 当前 Text 取数链路的实际来源 |
| --- | --- | --- |
| 系统知识 | 这类问题通常怎样分析 | 当前 `enable=False`，不会进入 Text 取数 Prompt |
| 关键词知识 | “同比、环比、贡献度”等运算语义 | DUCC `keyword_knowledge` 配置，无配置时使用本地默认词典，按最长词优先确定性匹配 |
| 用户知识 | 当前用户或业务方定义的口径 | 按 `user_knowledge_id` 从用户知识文档中向量召回片段 |
| 语义层知识 | 业务词与当前数据集字段、枚举值和同义词的关系 | 直接复用 Master 前面写入 `shared_data["knowledge_cache"]` 的结果 |

例如用户询问：

```text
计算本周苹果品牌成交金额的同比贡献
```

实际放入 Prompt 的知识可能是：

```text
### 关键词知识库：
检测到以下业务术语及其定义：
1. 同比贡献 [定义：（子维度本期值-子维度去年同期值）/总体去年同期值×100%]

### 用户知识库：
成交金额按支付成功金额计算，不扣除退款。

### 语义层知识库：
对于数据集：【订单明细】
成交金额的同义词：GMV、实付金额
品牌名称的相关枚举值：苹果、华为、小米
```

系统知识偏向分析框架，当前在 Text 取数链路关闭；关键词知识定义通用运算口径；用户知识补充用户专属的业务口径；语义层知识把业务语言和当前数据集联系起来。

### 4.1 字段 AGG 与关键词知识的边界

表字段配置解决的是基础值如何聚合，例如：

```text
成交金额 → SUM(pay_amount)
订单量   → COUNT(DISTINCT order_id)
品牌、日期 → 分组或筛选字段
```

“同比、环比、同比贡献、同期、漏斗”不是表中的真实字段，而是对多个基础聚合结果进行组合的分析运算规则。例如“苹果品牌同比贡献”需要先得到：

```text
A = 苹果品牌本期成交金额
B = 苹果品牌去年同期成交金额
C = 全部品牌去年同期成交金额

同比贡献 = (A - B) / C × 100%
```

`SUM(pay_amount)` 只能帮助得到 A、B、C，无法单独表达对比时间、分母和最终公式，所以当前项目用表外的关键词知识约束 Agent 的理解。

如果未来语义层已经统一配置“基础指标公式 + 时间运算符 + 派生指标公式 + 同义词”，那么关键词匹配只需负责把用户表达映射到运算符，不应再另外维护一套重复公式。

## 5. 工具选择和切换

普通主流程的工具路径如下：

| 数据场景 | 初始工具 | 后续可用工具 | 结束方式 |
| --- | --- | --- | --- |
| 存在线上 dataset | `text2data_tool` | `py_coder_tool` | `summary_tool` |
| 只有用户上传 file | `py_coder_tool` | 继续使用 `py_coder_tool` | `summary_tool` |

`get_tool_documentation` 是 Agent 内部系统工具。它不执行业务逻辑，只是将 `current_tool_name` 和 `current_tool_description` 切换到目标工具。例如从 SQL 取数切换到 Python 加工：

```json
{
  "tool_name": "get_tool_documentation",
  "arguments": {
    "target_tool": "py_coder_tool"
  }
}
```

切换完成后，下一轮 LLM 才会看到 `py_coder_tool` 的详细协议并调用它。转入最终输出前，同样需要切换到 `summary_tool`。

## 6. text2data_tool：线上数据集取数

`text2data_tool` 的入参是：

```json
{
  "query": "面向 Text2SQL 服务的完整自然语言查询",
  "dataset_id": "1001",
  "title": "该次取数结果的标题"
}
```

`Text取数Agent` 不会自己生成物理 SQL。`applications/data_process/tools/text2data_tools.py::text2data_tool` 会将 query 和 dataset ID 发送给 `config.text2sql_agent.server_url`，当前生产配置指向：

```text
http://datagent-text2sql.jd.local/api/chat
```

外部服务负责：

```text
自然语言查询
  → 匹配真实字段
  → 生成 SQL / DSL
  → 执行查询
  → 返回数据和 SQL 信息
```

如果当前只有一个 dataset，工具会把 LLM 给出的 ID 覆盖成这个唯一合法 ID；多个 dataset 时则校验该 ID 是否存在于 `dataset_id_list`。

例如：

```json
{
  "tool_name": "text2data_tool",
  "arguments": {
    "query": "查询2026-07-01至2026-07-14品牌为苹果的每日成交金额，按日期升序汇总。",
    "dataset_id": "1001",
    "title": "苹果每日成交金额"
  }
}
```

工具将完整 DataFrame 写入 workspace，获得新的 `process_data_id`。给 LLM 的 observation 只保留可控长度的展示内容，后续需要完整数据时，`py_coder_tool` 通过 workspace URL 重新读取，不依赖 Prompt 中的截断数据。

## 7. py_coder_tool：本地文件和二次加工

`py_coder_tool` 主要处理三类情况：读取用户上传的 file，合并多次 `text2data_tool` 结果，执行 SQL 不适合表达的复杂计算。它的调用参数包括 `think`、`code`、`title` 和 `output_var`。

例如前一轮取得每日成交金额后，计算 7 日移动平均：

```json
{
  "tool_name": "py_coder_tool",
  "arguments": {
    "think": "读取苹果每日成交金额文件，按日期排序并计算7日移动平均。",
    "code": "import pandas as pd\ndf = pd.read_csv('workspace中的文件URL')\ndf['日期'] = pd.to_datetime(df['日期'])\ndf = df.sort_values('日期')\ndf['7日移动平均'] = df['成交金额'].rolling(7, min_periods=1).mean()\noutput_var = df",
    "title": "苹果成交金额7日移动平均",
    "output_var": "output_var"
  }
}
```

解析器会检查代码是否使用 `pd.read_csv` 或 `read_excel` 读取 workspace 数据。工具执行后要求 `output_var` 指向一个 DataFrame，再把结果写回 workspace，因此一个步骤内可以继续读取上一轮的加工结果并多次调用 `py_coder_tool`。

当前实现会先执行语法检查和危险操作检查，再把通过检查的 LLM 代码写入临时 `.py` 文件，在 `config.web.upload_tmp_dir/chat/{pin}/{group_id}/` 目录下通过 `compile + exec` 在当前服务进程中执行。它有执行前代码约束，但不是独立容器或进程级沙箱。

## 8. Progressive ReAct 多轮执行

`ProgressiveReActAgent` 每轮会重新生成 workspace 描述，并组织以下消息：

```text
System Prompt（动态 Prompt + 已填充占位符）
  + Text取数Agent 的短期记忆
  + 当前计划步骤和知识
  + workspace 文件描述
  + 本次 ReAct 前几轮的工具调用和 observation
```

然后调用 `text2data_llm`，主模型失败时切换到 `text2data_llm_backup`。当前最多执行 20 轮。LLM 返回会经过 `func_parse_llm_response_for_tool` 解析：

1. 如果返回中存在 `</think>`，丢弃该结束标签之前的思考内容，再提取 JSON。
2. 一轮出现多个 JSON 工具调用时返回解析错误，要求 LLM 重写。
3. 普通工具调用成功后，把工具结果加入 `react_memory`，继续下一轮。
4. 识别到 `summary_tool` 时，将其转成 Agent 最终答案并结束循环。

因此 `Text取数Agent` 不是一次 LLM 请求，而是“LLM 决定下一步 → 程序执行工具 → 结果回灌 LLM”的多轮循环。

## 9. summary_tool 和最终后处理

所有取数和加工完成后，LLM 需要通过 `summary_tool` 提交结论：

```json
{
  "tool_name": "summary_tool",
  "arguments": {
    "summary": "2026-07-01至2026-07-14，苹果品牌成交金额整体呈上升趋势。下表展示每日成交金额和7日移动平均……"
  }
}
```

`summary_tool` 在这里是最终输出协议。解析器识别它之后，对 summary 执行：

```text
节点结果校验
  → 数值溯源
  → 数字标签化和格式化
  → 较大 dataset ID 标签清理
  → Markdown 表格修复
  → 转成最终 ANSWER
```

总结中的业务数字必须能在前面的工具 observation 中找到来源。如果首次发现无来源数字，解析器会要求 Agent 重写 summary，而不是直接展示。

## 10. 贯穿示例

当前计划步骤是：

```text
查询 2026-07-01 至 2026-07-14 苹果品牌的每日成交金额，
并计算 7 日移动平均，以表格输出。
```

完整执行顺序是：

```text
1. process_expert 整理订单明细 dataset、知识和 Prompt
2. Text取数Agent 首轮调用 text2data_tool
3. Text2SQL 服务匹配字段、生成 SQL、执行并返回每日成交金额
4. text2data_tool 把完整结果写入 workspace，产生 process_data_id=1
5. Agent 调用 get_tool_documentation 切换到 py_coder_tool
6. py_coder_tool 读取 workspace 文件，计算7日移动平均
7. Python 结果再次写入 workspace，产生 process_data_id=2
8. Agent 切换到 summary_tool，提交最终结论
9. 解析器完成数值和 Markdown 后处理，结束 ReAct
```

如果用户只要求查询每日成交金额，不需要二次加工，路径会缩短为：

```text
text2data_tool → summary_tool
```

如果只有用户上传文件，路径是：

```text
py_coder_tool → summary_tool
```

## 11. Agent 输出如何回收

`Text取数Agent` 结束后，`process_expert` 会对比执行前后的 workspace ID，收集本次新增的所有 `process_data_id`；同时从 `react_memory` 逆序提取最后一次数据工具的展示结果。

返回结构是：

```json
[
  {
    "process_data_ids": ["1", "2"],
    "output_data": [
      {
        "data": "最后一次 text2data_tool 或 py_coder_tool 的展示结果"
      }
    ],
    "conclusion": "summary_tool 生成并经过后处理的节点结论"
  }
]
```

`process_data_ids` 包含本次过程生成的所有临时数据，但 `output_data` 只保留最后一次命中的数据工具展示结果。`conclusion` 是供后续 `summary_expert` 和 `global_summary_agent` 使用的文本结论。

## 12. 关键边界

`Text取数Agent` 负责“一个步骤内怎么取数和加工”，不负责计划拆分、图依赖调度和跨节点最终总结。`text2data_tool` 负责把自然语言转交外部 Text2SQL 服务，外部服务才负责字段映射、SQL 生成和执行；`py_coder_tool` 负责文件读取和二次计算；`summary_tool` 负责按协议提交可展示结论。
