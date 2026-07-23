# Text2SQL Prompt 分析

本文分析 Text2SQL Agent 当前代码路径实际使用的 prompt。完整原文见 [Text2SQL Prompt 原文](text_2_sql_prompts_raw.md)。从代码绑定看，当前主链路实际使用 14 个 prompt；附件里额外包含 `llm_sql_fun_transform_prompt`，但当前 Text2SQL 主链路未发现引用；`llm_dataset_dsl2sql_prompt` 在 `laf_instance.py` 中有 getter，但本次附件没有提供，代码主链路也未发现实际绑定。

## 1. Prompt 使用总览

| Prompt 文档 | 使用 Agent / 调用点 | 所属阶段 | 当前状态 |
| -- | -- | -- | -- |
| [SQL 生成 Prompt](#llm_dataset_sql_prompt) | `llm_dataset_sql_agent` | SQL 生成 | 实际使用 |
| [SQL 纠错 Prompt](#llm_dataset_sql_correction_prompt) | `llm_dataset_sql_correction_agent` | SQL 执行失败后纠错 | 实际使用 |
| [SQL 默认条件补全 Prompt](#llm_sql_cond_infill_prompt) | `llm_sql_cond_infill_agent` | 展示 SQL 后处理 | 实际使用 |
| [SQL 多路选举 Prompt](#llm_dataset_sql_election_prompt) | `llm_dataset_sql_election_agent` | 多路 SQL 结果选举 | 实际使用 |
| [对话改写 Prompt](#llm_conv_rewriter_prompt) | `llm_conv_rewriter_agent` | 多轮追问改写 | 实际使用 |
| [澄清判断 Prompt](#llm_query_clarify_prompt) | `llm_query_clarify_agent` | 信息缺失判断 | 实际使用 |
| [DSL 查询 Prompt](#dsl_react_agent_prompt) | `llm_dsl_react_agent_*` | DSL ReAct 查询 | 实际使用 |
| [DSL 分析查询 Prompt](#dsl_react_analysis_agent_prompt) | `llm_dsl_react_analysis_agent_*` | 分析型 DSL ReAct 查询 | 实际使用 |
| [DSL 多路选举 Prompt](#llm_dsl_path_election_prompt) | `llm_dsl_path_election_agent` | 多路 DSL 结果选举 | 实际使用 |
| [总结 Prompt](#summary_prompt) | `summary_agent` | 查询结果总结 | 实际使用 |
| [总结润色 Prompt](#summary_polish_prompt) | `summary_polish_model` | 总结后处理 | 实际使用 |
| [Python 纠错 Prompt](#python_code_correction_prompt) | `python_code_correction_agent` | summary Python 代码失败后修复 | 实际使用 |
| [SQL 字段角色判断 Prompt](#dim_metric_judge_prompt) | `dim_metric_judge_agent` | 图表维度/指标识别 | 实际使用 |
| [DSL 字段角色判断 Prompt](#dsl_metric_judge_prompt) | `dsl_dim_metric_judge_agent` | DSL 图表维度/指标识别 | 实际使用 |
| [SQL 函数重构 Prompt](#llm_sql_fun_transform_prompt) | 未发现主链路引用 | 历史/备用 SQL 后处理 | 附件存在但当前未使用 |
| `llm_dataset_dsl2sql_prompt` | 仅发现 getter | 未确认 | 附件缺失且主链路未绑定 |

## 2. SQL 链路 Prompt

<a id="llm_dataset_sql_prompt"></a>
### SQL 生成 Prompt

原文见 [llm_dataset_sql_prompt](text_2_sql_prompts_raw.md#llm_dataset_sql_prompt)。这个 prompt 是 SQL 链路的核心，约束模型基于用户 query、数据集 schema、业务知识、当前日期生成 SQL。代码侧会在调用前注入 `prompt_query`、`prompt_dataset_name`、`prompt_dataset_schema`、`prompt_dataset_business_knowledge`、`prompt_date`、`prompt_date_week`。模型输出不是自由文本，而是 JSON 协议，解析器会提取 `think-process`、`clarification-process`、`sql-self-check` 和 `sql`。

这个 prompt 里的关键约束是聚合字段必须使用 `AGG()`、基础字段使用 `SUM/COUNT/AVG` 等常规聚合、禁止把指标字段嵌入 `CASE WHEN` 等表达式、用户未指定分组时不要冗余分组。代码还会做二次校验：SQL 必须包含 `from` 和数据集表名，聚合字段如果没有按聚合方式使用会触发一次重试。因此它是“prompt 约束 + 代码解析校验”共同保证 SQL 合规。

在多路 SQL 场景下，每一路都会独立使用这份 prompt 生成 SQL。因为输入的 `Text2SQLRequest` 和 `DatasetKnowledge` 相同，如果多个路径实际绑定的是同一个 Agent、同一个模型和同一套温度参数，那么生成结果可能高度相似；当前代码没有在执行前对相同 SQL 去重。多路带来的主要价值，是在模型输出存在轻微差异、某一路执行失败后可独立纠错、或者候选 SQL 口径不同的情况下，通过后续执行校验和选举筛出更可靠的一条。完整链路见 [Text2SQL SQL 多路生成、校验、纠错与选举详解](text_2_sql_sql_consensus.md)。

<a id="llm_dataset_sql_correction_prompt"></a>
### SQL 纠错 Prompt

原文见 [llm_dataset_sql_correction_prompt](text_2_sql_prompts_raw.md#llm_dataset_sql_correction_prompt)。它复用 SQL 生成的上下文，但额外注入 `prompt_bad_sql` 和 `prompt_sql_error_msg`。触发条件是 SQL 执行失败，且错误不是超时、内存等不适合 LLM 修复的问题。解析器复用 `llm_dataset_sql_agent_func_parse_llm_response()`，所以输出协议必须和 SQL 生成一致。

这份 prompt 的作用不是重新理解用户问题，而是在已有 SQL 和 DB 错误基础上做最小修复。代码中还会先跑工程纠偏，例如引号问题；只有仍需修复时才调用这个 Agent。纠错是否触发由执行结果决定：成功 SQL 不纠错，超时和内存类失败一般也不交给 LLM 纠错，其他 DB 异常才会进入动态纠错。当前 `llm_dataset_sql_correction_agent` 的 `max_react_rounds=1`，外层 `StatefulRetryExecutor(0, 0)` 不配置额外重试，因此它不是无限 ReAct，而是失败后一次保守修复再执行。

<a id="llm_sql_cond_infill_prompt"></a>
### SQL 默认条件补全 Prompt

原文见 [llm_sql_cond_infill_prompt](text_2_sql_prompts_raw.md#llm_sql_cond_infill_prompt)。它用于 `llm_sql_cond_infill_agent`，输入是已生成的展示 SQL 和查询接口返回的默认条件表达式。这个 prompt 不参与真正的数据查询，只修改前端展示用的 SQL，让权限、默认行级条件等隐含条件也能出现在 SQL 卡片里。

代码侧只从模型输出中提取 `<sql>...</sql>`，因此 prompt 的输出格式要求很硬。如果模型没有输出 `<sql>` 标签，解析会失败。

<a id="llm_dataset_sql_election_prompt"></a>
### SQL 多路选举 Prompt

原文见 [llm_dataset_sql_election_prompt](text_2_sql_prompts_raw.md#llm_dataset_sql_election_prompt)。当 LAF 配置了多个 SQL 路径时，系统会并发生成和执行多个候选 SQL，再把每个候选的 SQL 和样例结果拼成 `prompt_dataset_election` 交给这个 prompt 判断。它的输出只需要 `<id>...</id>`，对应最终被选中的 `PathState.path_id`。

这类 prompt 关注“候选方案是否真正回答用户问题”，而不是再生成 SQL。它的价值在于把多个可执行结果做语义裁决，尤其适合 SQL 都执行成功但口径、粒度或字段选择不同的情况。

需要注意，LLM 选举不是第一道质量关。默认策略链会先过滤失败路径，再用工程规则检查 `isAgg=1` 字段是否被正确聚合；只有仍剩多条候选时，才把候选 SQL 和前 5 行执行结果交给 `llm_dataset_sql_election_agent`。如果 LLM 选举失败或输出中没有 `<id>`，系统还会用“优先选有数据结果”的工程策略兜底。因此，多路 SQL 的质量控制是 prompt、解析器、远程执行、纠错和策略链叠加形成的，而不是只靠 election prompt 投票。

## 3. 对话改写与澄清 Prompt

<a id="llm_conv_rewriter_prompt"></a>
### 对话改写 Prompt

原文见 [llm_conv_rewriter_prompt](text_2_sql_prompts_raw.md#llm_conv_rewriter_prompt)。它只在 `ge_c_query_agent` 交互式链路中使用，用于基于历史问答、当前 query、数据集 schema 和业务知识判断当前问题是新话题还是追问，并生成 `rewritten_query`。代码会读取模型输出中的 `intent`，如果是非查询意图会直接抛不支持；如果不是新话题，会把 `text_2_sql_request.query` 替换为 `rewritten_query` 并重新召回数据集。

这和 Datagent 的 `rewritten_query` 思路类似，改写本身靠 prompt 让模型生成，代码负责读取并使用。

<a id="llm_query_clarify_prompt"></a>
### 澄清判断 Prompt

原文见 [llm_query_clarify_prompt](text_2_sql_prompts_raw.md#llm_query_clarify_prompt)。它用于判断问题是否缺少必要信息，例如指标、时间、主体或查询粒度不明确。代码通过 `query_clarify_validator()` 读取 `need_clarification` 和 `clarification_reason`；如果需要澄清，`ge_c_query_agent` 会直接把澄清原因发给用户并结束本轮，不再进入 SQL/DSL 查询。

这个 prompt 是否生效还受 `app_config.clarify_enabled` 控制。即使 prompt 配置存在，LAF 关闭澄清开关时也不会调用。

## 4. DSL 链路 Prompt

<a id="dsl_react_agent_prompt"></a>
### DSL 查询 Prompt

原文见 [dsl_react_agent_prompt](text_2_sql_prompts_raw.md#dsl_react_agent_prompt)。它用于普通 DSL 数据集的 ReAct 查询，工具主要是 `fetch_data_and_save_file` 和 `pandas_repl`。Prompt 强调字段必须来自数据集元数据、聚合要下沉到 DSL 查询层、时间范围要显式推导、禁止编造维值或文件名，最终输出包含 `status`、`message`、`total`、`file_name` 等字段的 JSON。

这条链路和 SQL 生成不同：模型不是直接输出 SQL，而是通过工具取数、保存文件、必要时用 pandas 做二次加工。最终 `invoke_dsl_agent()` 会读取文件结果并转换成统一 `PathState`。

<a id="dsl_react_analysis_agent_prompt"></a>
### DSL 分析查询 Prompt

原文见 [dsl_react_analysis_agent_prompt](text_2_sql_prompts_raw.md#dsl_react_analysis_agent_prompt)。它用于分析型 DSL 数据集，整体协议和普通 DSL prompt 相近，但更强调分析过程、结果解释和多步取数。代码通过 `DatasetType.ANALYTICAL_DATASET` 判断是否选择 `llm_dsl_react_analysis_agent_i`。

需要注意的是，DSL prompt 明确限制“最多 5000 条样本、不能基于非全量样本做错误聚合”，这和选举 prompt 中的评估规则相互呼应。

<a id="llm_dsl_path_election_prompt"></a>
### DSL 多路选举 Prompt

原文见 [llm_dsl_path_election_prompt](text_2_sql_prompts_raw.md#llm_dsl_path_election_prompt)。它和 SQL 多路选举类似，但评估对象是 DSL 候选方案，重点判断字段语义、查询粒度、过滤条件、排序限量和执行结果质量。输出同样是 `<id>...</id>`。

这份 prompt 中有一个重要原则：所有分组聚合应在 DSL 层完成，不能先查明细再用 pandas 聚合。它能过滤掉“结果看起来有数据，但取数粒度错误”的候选路径。

## 5. 总结、图表与 Python Prompt

<a id="summary_prompt"></a>
### 总结 Prompt

原文见 [summary_prompt](text_2_sql_prompts_raw.md#summary_prompt)。它用于 `summary_agent`，输入包含用户问题、业务知识、展示 SQL、查询结果样例、当前日期和内存 DataFrame 变量名。Prompt 允许模型在需要复杂计算时调用工具，但当前 `ge_c_query_oxy_space.py` 的 `additional_prompt` 明确要求：由 `summary_agent` 自己生成 `python_code`，再调用 `python_execute_tool` 执行，不要调用旧的 `summary_dataframe_analysis_tool` 和 `llm_python_code_agent`。

代码侧对 summary 输出很宽容：可以是工具调用 JSON、`{"summary": "..."}` JSON，也可以是纯文本。总结完成后 workflow 会强制调用 `summary_format_tool` 做数字占位符渲染，所以格式化不依赖模型主动调用工具。

<a id="summary_polish_prompt"></a>
### 总结润色 Prompt

原文见 [summary_polish_prompt](text_2_sql_prompts_raw.md#summary_polish_prompt)。它不是 ReActAgent prompt，而是 summary 后处理模型调用的模板。它的职责是修正表达和格式，例如单位、百分数、段落结构，但要求不改原始数据和事实口径。

这一步位于 `summary_format_tool` 之后，用于提升最终展示文本的可读性。

<a id="python_code_correction_prompt"></a>
### Python 纠错 Prompt

原文见 [python_code_correction_prompt](text_2_sql_prompts_raw.md#python_code_correction_prompt)。当 `python_execute_tool` 执行 summary 生成的 Python 代码失败时，会把用户需求、原始代码、错误信息和可用 DataFrame 名称渲染进该 prompt，让模型返回修复后的 Python 代码块。

代码有安全校验和一次纠错重试。它要求执行结果必须赋值给 `result`，并且结果必须是 DataFrame。

<a id="dim_metric_judge_prompt"></a>
### SQL 字段角色判断 Prompt

原文见 [dim_metric_judge_prompt](text_2_sql_prompts_raw.md#dim_metric_judge_prompt)。它用于 SQL 查询后的图表准备阶段，根据用户问题和 SQL 判断结果字段哪些是维度、哪些是度量，以及度量类型。结果会进入 `PureTableChart` 和 `ge_c_query_chart_code_tool`，影响 C 端图表渲染协议。

这说明图表配置不是由 SQL Agent 直接决定，而是由字段角色判断 prompt 和图表配置服务共同完成。

<a id="dsl_metric_judge_prompt"></a>
### DSL 字段角色判断 Prompt

原文见 [dsl_metric_judge_prompt](text_2_sql_prompts_raw.md#dsl_metric_judge_prompt)。它用于 DSL 查询结果字段判断，输入更偏向字段信息和数据结果，而不是 SQL 语句。其输出同样服务于图表渲染。

## 6. 未使用或缺失 Prompt

<a id="llm_sql_fun_transform_prompt"></a>
### SQL 函数重构 Prompt

原文见 [llm_sql_fun_transform_prompt](text_2_sql_prompts_raw.md#llm_sql_fun_transform_prompt)。附件中提供了这个 prompt，它的职责是把 SQL 中聚合属性字段的聚合函数重构成 `AGG()`。但当前代码检索没有发现 `llm_sql_fun_transform_prompt`、`fun_transform` 或相关 getter 的实际引用，并且 `SqlPathRunner.query_and_sql_visualize()` 中有注释说明“已移除 SQL 函数变换/可视化环节”。因此它更像历史遗留或备用配置，不应算作当前 Text2SQL 主路径必需 prompt。

### `llm_dataset_dsl2sql_prompt`

`llm_dataset_dsl2sql_prompt` 在 `laf_instance.py` 中有 getter，但本次附件没有找到配置，代码主链路也没有发现实际绑定。当前 DSL 主链路使用的是 [DSL 查询 Prompt](#dsl_react_agent_prompt)、[DSL 分析查询 Prompt](#dsl_react_analysis_agent_prompt) 和 [DSL 多路选举 Prompt](#llm_dsl_path_election_prompt)。
