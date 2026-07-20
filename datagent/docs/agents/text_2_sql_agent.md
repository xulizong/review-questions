# Text2SQL Agent

Text2SQL Agent 是 `AGENT=text-2-sql-agent` 时的 master agent，代码入口是 `modules/agents/text_2_sql_agents/main.py`，顶层 workflow 在 `modules/agents/text_2_sql_agents/text_2_sql_agent.py`。它负责自然语言到 SQL/DSL 查询的完整链路，但 master 自己不生成 SQL，而是根据 `shared_data.app`、`shared_data.scene`、`shared_data.type` 把请求分派到不同入口 Agent，再由入口 Agent 编排数据集召回、问题改写、澄清、SQL/DSL 生成、执行、纠错、选举、可视化和总结。

## 1. Master 路由结构

`text_2_sql_agent` 是 `is_master=True` 的 `WorkflowAgent`。外部 `/sse/chat` 或 `/api/chat` 请求进入后，`text_2_sql_agent_func_workflow()` 调用 `find_portal_agent_name(req)` 选择真正执行的 portal Agent。优先级是先读 `req.shared_data` 中的 `app`、`scene`、`type`，匹配不到时尝试把 `query` 当 JSON 解析，再从 JSON 中取同样三个字段；仍然匹配不到时默认进入 `ge_c_query_analysis_agent`。

| scene 匹配 | 入口 Agent | 使用场景 | 返回风格 |
| -- | -- | -- | -- |
| `("ge", "c", None)` | `ge_c_query_analysis_agent` | API 型 Text2SQL 查询，也是默认入口 | 返回 `Text2SQLResponse` 结构化结果 |
| `("all", "bi-e-query", None)` | `ge_c_query_analysis_agent` | BI-E 查询入口 | 返回 `Text2SQLResponse`，SQL 执行时补 `authGranularity=1` |
| `("all", "ge-c-query", None)` | `ge_c_query_agent` | 黄金眼 C 端交互式查询 | 发送 SSE 卡片、SQL、图表、总结 |
| `("all", "query-with-summary", None)` | `query_with_summary_agent` | 查询后直接总结 | 发送 SQL、数据、总结，不做完整图表渲染 |

顶层输出还有一层日志截断处理。`text_2_sql_agent_func_process_output()` 会根据 LAF 配置 `dataset_data_query_log_cut_off` 截短错误详情或大结果集，用于节点日志和 trace 展示；`func_format_output()` 会在最终响应时还原 `raw_output`，避免对真实业务返回造成截断。

## 2. Oxy Space 注册结构

`main.py` 先注册 Text2SQL 公共模型、数据集工具、SQL 生成/纠错/选举 Agent、portal Agent，然后再 extend 进问题改写、澄清、DSL、GE C Query、Query With Summary 的独立 `oxy_space`。因此这个 Agent 的能力不是单个文件注册完的，而是由多个 scene 模块拼装成一个总 `oxy_space`。

| 注册对象 | 类型 | 作用 | 关键依赖 |
| -- | -- | -- | -- |
| `text_2_sql_agent` | `WorkflowAgent` | master 路由 | `AgentPortalEnum` |
| `ge_c_query_analysis_agent` | `WorkflowAgent` | API 查询入口 | 数据集召回、SQL/DSL 路径、结构化响应 |
| `ge_c_query_agent` | `WorkflowAgent` | 交互式查询入口 | 改写、澄清、SQL/DSL、OSS、图表、总结 |
| `query_with_summary_agent` | `WorkflowAgent` | 查询后总结入口 | SQL 查询、数据消息、summary |
| `llm_dataset_sql_agent` | `ReActAgentProxy` | 生成标准 SQL | `llm_dataset_sql_prompt`、`convert_calendar` |
| `llm_dataset_sql_correction_agent` | `ReActAgentProxy` | 根据执行错误纠正 SQL | `llm_dataset_sql_correction_prompt`、`convert_calendar` |
| `llm_sql_cond_infill_agent` | `ReActAgentProxy` | 给展示 SQL 补默认条件 | `llm_sql_cond_infill_prompt` |
| `llm_dataset_sql_election_agent` | `ReActAgentProxy` | 多路 SQL 候选结果选举 | `llm_dataset_sql_election_prompt` |
| `single_path_sql_agent` | `WorkflowAgent` | 单条 SQL 路径执行器 | `SqlPathRunner` |
| `sql_path_election_strategy_agent` | `WorkflowAgent` | SQL 多路径选举入口 | `SqlPathElectionStrategyChain` |
| `llm_conv_rewriter_agent` | `ReActAgentProxy` | 基于历史对话改写问题 | `llm_conv_rewriter_prompt` |
| `llm_query_clarify_agent` | `ReActAgentProxy` | 判断是否需要澄清 | `llm_query_clarify_prompt` |
| `dsl_agent` | `WorkflowAgent` | DSL 查询路径入口 | `llm_dsl_react_agent_*` |
| `llm_dsl_react_agent_*` | `ReActAgent` | DSL 数据集查询 | `dsl_react_agent_prompt`、`pandas_repl`、`fetch_data_and_save_file` |
| `llm_dsl_react_analysis_agent_*` | `ReActAgent` | 分析型 DSL 数据集查询 | `dsl_react_analysis_agent_prompt`、`pandas_repl`、`fetch_data_and_save_file` |
| `llm_dsl_path_election_agent` | `ReActAgentProxy` | 多路 DSL 候选结果选举 | `llm_dsl_path_election_prompt` |
| `summary_agent` | `ReActAgentProxy` | 查询结果总结 | `summary_prompt`、`autobots_dataframe_fetch_tools`、`python_execute_tool` |
| `dim_metric_judge_agent` | `ReActAgentProxy` | SQL 结果字段维度/指标判断 | `dim_metric_judge_prompt` |
| `dsl_dim_metric_judge_agent` | `ReActAgentProxy` | DSL 结果字段维度/指标判断 | `dsl_metric_judge_prompt` |
| `python_execute_tool` | `WorkflowAgent` | 执行 summary 生成的 Python 代码 | `python_execute_code_tool`、`python_code_correction_agent` |

## 3. Prompt 来源和 Agent 对应关系

Text2SQL 的主要 prompt 不在仓库里写死，而是通过 `modules/agents/text_2_sql_agents/instance/laf_instance.py` 中的 `FunctionLafClient` 从 LAF 配置读取。LAF namespace 是 `datagent/config/text2sql_agent`，环境由 `application-text-2-sql-agent-*.yml` 中的 profile 决定。`success_callback()` 会在 LAF 更新成功后刷新模型配置和多 Agent 模型映射，所以 prompt 和部分模型策略是动态配置。Prompt 原文已整理到 [Text2SQL Prompt 原文](text_2_sql_prompts_raw.md)，逐项分析见 [Text2SQL Prompt 分析](text_2_sql_prompt_analysis.md)。SQL 生成、执行校验、纠错和多路选举的完整链路见 [Text2SQL SQL 多路生成、校验、纠错与选举详解](text_2_sql_sql_consensus.md)。

| Prompt 文档 | 使用 Agent | 主要约束 | 代码消费方式 |
| -- | -- | -- | -- |
| [SQL 生成 Prompt](text_2_sql_prompt_analysis.md#llm_dataset_sql_prompt) | `llm_dataset_sql_agent` | 根据 query、数据集 schema、业务知识生成 SQL，必要时可调用 `convert_calendar` | 解析 JSON，提取 `think-process`、`clarification-process`、`sql-self-check`、`sql` |
| [SQL 纠错 Prompt](text_2_sql_prompt_analysis.md#llm_dataset_sql_correction_prompt) | `llm_dataset_sql_correction_agent` | 根据 bad SQL 和 DB 错误信息修正 SQL | 复用 SQL Agent 的解析器 |
| [SQL 默认条件补全 Prompt](text_2_sql_prompt_analysis.md#llm_sql_cond_infill_prompt) | `llm_sql_cond_infill_agent` | 把查询接口返回的默认条件补进展示 SQL | 从 `<sql>...</sql>` 提取 SQL |
| [SQL 多路选举 Prompt](text_2_sql_prompt_analysis.md#llm_dataset_sql_election_prompt) | `llm_dataset_sql_election_agent` | 从多个 SQL 候选方案中选出最可信路径 | 从 `<id>...</id>` 提取 path id |
| [对话改写 Prompt](text_2_sql_prompt_analysis.md#llm_conv_rewriter_prompt) | `llm_conv_rewriter_agent` | 基于历史问答、schema、业务知识判断新话题/追问并生成 `rewritten_query` | 解析 JSON，读取 `intent` 和 `rewritten_query` |
| [澄清判断 Prompt](text_2_sql_prompt_analysis.md#llm_query_clarify_prompt) | `llm_query_clarify_agent` | 判断 query 是否信息不足，需要用户补充 | 解析 JSON，读取 `need_clarification` 和 `clarification_reason` |
| [DSL 查询 Prompt](text_2_sql_prompt_analysis.md#dsl_react_agent_prompt) | `llm_dsl_react_agent_*` | DSL 数据集 ReAct 查询规则 | JSON 工具调用或最终 `status=200/-1` |
| [DSL 分析查询 Prompt](text_2_sql_prompt_analysis.md#dsl_react_analysis_agent_prompt) | `llm_dsl_react_analysis_agent_*` | 分析型 DSL 数据集 ReAct 查询规则 | 同 DSL ReAct 解析器 |
| [DSL 多路选举 Prompt](text_2_sql_prompt_analysis.md#llm_dsl_path_election_prompt) | `llm_dsl_path_election_agent` | 从多个 DSL 候选结果中选出最可信路径 | 复用 `<id>` 选举解析 |
| [总结 Prompt](text_2_sql_prompt_analysis.md#summary_prompt) | `summary_agent` | 基于 query、SQL、结果样例、业务知识生成总结，必要时调用工具读取 DataFrame | 支持工具调用 JSON、`summary` JSON 和纯文本总结 |
| [总结润色 Prompt](text_2_sql_prompt_analysis.md#summary_polish_prompt) | `summary_polish_model` 直接调用 | 对 summary 做表达润色，不改事实口径 | workflow 在 summary 后显式调用 |
| [Python 纠错 Prompt](text_2_sql_prompt_analysis.md#python_code_correction_prompt) | `python_code_correction_agent` | 根据失败 Python 代码和错误信息修复代码 | 提取 Python 代码块 |
| [SQL 字段角色判断 Prompt](text_2_sql_prompt_analysis.md#dim_metric_judge_prompt) | `dim_metric_judge_agent` | 判断 SQL 查询结果字段的维度和指标角色 | 输出用于图表协议 |
| [DSL 字段角色判断 Prompt](text_2_sql_prompt_analysis.md#dsl_metric_judge_prompt) | `dsl_dim_metric_judge_agent` | 判断 DSL 查询结果字段的维度和指标角色 | 输出用于图表协议 |

仓库中还有本地 `text_2_dsl_agent_prompt`，绑定在 `text_2_dsl_react_agent` 上，主要描述工具调用格式、DSL 查询限制、`run_pandas` 二次处理和最终 JSON 输出格式。但当前主 DSL 查询入口更多使用 `dsl_agent` 及 LAF prompt 驱动的 `llm_dsl_react_agent_*` 实例池。

## 4. SQL 路径执行细节

SQL 路径的核心实现是 `SqlPathRunner.run()`。它先调用 `llm_dataset_sql_agent` 生成 `sql_info`，再调用 `query_dataset_data_tool` 执行 SQL。如果模型没有生成 SQL 但给出 `sql_build_clarification`，路径会返回 `USER_QUERY_CLARIFICATION` 状态；如果执行失败，会先尝试工程规则纠偏，例如 `MissingQuoteSQLCorrection` 处理引号问题，仍失败时再调用 `llm_dataset_sql_correction_agent` 让模型根据 DB 错误修 SQL 并重查。更细的 prompt、工具调用、重试和选举机制见 [Text2SQL SQL 多路生成、校验、纠错与选举详解](text_2_sql_sql_consensus.md)。

```text
SQL 路径
    ↓
llm_dataset_sql_agent 生成 sql_info
    ↓
query_dataset_data_tool 执行 SQL
    ↓
工程规则纠偏 MissingQuoteSQLCorrection
    ↓
llm_dataset_sql_correction_agent 纠错 SQL
    ↓
再次 query_dataset_data_tool
    ↓
PathState 记录 sql_info、exec_result、is_success、exception、latency
```

`llm_dataset_sql_agent` 的解析器要求 SQL 至少包含 `from`，并且 SQL 中要出现当前数据集表名。它还会检查聚合字段是否按聚合方式使用，如果没有满足聚合约束且尚未重试，会通过 `get_app_ducc_config().llm_sql_agg_rhetorical_prompt` 让 ReActAgent 重试。这个逻辑说明 SQL prompt 不是唯一约束，代码还会对模型输出做结构和业务规则校验。

多路 SQL 由 `SqlPathAgentParallelManager` 管理。LAF key `multi_sql_path_parallel_agents` 如果配置了多个 Agent 名称，workflow 会并发调用这些路径；每一路都独立完成 SQL 生成、远程执行、必要时纠错和再次执行。manager 通过 `asyncio.gather()` 等全部路径结束后，才把多个 `PathState` 交给 `sql_path_election_strategy_agent`。选举策略链先过滤失败路径，再检查聚合属性字段是否按 `AGG()` 规则使用，仍无法唯一确定时才调用 `llm_dataset_sql_election_agent`，最后用“有数据优先”的工程规则兜底。

以“上月各事业部的采购亏损金额分布及同比变化情况”为例，如果数据集里 `采购亏损金额` 是 `isAgg=1` 的聚合属性字段，生成 prompt 会要求 SQL 用 `AGG(\`采购亏损金额\`)` 分别计算本期和去年同期，再按 `事业部` 关联计算同比。若某一路直接使用 `SUM(\`采购亏损金额\`)`，即使 SQL 能执行，也会在 AGG 选举策略中处于劣势；若另一路因为日期类型或引号错误执行失败，会先走工程修复或 LLM 纠错，再以纠错后的执行结果参与选举。这个例子的完整时序和 SQL 形态见 [SQL 多路选举示例](text_2_sql_sql_consensus.md#7-结合例子看完整流程)。

## 5. DSL 路径执行细节

入口 Agent 会先调用 `dataset_info_tool` 查询数据集类型。如果 `DatasetType.is_dsl(dataset_info.get("dataset_type"))` 为 true，就走 DSL 路径，否则走 SQL 路径。DSL 路径由 `dsl_agent` 调用 `invoke_llm_dsl_react_agent()`，再根据数据集类型选择普通 `llm_dsl_react_agent_i` 或分析型 `llm_dsl_react_analysis_agent_i`。

DSL ReAct Agent 的工具是 `pandas_repl` 和 `fetch_data_and_save_file`。它通过工具查询数据并把结果写入文件，最终返回包含 `status`、`message`、`total`、`file_name` 的 JSON。`llm_dsl_react_agent_func_process_output()` 会把 ReAct 过程中的 think、查询 SQL 信息、pandas 代码和文件名整理成统一输出。`invoke_dsl_agent()` 再从文件中读取 `data` 和 `metadata`，构造成与 SQL 路径一致的 `PathState`，这样上层的图表和总结逻辑可以复用。

DSL 多路并发由 `multi_dsl_path_parallel_agents` 控制。为了避免多个 ReAct 路径共享同一个 LLM semaphore 导致准串行，`text_2_dsl_oxy_space.py` 注册了 `DSL_REACT_MODEL_POOL_SIZE = 3` 个独立 `dsl_react_agent_model_i` 和对应 Agent 实例。多路结果通过 `dsl_path_election_strategy_agent` 和 `llm_dsl_path_election_agent` 选出最终路径。

## 6. 三个入口流程

`ge_c_query_analysis_agent` 是结构化 API 路径。它把 query 转成 `Text2SQLRequest`，校验参数，调用 `call_back_dataset_knowledge()` 召回数据集 schema 和业务知识，再调用 `dataset_info_tool` 判断 SQL/DSL。SQL 数据集走 `SqlPathRunner` 或多路 SQL 选举；DSL 数据集走 `dsl_agent` 或多路 DSL 选举。最终 `build_text_2_sql_response()` 返回 `status`、`trace_id`、`is_dsl`、`truncate` 和 `data`，其中 `data` 包含查询结果、meta_data、total 和 `llm_info`。

`ge_c_query_agent` 是交互式完整路径。它会从页面 payload 中解析 query、book_id、dataset_ids、short_memory 和 trace_ids，先用当前问题召回一次数据集，再调用 `llm_conv_rewriter_agent` 基于历史改写问题。如果判断不是新话题，会用改写后的问题再次召回数据集。之后 `llm_query_clarify_agent` 判断是否需要澄清；如果需要，就直接发送澄清文本并结束。信息足够时进入 SQL/DSL 查询，发送 SQL 思考和 SQL 卡片，执行成功后把结果上传 OSS，生成图表渲染协议，再调用 `summary_agent`、`summary_format_tool` 和 summary polish 输出总结。

```text
ge_c_query_agent
    ↓
build_text_2_sql_request
    ↓
call_back_dataset_knowledge
    ↓
llm_conv_rewriter_agent
    ↓
必要时再次 call_back_dataset_knowledge
    ↓
llm_query_clarify_agent
    ↓
SQL / DSL 查询
    ↓
llm_sql_cond_infill_agent + SQL 卡片
    ↓
OSS 上传 + dim_metric_judge_agent / dsl_dim_metric_judge_agent
    ↓
ge_c_query_chart_code_tool
    ↓
summary_agent → summary_format_tool → summary_polish_model
```

`query_with_summary_agent` 是轻量总结路径。它把 query 转为 `Text2SQLRequest` 后调用公共 `text_2_sql_see_query()` 完成 SQL 查询，发送 SQL 和数据消息；如果成功，再调用 summary 链路输出总结。它没有 GE C Query 的历史改写、澄清、OSS 上传和图表渲染。

## 7. Tool 详解和对应 Agent

| Tool / FunctionHub | 被谁使用 | 功能 | 外部依赖 / 返回 |
| -- | -- | -- | -- |
| `dataset_list_query_tool.dataset_list_query` | 数据集召回流程 | 根据 `bookId`、query、erp 查询报表关联数据集列表 | `source_info_service.querySourceInfoList`，返回数据源列表 |
| `autobots_call_back_dataset_full_tools.call_back_dataset_full_tool` | `call_back_dataset_knowledge()` | 根据 dataset_ids、query、erp、book_id 召回数据集 schema 和业务知识 | `dataset_rag_rpc_service.callBackDatasetFull`，返回 `DatasetKnowledge` |
| `dataset_detail_tools.dataset_info_tool` | portal Agent | 判断数据集类型，决定 SQL 还是 DSL | `dataset_detail_service.getDatasetDetail`，返回 `dataset_type` |
| `autobots_query_dataset_data_tools.query_dataset_data_tool` | SQL 路径 | 执行模型生成的 SQL | `execute_sql_rpc_service.executeSqlAntlr4/executeStandardQuery`，返回 `UnifiedResponse` |
| `gregorian_lunar_calendar_conversion_tool.convert_calendar` | `llm_dataset_sql_agent`、`llm_dataset_sql_correction_agent` | 公历/农历互转 | 仅模型明确需要农历日期转换时调用 |
| `autobots_sql_path_election_strategy_tools.sql_path_election_strategy_tool` | `sql_path_election_strategy_agent` | 对多路 SQL `PathState` 做策略选举 | 返回 `ConsensusDecision` |
| `fetch_data_and_save_file` | DSL ReAct Agent | 查询 DSL 数据并保存文件 | 返回文件名、样例、SQL 信息 |
| `pandas_repl` | DSL ReAct Agent | 对已取数文件做 pandas 二次加工 | 返回加工后的文件或结果 |
| `run_pandas` | `text_2_dsl_react_agent` | 本地 pandas 计算 | 用于多批次 DSL 结果合并、排序、计算 |
| `fetch_chart_data_and_save_file` | `text_2_dsl_react_agent` | 获取图表数据并保存 | 本地老 DSL 入口使用 |
| `autobots_ge_c_query_chart_code_tools.ge_c_query_chart_code_tool` | `ge_c_query_viz_render()` | 把图表协议转换为 C 端渲染配置 | `chart_config_for_c_service.chart-config-for-c` |
| `autobots_summary_format_tools.summary_format_tool` | summary workflow | 渲染总结中的数字占位符，做万/亿/百分比格式化 | 纯函数工具，异常时返回原文 |
| `autobots_dataframe_fetch_tools` | `summary_agent` | 读取当前 trace 注册的 DataFrame 信息 | 供 summary 按需查看全量数据 |
| `autobots_python_execute_tools.python_execute_code_tool` | `python_execute_tool` | 执行 summary 生成的 pandas/numpy 代码 | 要求 `result` 是 DataFrame，支持一次纠错 |

从调用方式看，数据集召回、数据集详情、SQL 执行、选举和 summary 格式化大多由 workflow 显式调用；`convert_calendar`、DSL 数据工具、summary 的 DataFrame 工具则是暴露给 ReAct 模型选择调用。这个区分很重要：前者是确定性编排，prompt 只影响其前后输入；后者则依赖 prompt 让模型按协议输出 `tool_name` 和 `arguments`。

## 8. Summary、Python 和 DataFrame 机制

`summary_agent` 的 prompt 来自 `summary_prompt`，输入由 `summary_agent_func_format_input()` 组装，包含用户 query、数据集业务知识、查询 SQL、当前日期、结果样例、schema 和 `initial_df_var_name`。真实查询结果会被注册成当前 trace 隔离的 pandas DataFrame，prompt 中会告诉模型变量名；当结果较大时，prompt 只展示样例数据，但 Python 计算可以基于内存里的全量 DataFrame。

`summary_agent` 支持三种输出：工具调用 JSON、包含 `summary` 字段的 JSON、普通自然语言文本。若需要复杂计算，模型可以调用 `python_execute_tool`，而 `python_execute_tool` 内部再调用 `python_execute_code_tool` 执行代码；失败时调用 `python_code_correction_agent` 修正一次。workflow 在 summary 完成后总会被动调用 `summary_format_tool`，所以数字占位符格式化不依赖模型主动选择工具。

图表链路先通过 `dim_metric_judge_agent` 或 `dsl_dim_metric_judge_agent` 判断字段角色，再由 `PureTableChart` 生成基础渲染协议，最后调用 `ge_c_query_chart_code_tool` 请求 C 端图表配置服务。也就是说，图表选择不是直接由 SQL Agent 决定，而是由查询结果字段角色和独立图表服务共同决定。

## 9. 配置与动态刷新

`application-text-2-sql-agent-*.yml` 配置模型、LAF、OSS 和远程 JSF 服务地址。LAF 中除 prompt 外，还控制 `multi_sql_path_parallel_agents`、`multi_dsl_path_parallel_agents`、`llm_models`、`multi_agent_llm_model`、异常诊断映射、日志截断阈值、澄清开关、总结样例阈值等。`oxy_space_refresh.llm_model_refresh(get_llm_models())` 和 `agent_refresh(get_multi_agent_llm_model())` 让这些动态配置能影响运行中的 Agent 模型选择。

远程服务主要包括数据源列表服务、数据集 RAG 服务、SQL 执行服务、数据集详情服务和 C 端图表配置服务。SQL 路径和 DSL 路径最终都被包装成 `PathState`，因此上层只关心 `llm_sql_info`、`exec_result`、`is_success`、`exception` 和 `extra.data_size`，不需要区分底层是 SQL 还是 DSL。

## 10. 示例

如果请求是 `{"query":"统计上月各渠道成交金额","bookId":12345,"erp":"zhangsan"}`，且没有显式 scene，master 默认进入 `ge_c_query_analysis_agent`。workflow 会根据 `bookId` 取数据集列表，再召回最相关的数据集 schema 和业务知识。如果数据集是 SQL 类型，`llm_dataset_sql_agent` 生成 SQL，`query_dataset_data_tool` 执行，成功后返回 `status=200`、`trace_id`、`is_dsl=0`、`data.data`、`data.meta_data`、`data.total` 和 `data.llm_info.llm_sql`。

如果用户在黄金眼 C 端页面问“按渠道看上月销售额，并总结变化”，scene 命中 `ge_c_query_agent`。它会基于页面 payload 和历史会话构造 `Text2SQLRequest`，先改写问题并判断是否澄清，再执行 SQL/DSL 查询。成功后前端会收到 SQL 思考、SQL 卡片、图表渲染协议和最终 summary；若用户只问“这个怎么样”且缺少指标或时间，澄清 Agent 会先返回补充问题。

如果数据集类型是 DSL，查询不会进入 `llm_dataset_sql_agent`，而是由 `dsl_agent` 选择 `llm_dsl_react_agent_i` 或 `llm_dsl_react_analysis_agent_i`。DSL ReAct Agent 通过 `fetch_data_and_save_file` 获取数据，必要时用 `pandas_repl` 做二次加工，最终文件再被读回并转成 `PathState`，上层继续复用图表和总结链路。

## 11. 当前实现需要注意的点

`llm_dataset_dsl2sql_prompt` 在 LAF getter 中存在，但当前代码检索没有发现主链路实际使用它；DSL 主链路使用的是 `dsl_react_agent_prompt`、`dsl_react_analysis_agent_prompt` 和 `llm_dsl_path_election_prompt`。如果后续要调整 DSL 生成逻辑，需要先确认线上配置和实际绑定 Agent，而不是只看 prompt key 名称。

`query_with_summary_agent_oxy_space.py` 中 `llm_model="query_with_summary_agent"` 看起来不像已注册的 LLM 名称，但这个 workflow 的实现不直接依赖自身 LLM 推理，而是调用子 Agent 和工具完成任务；如果未来让该 workflow 自身调用模型，需要核对这个模型名配置。

`ge_c_query_agent` 的交互式链路依赖页面侧 `shared_data.payload` 格式。如果 payload 中没有 `report_pref.book_id` 或 `user_data.dataset_id`，会退化为普通 `Text2SQLRequest` JSON 解析。调试时需要区分“API 请求体模式”和“页面 payload 模式”，否则容易误判为参数转换失败。
