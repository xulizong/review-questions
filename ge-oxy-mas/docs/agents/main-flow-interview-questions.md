# 主链路 Agent 面试题与参考答案

本文只覆盖当前项目普通 chat/report 多步骤主链路：`master_agent → data_planner → data_executer → process_and_summary_agent → process_expert/Text取数Agent → summary_expert → global_summary_agent`。非主链路分支只在答案中作为边界说明。

## 1. 主链路与结构设计

| 问题 | 参考答案 |
| --- | --- |
| 当前项目主链路从用户请求到最终回答是怎么流转的？ | 普通请求进入 `master_agent` 后，先执行 `add_data_desc` 补齐数据源描述，再由 `pre_process_clarifier` 生成 `all_data_summary`、时间信息、知识和 few-shot，然后 `get_input_mode` 判断 chat/report 模式。常规多步骤链路会调用 `data_planner` 生成 `make_graph_plan` 协议 JSON，解析成 `GraphPlan` 后交给 `data_executer` 按依赖调度。每个步骤由 `process_and_summary_agent` 执行，内部先走 `process_expert/Text取数Agent` 取数和加工，再走 `summary_expert` 生成节点结论，最后由 `global_summary_agent` 聚合成 Markdown 或 HTML。 |
| 为什么 `master_agent` 适合做 `WorkflowAgent`，而不是直接让一个 LLM 自主完成所有步骤？ | 因为主链路包含数据源补全、模式分流、状态写入、planner 调用、执行器调度、步骤附件合并和最终总结，这些动作需要稳定的工程编排。`WorkflowAgent` 让 Python workflow 明确控制执行顺序和状态边界，LLM 只在适合的子节点里承担规划、工具选择或总结能力。 |
| 当前项目更像 manager/orchestrator 架构还是 handoff 架构？ | 更像 manager/orchestrator。`master_agent` 和各个 workflow 始终保留控制权，把 `data_planner`、`data_executer`、`process_and_summary_agent`、`global_summary_agent` 当作专门能力调用；它不是把用户会话完全交给某个子 Agent 接管。 |
| `data_planner` 和 `data_executer` 为什么要拆开？ | `data_planner` 负责把用户问题、数据概况、知识和历史转成图结构计划；`data_executer` 负责消费已经生成的 `GraphPlan`，按依赖关系调度步骤。拆开后，计划质量、执行并发、失败处理和状态合并可以分别优化。 |
| `data_executer` 为什么不是真正的数据执行器？ | 它不直接取数、不执行 Python、不生成业务结论。它的职责是判断哪些图节点可以启动，把 `current_step_index` 写入 request 副本，然后调用 `process_and_summary_agent`。真正的数据处理发生在 `process_expert → Text取数Agent → tools`。 |
| `GraphPlan` 在主链路里承担什么角色？ | `GraphPlan` 是 planner 和 executer 之间的状态契约，保存 `id`、`title`、`step_command`、`dependencies`、`plan_attachment`、`step_result`、`step_status`、`completed_step` 等字段。planner 生成图，executer 根据这些字段执行和回写状态。 |
| `shared_data`、`arguments`、`short_memory` 的边界是什么？ | `arguments` 是当前 Agent 调用的入参；`shared_data` 是一次请求内跨 Agent 共享的状态和产物，比如 `plan`、`current_step_index`、`workspace_file_dict`；`short_memory` 是历史上下文。主链路最关键的是 `shared_data["plan"]` 和 `shared_data["current_step_index"]`。 |
| 哪些分支不属于本文关注的多步骤主执行流？ | `query_data_flow` 查数快捷流、`easy_analysis_flow` 单步分析、`read_report_agent` 报告解读、`explore_agent` 探索、QA、全托管、direct 执行和 HITL 占位符都不是常规多步骤图计划执行主链路。 |

## 2. 各 Agent 特点

| 问题 | 参考答案 |
| --- | --- |
| `DataGraphPlanner` 和普通 `ReActAgent` 有什么不同？ | `DataGraphPlanner` 是定制 planner。它仍然按 ReAct 方式解析 LLM JSON，但把 `make_graph_plan` 放进 `trans_to_answer_tools`，所以模型输出 `make_graph_plan` 时会被当作最终规划结果，不会真实执行这个函数。 |
| `make_graph_plan` 是真实工具调用吗？ | 在 planner 主路径里不是。它是提交计划的协议格式；真实工具调用的是 `search_knowledge`。如果 `make_graph_plan` 函数被真实执行，代码里反而会返回“此时使用make_graph_plan是错误的。” |
| `search_knowledge` 在 planner 中解决什么问题？ | 它补充系统知识、用户知识、语义层知识和分析大纲，减少 planner 只靠用户 query 猜计划的风险。工具 observation 会写回 planner 的 ReAct memory，进入下一轮规划。 |
| `DataExecuter` 的核心状态有哪些？ | 核心状态包括 `completed_indices`、`running_tasks`、`task_memories`、`id_to_index`、`step_status`、`step_result` 和 `plan_attachment`。这些状态分别用于判断依赖、管理并发、传递上游 memory、映射步骤 id、记录执行状态和保存步骤产物。 |
| `process_and_summary_agent` 为什么单独存在？ | 它是单步骤执行边界。`data_executer` 只负责调度，真正进入某个步骤后，需要一个固定 workflow 串起数据处理和节点总结，所以 `process_and_summary_agent` 先调用 `process_expert`，再调用 `summary_expert`，并把两者产物写入当前步骤附件。 |
| `process_expert` 和 `Text取数Agent` 的关系是什么？ | `process_expert` 负责准备动态 prompt、数据源描述、workspace、知识和工具描述；`Text取数Agent` 负责真正的 ReAct 工具循环，调用 `text2data_tool`、`py_coder_tool`、`metric_analyze_tool`、`web_search_tool` 等工具。 |
| `Text取数Agent` 为什么是主链路里最像传统 Agent 的部分？ | 因为它每轮让 LLM 输出工具调用 JSON，真实执行工具，把 observation 写回上下文，再继续下一轮，直到通过 `summary_tool` 输出最终业务结论。 |
| `summary_expert` 当前是否会重新调用 LLM 总结节点？ | 当前节点总结主要由 `node_summary_agent` 从 `process_expert` 附件里抽取 `conclusion` 字段并修复 Markdown 表格，不是重新让 LLM 从零总结。 |
| `global_summary_agent` 会重新取数或重新执行步骤吗？ | 不会。它只读取每个步骤保存的 `summary_expert` 和 planner 大纲，合并成最终 Markdown 或 HTML。最终报告缺内容时，应先检查对应步骤附件，而不是先改全局总结 prompt。 |

## 3. Planner 相关问题

| 问题 | 参考答案 |
| --- | --- |
| `data_planner` 请求 LLM 前会放入哪些上下文？ | 会放入用户 query、`all_data_summary`、`day_info`、`default_knowledge`、`recall_few_shot`、`planner_web_search_info`、短期记忆、用户级历史计划记忆和工具描述。 |
| `AUTO/REPORT/DEEP_RESEARCH` 模式如何影响 planner？ | 普通 chat 默认 `AUTO`，`shared_data.type=2` 且 mode 仍为 `AUTO` 时会切到 report prompt，`shared_data.mode="DEEP_RESEARCH"` 时用 deep research prompt。不同模式会影响可选 scene、规划风格和步骤要求。 |
| planner 输出的 `scene` 有什么作用？ | `scene` 决定后续主流程分流。`无关问题` 直接回答，`报告解读` 转报告解读，`查数` 且只有一步时走快捷查数，常规多步骤 `确定性分析` 或 `探索性分析` 才进入 `data_executer`。 |
| planner 输出的 `dependencies` 为什么引用步骤 id 而不是数组下标？ | planner 输出的是逻辑步骤 id，下游 `GraphPlan` 的状态数组按 index 存储，所以 `data_executer` 会先构造 `id_to_index`，再把依赖 id 映射成数组下标。这样图结构表达更稳定。 |
| `global_config` 后续怎么使用？ | `parse_graph_planner_result` 会把 `global_config` 拼到每个步骤 command 前面，使全局时间、商品范围、筛选条件等限制在所有步骤中都生效。 |
| planner 输出不是合法 JSON 会发生什么？ | `DataGraphPlanner` 会把格式错误写回 ReAct memory，让模型按正确 JSON 格式重试；如果最终 `parse_graph_planner_result` 解析失败，可能返回 `("报错", {})`，后续读取 `plan_dict` 字段时仍可能中断。 |
| 为什么 planner 不应该直接调用 `process_expert`？ | planner 的职责是“生成计划”，不是“执行计划”。如果 planner 直接执行，会把规划和执行状态耦合，破坏 `GraphPlan` 调度、步骤附件沉淀和失败重试边界。 |
| 如何判断一个 planner 计划质量高不高？ | 高质量计划应步骤粒度适中、依赖关系正确、每步有明确计算方法和输出格式、没有遗漏全局限制、能正确区分查数/确定性分析/探索性分析，并且不会把简单问题过度拆分。 |
| planner 的 few-shot 记忆有什么价值？ | 它把历史对话中的计划结构转成类似“Tool [make graph plan] execution result”的上下文，让模型参考过往问题的拆分方式和计划风格。 |
| planner 最容易出问题的点有哪些？ | 主要是 JSON 格式不稳定、scene 误判、步骤过细或过粗、依赖环、依赖缺失、未知依赖 id，以及知识召回噪声污染计划。 |

## 4. Executer 调度问题

| 问题 | 参考答案 |
| --- | --- |
| `data_executer` 收到的上游内容是什么？ | 它读取的是 `shared_data["plan"]` 里的 `GraphPlan`，不是 planner 原始 JSON，也不是用户原始问题。planner 原始输出已经被 `parse_graph_planner_result` 压平成 `id/title/command/dependencies`。 |
| `id_to_index` 为什么必要？ | `dependencies` 里引用的是步骤 id，但 `step_status`、`step_result`、`plan_attachment` 都按数组下标存储。`id_to_index` 用于把模型生成的依赖 id 转成执行器内部 index。 |
| `completed_indices` 代表什么？ | 当前实现里它代表“步骤已经结束或已有结果”，不是严格意义上的“步骤成功”。这也是失败传播风险的核心：失败步骤也可能被加入 `completed_indices`。 |
| 没有显式 `dependencies` 时如何调度？ | 如果没有依赖字段，代码会退化成串行执行，第 N 步默认等待第 N-1 步完成。 |
| 有多个依赖满足的节点时如何并发？ | executer 会把依赖满足的节点放入 candidates，在 `get_max_concurrent()` 限制内用 `asyncio.create_task` 并发启动。 |
| 为什么执行步骤时要 deep copy `OxyRequest`？ | 并行任务如果直接共享原 request，会互相覆盖 `current_step_index`、步骤附件和临时状态。deep copy 可以隔离单步骤写入，步骤结束后再把附件和 `workspace_file_dict` 合并回主 request。 |
| `current_step_index` 的作用是什么？ | `data_executer` 不直接把步骤指令放进调用 arguments，而是在 request 副本里写入 `shared_data["current_step_index"]`。`process_and_summary_agent` 再根据这个 index 从 `GraphPlan.step_command` 取当前步骤 command。 |
| 上游步骤结果如何传给下游依赖步骤？ | executer 会把依赖步骤产生的增量 memory 加入当前步骤输入；同时步骤产物保存在 `GraphPlan.plan_attachment`，下游可以通过当前步骤或依赖步骤附件读取。 |
| `step_result` 和 `plan_attachment[*]["summary_expert"]` 有什么区别？ | `step_result` 是执行器层面的状态摘要，比如“步骤已执行”；真正业务结论在 `summary_expert` 附件里。最终全局总结主要依赖 `plan_attachment[*]["summary_expert"]`。 |
| `get_master_config().use_llm` 对 executer 有什么影响？ | 默认情况下 executer 不让 LLM 判断下一步，而是直接构造 `update_plan` 且 command 为 `continue`。只有配置打开 `use_llm` 后，才会把步骤上下文发给 LLM，并解析模型输出的 `update_plan`。 |

## 5. 成功失败处理问题

| 问题 | 参考答案 |
| --- | --- |
| 某个步骤失败后一定会 `skip` 吗？ | 不一定。`workflow_until_target` 没有返回结果时会使用备选 `skip`；`_execute_single_step` 捕获异常时返回 `error`；如果 `process_expert` 被 `process_and_summary_agent` 捕获，可能只是返回失败文案，外层仍可能默认标记为 `continue`。 |
| 当前实现为什么没有某一步失败就立刻中断？ | 当前设计偏 fail-soft，希望独立或弱依赖步骤仍能继续产出部分结果，避免单个工具或步骤失败导致用户完全没有结果。只有所有已执行步骤都是 `skip` 时，才会生成整体失败兜底提示。 |
| 这个 fail-soft 设计有什么风险？ | 对强依赖数据分析来说，上游失败后仍可能被加入 `completed_indices`，下游依赖会被认为满足，导致后续步骤基于缺失或错误上下文继续执行，最终结论可能不可靠。 |
| 当前依赖判断是否区分“完成”和“成功”？ | 不区分。依赖调度只检查父步骤 index 是否在 `completed_indices`，不检查父步骤 `step_status` 是否为 `continue`，也不检查父步骤是否有有效 `summary_expert` 附件。 |
| 如果上游步骤是 `error`，下游还可能执行吗？ | 可能。当前 `error` 也会写入 `step_result`，并且执行完成后会加入 `completed_indices`，因此下游依赖可能被判定为满足。 |
| `process_expert` 抛异常后为什么可能被外层认为成功？ | `combine_data_expert_workflow` 捕获 `process_expert` 异常后返回“当前步骤无法顺利执行，请执行下一步骤。”。`data_executer` 默认 `update_plan` 是 `continue`，没有识别这段文案代表步骤失败。 |
| 什么情况下会返回整体兜底提示？ | `data_executer` 在所有已执行步骤的 `step_status` 都是 `skip` 时，会调用 LLM 生成整体失败说明。如果状态是 `error` 或伪 `continue`，不一定触发这个兜底。 |
| 如果要改成严格 fail-fast，应该怎么做？ | 依赖判断应从“父节点完成”改为“父节点成功且产物有效”。可以维护 `success_indices`，要求依赖父节点 `step_status == "continue"` 且存在有效 `summary_expert` 或关键数据附件；父节点失败时，子节点标记为 `blocked_by_dependency`，然后进入重试、重新规划或整体兜底。 |
| 如果保留 fail-soft，怎么降低错误传播？ | 在下游启动前检查父节点状态和关键附件；全局总结前过滤失败节点或显式标注失败步骤；最终输出中说明哪些结论缺失或不完整，避免把部分结果包装成完整结论。 |
| 如何评价当前失败处理设计？ | 它的产品目标是尽量给用户部分结果，适合弱依赖或并行分析任务；但对强依赖链路不够严格，应该补充依赖成功判断、失败阻断、重新规划或明确的不完整结果标注。 |

## 6. 数据源、工具和 Prompt 问题

| 问题 | 参考答案 |
| --- | --- |
| `add_data_desc` 对 dataset 和 file 的处理差异是什么？ | dataset 会调用外部数据集详情接口，补齐字段名、字段类型和数据集描述；file 主要在 workspace 本地读取文件前几行和字段类型，必要时根据 URL 兜底下载。 |
| `pre_process_clarifier` 对 planner 有什么作用？ | 它生成 planner 需要的 `all_data_summary`、时间信息、默认知识、few-shot 和联网提示。planner 看到的是预处理后的分析上下文，而不是原始用户输入。 |
| `colDesc` 是接口直接返回的吗？ | 不是。`colDesc` 是本地根据接口返回的字段列表整理出来的列描述文本，不是单独的接口字段。 |
| `process_expert` 如何决定 Text取数Agent 可用工具？ | 它根据数据源和配置生成 active tools：有在线 dataset 时启用 `text2data_tool`，`py_coder_tool` 始终启用，`metric_analyze_tool` 和 `web_search_tool` 根据路由或 `tool_list` 条件注入。 |
| 为什么 `Text取数Agent` 注册时 prompt 为空也能工作？ | 因为它的真实 system prompt 是 `process_expert` 运行时通过 `build_multi_data_prompt` 动态构造并传入的，注册代码里的空 prompt 不是最终入模 prompt。 |
| 工具描述为什么重要？ | 工具描述是 LLM 和确定性工具之间的契约。它会影响模型是否选择正确工具、是否生成正确参数、是否理解工具返回结果，以及 token 使用效率。 |
| `py_coder_tool` 调用前有什么校验？ | `llm_response_parser` 会检查代码是否从 workspace 通过 `pd.read_csv` 或 `pd.read_excel` 读取数据，避免模型在代码里凭空构造数据。 |
| `summary_tool` 为什么不是普通业务工具？ | 它是 `Text取数Agent` 的结束协议。解析到 `summary_tool` 后，系统会做数值来源校验、后置裁判、数值标签化和 Markdown 表格修复，再把结果作为业务结论返回。 |
| `workspace_file_dict` 在主链路中有什么作用？ | 它保存工具生成的临时文件、数据处理 id、OSS 地址和文件名。report 模式最终会尝试把这些信息追加到 HTML 的数据来源区域。 |
| DUCC/LAF 配置在排查 prompt 问题时怎么定位？ | 先找对应 provider，比如 `get_planner_prompt`、`get_executor_prompt`、`get_global_summary_prompt_md/html`、各类 tool description provider，再看 `docs/prompt文档.md` 中当前快照和本地 fallback。 |

## 7. 单步骤执行与总结问题

| 问题 | 参考答案 |
| --- | --- |
| `process_and_summary_agent` 执行一个步骤时具体做什么？ | 它先根据 `shared_data["current_step_index"]` 读取当前步骤 command，调用 `process_expert({"query": command})` 完成取数、加工和分析，并保存 `process_expert` 附件；然后调用 `summary_expert({"query": "无"})` 生成节点结论，并保存 `summary_expert` 附件。 |
| 为什么 `summary_expert` 的 query 是 `"无"`？ | 因为 summary 侧主要读取当前步骤附件中的 `process_expert` 输出，而不是依赖 query 重新分析。query 固定为 `"无"` 表示总结任务不再由用户原始问题驱动。 |
| `node_summary_agent` 当前怎么生成节点总结？ | 它从当前步骤 `process_expert` 附件里遍历每个 item 的 `conclusion` 字段，把结论拼接起来，再尝试修复 Markdown 表格。 |
| 如果 `process_expert` 输出没有 `conclusion`，会怎样？ | `node_summary_agent` 抽不到有效内容，节点总结可能为空，后续 `summary_expert` 附件和全局总结对应段都会缺内容。 |
| `visual_agent` 在主链路里是什么定位？ | 它是节点总结后的可视化补充，不是主调度器，也不是取数工具。只有命中可视化协议或 EasyBI 相关逻辑时才会产生实际可视化结果。 |
| `process_expert` 的输出为什么要存入 `plan_attachment`？ | 因为后续 `summary_expert`、可视化和全局总结都依赖步骤产物。`plan_attachment` 是步骤级产物沉淀位置，也是排查最终报告缺内容时最重要的状态。 |

## 8. 最终总结与输出问题

| 问题 | 参考答案 |
| --- | --- |
| `pre_process_report_expert` 做了什么？ | 它遍历 `GraphPlan.plan_attachment`，收集每个步骤的 `summary_expert` 作为 `report_segment`，同时把步骤标题组成 `outline_full`，再把这些内容交给 `global_summary_agent`。 |
| `global_summary_agent` 的输入主体是什么？ | `global_summary_workflow` 会调用 `_build_merged_query`，把每个子步骤大纲和对应步骤报告按段拼接，中间用分隔符保留结构。 |
| chat 模式和 report 模式最终总结有什么差异？ | chat 模式调用 `global_summary_agent_chat`，输出 Markdown；report 模式调用 `global_summary_agent_report`，输出 HTML，并做模型输出清洗、样式注入、数据来源拼接和长度兜底。 |
| 如果最终报告缺了某个分析点，优先排查哪里？ | 优先查对应步骤的 `plan_attachment[*]["summary_expert"]`，再查 `process_expert` 附件和工具输出。如果步骤附件本身缺失，全局总结无法凭空补齐。 |
| report 模式输出过短怎么兜底？ | 如果 HTML 输出长度小于原始 Markdown 分段内容的 10%，`global_summary_workflow` 会使用 Markdown 转 HTML 的结果作为兜底。 |
| 全局总结是否应该重新校验每个数值？ | 当前实现主要信任上游步骤结果，不重新取数。若业务要求更高可信度，应在步骤内或全局总结前增加数值溯源、一致性校验和失败标注。 |

## 9. 状态、并发和一致性问题

| 问题 | 参考答案 |
| --- | --- |
| 并发执行时最容易出现什么状态问题？ | 容易出现 `current_step_index` 覆盖、`shared_data["plan"]` 冲突、附件互相污染、workspace 文件合并顺序不稳定等问题。当前通过 request deep copy 和步骤结束后合并来降低风险。 |
| `plan_attachment` 为什么按 step index 存储？ | 因为每个步骤都有独立的 `process_expert` 和 `summary_expert` 产物，按 index 存储便于当前步骤和依赖步骤快速读取。 |
| `completed_step` 在并发模式下有什么局限？ | 它更像完成数量，不一定表示“连续执行到第几步”。并发模式下真正判断依赖主要依赖 `completed_indices`。 |
| planner 生成循环依赖会怎样？ | executer 可能找不到可启动节点，记录死锁日志后跳出循环。当前这不是完整的用户友好失败处理，最好在 planner 输出解析后先做 DAG 合法性校验。 |
| dependency 指向不存在的 id 会怎样？ | 当前代码记录 warning 后暂时忽略未知依赖，这可能导致本应等待的步骤提前执行。更严格的做法是把计划判为非法并要求重新规划。 |
| 重跑场景如何影响执行？ | executer 会读取 restart 信息，从指定步骤之后覆盖 `step_command` 和 `title`，并注入 `_is_read_cache_for_restart` 控制缓存读取。 |
| 如果提高最大并发，会影响哪些方面？ | 会影响执行速度、LLM/tool 并发压力、外部服务限流、workspace 写冲突、状态合并顺序和依赖 memory 注入稳定性。 |

## 10. 工程化与可靠性问题

| 问题 | 参考答案 |
| --- | --- |
| 主链路可观测性应该记录哪些事件？ | 应记录每次 LLM 请求、工具调用、Agent 调用、step start/finish、plan 变更、附件写入、fallback、error/skip、全局总结输入输出长度和 trace_id/node_id。 |
| 为什么 Agent 系统需要 tracing？ | 多 Agent 链路很长，问题可能发生在 planner、工具调用、执行器状态、步骤附件或全局总结任一层。Tracing 能还原 LLM 生成、工具调用、子 Agent 调用和自定义事件，方便定位最终报告缺失或错误来源。 |
| 当前项目有哪些 guardrail 或校验点？ | 主要包括 planner JSON 解析、`py_coder_tool` 数据读取校验、`summary_tool` 后置裁判/数值校验、Markdown 表格修复和 report HTML 长度兜底。当前不足是依赖调度没有严格校验父节点成功。 |
| 如何为主链路做 eval？ | 可以准备固定数据源和固定用户问题，分别断言 planner 步骤、依赖关系、工具调用轨迹、关键数值结果、步骤结论和最终总结。评估应拆成 planner eval、工具选择 eval、步骤执行 eval 和 global summary eval。 |
| 如果要新增一个业务工具，需要改哪些地方？ | 需要实现工具函数，注册到对应 FunctionHub，增加 tool description provider，修改 `process_expert` 的工具选择逻辑，必要时补充 Text取数Agent parser/参数校验，再增加 prompt 示例和 eval 用例。 |
| 如何防止 LLM 在工具调用前凭空计算？ | prompt 中明确要求必须使用工具，parser 校验关键工具参数，`py_coder_tool` 要求从 workspace 读数据，`summary_tool` 做数值来源检查，必要时用 judge Agent 或外部校验器复核。 |
| 哪些操作适合 human-in-the-loop？ | 报告大纲确认、执行高成本工具、执行 SQL/Python、写入或删除文件、对业务决策影响较大的最终报告，都适合加人工确认。 |
| 如何管理 prompt 版本和线上回滚？ | 通过 DUCC/LAF provider 定位 prompt 来源，保留本地 fallback，记录版本、灰度范围、效果指标和回滚策略。排查线上行为时不能只看本地 prompt，还要看远端配置快照。 |
| 如何控制成本和延迟？ | 控制 planner 步骤数，限制 executer 最大并发，压缩工具返回内容，减少无效重试，把总结分层聚合，并在失败时尽早阻断明显无意义的下游执行。 |
| 数据安全相关会问什么？ | 会关注文件下载和 workspace 隔离、OSS 输出、SQL/Python 执行边界、用户数据隔离、敏感字段脱敏、工具调用审计和错误日志是否泄露数据。 |
| 为什么“给 Agent 用的工具”和普通 API 不完全一样？ | 普通 API 主要面向确定性程序；Agent 工具还要让 LLM 看懂边界、参数、返回结果和错误信息。工具描述、返回上下文、错误提示和 token 长度都会影响 Agent 是否能正确使用工具。 |

## 11. 场景追问

| 问题 | 参考答案 |
| --- | --- |
| 第 1 步取数失败，第 2 步依赖第 1 步，当前代码会怎样？ | 第 1 步可能被标成 `skip`、`error` 或伪 `continue`。只要它写入 `step_result` 并加入 `completed_indices`，第 2 步仍可能启动，因为当前依赖判断只看“是否完成”，不看“是否成功”。 |
| 如果让你修复上面的问题，你会怎么改？ | 增加成功依赖判断，例如维护 `success_indices`，要求父节点 `step_status == "continue"` 且关键附件非空；失败的子节点标记为 `blocked_by_dependency`，最终进入整体兜底、重试或重新规划。 |
| planner 输出 5 个步骤，其中两个可以并行，executer 如何识别？ | executer 遍历每个步骤的 `dependencies`，只要依赖 id 对应的 index 都在 `completed_indices` 中，就把该步骤加入 candidates，并在并发上限内启动。 |
| planner 输出 `dependencies=["9"]`，但没有 id 为 9 的步骤，会怎样？ | 当前实现会记录 warning，但暂时忽略未知依赖，可能导致该步骤提前执行。更合理的处理是把计划判定为非法，要求 planner 重新生成合法 DAG。 |
| `process_expert` 返回结构化数据但没有 `conclusion`，最终报告会怎样？ | `node_summary_agent` 抽不到节点结论，导致当前步骤的 `summary_expert` 可能为空，global summary 对应段也会缺内容。 |
| 本地 file 格式损坏，失败会在哪一层暴露？ | 可能在 `add_data_desc` 的 pandas 预读阶段暴露，也可能在 `Text取数Agent → py_coder_tool` 真正读取文件时暴露，取决于入口下载和预处理是否已经成功。 |
| 用户说“生成一份报告”，系统如何决定进入 report 模式？ | 主要看 `shared_data.type`、`report_pref` 和 `get_input_mode`，不是只靠用户自然语言。`type=2` 通常进入 report 链路。 |
| 最终 HTML 很短但每步 summary 都有内容，系统怎么兜底？ | report 模式下，如果 HTML 长度小于 Markdown 分段内容的 10%，`global_summary_workflow` 会把 Markdown 分段内容直接转换成 HTML 作为兜底。 |
| 要加入工具执行审批，应该放在哪一层？ | 如果审批具体工具，放在 `Text取数Agent` 工具调用前；如果审批某个计划步骤，放在 `data_executer` 启动步骤前；如果审批整体大纲，放在 report 计划确认阶段。 |
| 怎么判断线上问题是 planner 问题还是 executer 问题？ | 先看 `GraphPlan` 是否合理。计划步骤、依赖和 scene 已经错，就是 planner 问题；计划正确但执行顺序、附件合并、失败传播异常，是 executer 问题；单步结论错，通常在 `process_expert/Text取数Agent/tools`。 |

## 12. 跨数据集聚合与关联设计

### 问题：一个 query 需要聚合两个数据集才能完成时，应该让 LLM 自己关联，还是提前配置数据集关系？

当前项目里，一次 `text2data_tool` 调用只选择一个 `dataset_id`。一个问题需要两个数据集时，`Text取数Agent` 需要分别调用两次 `text2data_tool`，把结果保存成两个临时 CSV，再调用 `py_coder_tool` 生成 Python 代码，通过 `pd.merge()` 完成合并。当前主流程没有读取统一的数据集关系配置，所以关联字段、关联粒度和 Join 方式主要由 LLM 根据 query、字段描述和数据样例推断。

这种实现能完成简单的合并，但不能稳定保证结果正确。字段名相似不代表业务含义相同，同样的值也不代表两个字段可以直接关联。更重要的是，如果两张表的数据粒度不同，即使 Join Key 选对了，直接关联也可能形成多对多连接，导致指标被重复累加。

### 具体例子

假设用户上传了两个数据集：

```text
订单明细，dataset_id=1001
字段：下单日期、SKU编码、订单号、成交金额
原始粒度：一行代表一个订单商品

流量数据，dataset_id=1002
字段：统计日期、商品SKU、渠道、曝光UV
原始粒度：一行代表一个日期、SKU和渠道组合
```

用户提问“分析每个 SKU 的曝光 UV 与成交金额之间的关系”。如果 LLM 直接按“日期 + SKU”关联两份原始数据，某天某 SKU 在订单表有 100 行、在流量表有 5 个渠道行时，Join 会产生 500 行。每条订单会被重复 5 次，每个渠道的曝光 UV 也会被重复 100 次，最终聚合结果是错误的。

正确处理是先把两个数据集分别聚合到相同的目标粒度，再进行关联：

```python
order_daily = (
    orders.groupby(["下单日期", "SKU编码"], as_index=False)
    .agg(成交金额=("成交金额", "sum"))
)

traffic_daily = (
    traffic.groupby(["统计日期", "商品SKU"], as_index=False)
    .agg(曝光UV=("曝光UV", "sum"))
)

result = order_daily.merge(
    traffic_daily,
    left_on=["下单日期", "SKU编码"],
    right_on=["统计日期", "商品SKU"],
    how="inner",
    validate="one_to_one",
)
```

`validate="one_to_one"` 用来强制检查聚合后的关联键是否唯一。如果任意一边仍然存在重复键，处理应直接失败，而不是继续输出可能被放大的数值。

### 推荐的正确方案

生产环境应采用“关系配置为主，LLM 选择为辅”的模式。不应让 LLM 自由发明关联字段和聚合粒度，而应提前建立数据集关系配置，至少包含以下内容：

```json
{
  "relation_id": "order_traffic_by_day_sku",
  "left_dataset_id": "1001",
  "right_dataset_id": "1002",
  "join_keys": [
    {"left": "下单日期", "right": "统计日期", "type": "date"},
    {"left": "SKU编码", "right": "商品SKU", "type": "string"}
  ],
  "target_grain": ["日期", "SKU"],
  "left_aggregation": {"成交金额": "sum"},
  "right_aggregation": {"曝光UV": "sum"},
  "join_type": "inner",
  "expected_cardinality": "one_to_one"
}
```

LLM 只负责理解用户意图，判断需要数据集 `1001` 和 `1002`，然后从已配置的关系中选择 `order_traffic_by_day_sku`。关联字段、聚合粒度、聚合方式、Join 类型和预期基数都由配置确定。执行层根据配置分别调用两次 `text2data_tool`，再让 `py_coder_tool` 按受约束的关系生成代码，并对关联前后的唯一性、行数和指标总量进行校验。

| 内容 | 应由谁决定 |
| --- | --- |
| 本次 query 需要哪些数据集 | 数据集路由器和 LLM |
| 使用哪条已配置关系 | LLM 根据用户意图选择 |
| 关联字段和字段类型 | 数据集关系配置 |
| 关联前聚合粒度和聚合方式 | 指标定义和数据集关系配置 |
| 时间范围、指标和筛选条件 | LLM 从用户 query 中提取 |
| 关联执行代码 | LLM 生成，但必须符合关系配置 |
| 关联键唯一性、基数和行数膨胀检查 | 程序强制校验 |

并不需要为所有数据集两两手工配置。可以先建立“业务日期”、“SKU”、“品牌ID”等标准业务维度，将不同数据集的字段映射到同一标准维度。只有 SKU 映射 SPU、旧编码映射新编码等复杂关系，才需要单独的关系表或映射规则。

如果两个临时上传的数据集没有已配置关系，系统可以根据字段名、类型、唯一率和值重合率生成候选关系，但不应静默执行。候选关系不唯一、存在多对多风险或业务含义无法确认时，应该让用户选择关联字段或明确两份数据的对齐粒度。

因此，这个问题的标准回答是：**当前项目主要由 LLM 完成多份取数结果的关联；面向稳定的生产实现，应提前配置数据集之间的关联键、粒度、聚合方式和基数，LLM 只负责选择关系和填充用户条件，执行层负责强制校验。**

## 13. 多数据集 Query 匹配与路由

### 问题：存在多个数据集时，如何正确匹配用户的 query 去查询？

当前项目会先通过 `add_data_desc` 补齐所有数据集的名称、`dataset_id`、数据描述和字段信息，然后使用同一个用户 query 对每个数据集分别调用语义召回接口。召回结果按数据集名称分组，再和所有数据集的字段描述一起放入 `Text取数Agent` 的 prompt。`Text取数Agent` 根据 query、数据集名称、字段和枚举值自己选择 `dataset_id`，然后调用 `text2data_tool`。

当前工具层只保证 LLM 选择的 `dataset_id` 属于用户本次上传的数据集列表，不会检查它是否是语义上最合适的数据集。语义接口返回的数据集和字段 `score` 在格式化为 Markdown 时也没有保留，当前主流程没有基于分数的跨数据集排序、歧义阈值或执行后自动纠错。因此，当同一个业务词、字段或枚举值同时出现在多个数据集中时，当前选择属于 LLM 软匹配，不能从代码层保证一定选对。

### 具体例子

假设存在两个数据集：

```text
订单明细，dataset_id=1001
业务范围：订单交易
字段：下单日期、品牌、成交金额
品牌枚举值：包含“苹果”

流量归因，dataset_id=1002
业务范围：流量和渠道归因
字段：访问日期、品牌、渠道、引导成交金额
品牌枚举值：包含“苹果”
```

用户提问“分析苹果销售额的变化趋势”时，“苹果”在两个数据集中都能召回，只能证明两个数据集都支持 `品牌=苹果` 这个筛选条件，不能用来决定数据源。真正需要判断的是用户所说的“销售额”究竟是订单实际成交金额，还是渠道引导成交金额。用户没有给出这个信息时，问题本身就存在歧义，无法通过更大的模型或更长的 prompt 保证猜对。

如果用户问“分析苹果的渠道引导销售额趋势”，“渠道”和“引导”与数据集 `1002` 的业务范围和指标口径一致，可以唯一选择 `1002`。如果用户问“对比苹果的实际成交金额和渠道引导成交金额”，则不应选择一个数据集，而应识别为多数据集任务，分别查询 `1001` 和 `1002`，然后按第 12 章的关系配置和粒度校验进行对齐。

### 推荐的正确方案

应在 `Text取数Agent` 之前增加一个可校验的数据集路由阶段，完整流程为：

```text
用户 query
  → 结构化意图识别
  → 按数据集召回候选字段和枚举值
  → 必要字段硬过滤
  → 候选数据集评分
  → 单数据集 / 多数据集 / 歧义澄清
  → 绑定 dataset_id 执行
```

首先要把用户 query 拆成结构化意图。对于“分析苹果销售额的变化趋势”，可以拆成：

```json
{
  "metrics": [{"name": "销售额", "qualifiers": []}],
  "dimensions": ["日期"],
  "filters": [{"dimension": "品牌", "value": "苹果"}],
  "source_hint": null,
  "analysis_type": "trend"
}
```

然后对每个数据集进行硬条件检查。数据集必须能提供用户要求的指标、分组维度、筛选字段和时间字段，缺少任一必要能力就应直接淘汰，而不是让 LLM 尝试生成 SQL。对于通过硬过滤的候选数据集，再综合指标口径、枚举值、维度、业务范围和数据粒度评分。

| 匹配依据 | 建议作用 |
| --- | --- |
| 用户明确指定数据集名称或 ID | 作为硬约束，验证后直接选择 |
| 指标名称、同义词和业务口径 | 主要匹配依据 |
| 筛选字段和枚举值 | 确认数据集能否表达筛选条件 |
| 时间和分组维度 | 确认数据集能否按目标粒度输出 |
| 数据集名称、描述和业务域 | 判断指标所属业务范围 |
| 数据粒度和更新周期 | 判断是否满足本次分析要求 |

一组可用于初期实现的决策规则是：最高候选分数达到可用阈值，且与第二名存在足够差距时，自动选择一个数据集；用户明确要求两个不同指标口径时，识别为多数据集任务；前两名分数接近时，不允许 LLM 默认选择，而是进入澄清。例如可先设置“最高分不低于 `0.70`，且前两名差值不低于 `0.15`”的初始规则，再根据真实 query 标注集校准，不能把示例阈值直接当作固定业务规则。

对于“苹果销售额趋势”，候选结果可能是：

```json
{
  "route_type": "clarify",
  "candidates": [
    {
      "dataset_id": "1001",
      "score": 0.86,
      "metric_mapping": {"销售额": "成交金额"},
      "reason": "匹配品牌、日期和实际成交口径"
    },
    {
      "dataset_id": "1002",
      "score": 0.81,
      "metric_mapping": {"销售额": "引导成交金额"},
      "reason": "匹配品牌、日期和渠道归因口径"
    }
  ],
  "question": "请确认‘销售额’指订单实际成交金额、渠道引导成交金额，还是需要对比两者？"
}
```

用户确认“订单实际成交金额”后，路由器应产生可执行决策：

```json
{
  "route_type": "single",
  "selected_dataset_ids": ["1001"],
  "rewritten_query": "查询品牌为苹果的实际成交金额，按下单日期汇总",
  "metric_mapping": {"销售额": "成交金额"}
}
```

路由结果确定后，只应把选中数据集的描述和 `dataset_id` 交给 `Text取数Agent`，而不是继续展示全部候选数据集让它重新选择。`text2data_tool` 也应校验 LLM 提交的 ID 是否属于路由器最终选中的 ID，而不只是判断它是否在全部上传列表中。对于单数据集决策，工具层可以直接覆盖为已选定的 `dataset_id`，从执行层消除再次选错的可能。

执行结果中还应保存 `dataset_id`、数据集名称、指标到字段的映射、最终 SQL 和数据粒度，用于结果溯源和下一轮对话。用户下一轮只说“再按地区拆分”时，应默认继承上一轮已确认的数据集和指标口径，除非新 query 明确改变了数据范围。如果已选数据集查询结果为空，也不应静默切换到另一个数据集，因为这会同时改变指标口径。

在当前代码中落地时，应在 `_get_data_result` 把全部 `dataset_id_list` 交给 `Text取数Agent` 之前插入路由阶段；语义召回不再只返回 Markdown，而是同时保留每个数据集的原始分数、命中字段和枚举值；并将 `_build_workflow_context` 中数据源字典的键从 `table_name` 改为 `dataset_id`，避免同名数据集相互覆盖。

因此，这个问题的标准回答是：**当前项目通过“每个数据集分别语义召回，再由 LLM 选择 `dataset_id`”完成软匹配，只能保证不访问上传列表以外的数据集，不能确定性保证数据源选择正确。正确的生产方案是增加结构化数据集路由，通过硬条件过滤、可解释评分和歧义澄清确定单数据集或多数据集，再将选中的 `dataset_id` 绑定到执行层。**

## 14. 少量数据表的 LLM 选表与宽表渐进加载

### 问题：用户个人空间中的表数量不多时，是否可以直接把表元信息交给 LLM 选表？如果某张表的列非常多，应该怎么处理？

在“候选表只来自当前用户个人空间、表数量较少、全部元信息能放进模型上下文”的前提下，可以直接让 LLM 阅读用户 query 和候选表元信息，同时完成意图理解、选表和字段映射。这是当前规模下更合适的第一版方案，不需要一开始就引入 BM25、向量库、枚举值索引和复杂的分数融合。

Spring AI Alibaba DataAgent 也包含类似的 LLM Schema 精选阶段。它先通过 [`SchemaRecallNode`](https://github.com/spring-ai-alibaba/DataAgent/blob/main/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/workflow/node/SchemaRecallNode.java) 限定候选 Schema，再把用户问题、表名、表描述、字段名、字段描述、类型、主键、少量示例值和外键交给 LLM，由 [`mix-selector`](https://github.com/spring-ai-alibaba/DataAgent/blob/main/data-agent-management/src/main/resources/prompts/mix-selector.txt) 输出相关表名。当用户空间本身已经把表范围缩小到可控规模时，可以直接把这些表作为候选集，保留后面的 LLM 精选和程序校验。

### 表元信息能一次放入 Prompt 时

可以将当前用户有权限的所有表组装成结构化 Schema。每张表至少包含表 ID、名称、业务描述、适合和不适合的用途、数据粒度、时间字段、更新周期、全部可查询业务字段、少量代表性示例值以及已配置的表关系。

```json
{
  "table_id": "1001",
  "name": "订单明细",
  "description": "订单商品粒度的实际交易数据",
  "suitable_for": ["实际成交金额", "订单量", "客单价"],
  "not_suitable_for": ["曝光UV", "渠道引导成交金额"],
  "grain": ["订单号", "SKU"],
  "time_field": "下单日期",
  "update_frequency": "T+1",
  "columns": [
    {
      "name": "order_date",
      "display_name": "下单日期",
      "type": "date",
      "description": "订单创建日期"
    },
    {
      "name": "brand_name",
      "display_name": "品牌",
      "type": "string",
      "description": "商品所属品牌",
      "examples": ["苹果", "华为", "小米"]
    },
    {
      "name": "pay_amount",
      "display_name": "成交金额",
      "type": "decimal",
      "description": "用户实际支付的商品金额",
      "aliases": ["实际销售额", "实际成交额", "GMV"],
      "aggregation": "sum"
    }
  ]
}
```

LLM 不只输出一个表名，而应输出可校验的结构化决策：

```json
{
  "decision": "single",
  "selected_tables": ["1001"],
  "field_mapping": {
    "销售额": "1001.pay_amount",
    "日期": "1001.order_date",
    "品牌": "1001.brand_name"
  },
  "matched_requirements": ["实际成交金额", "下单日期", "品牌"],
  "missing_requirements": [],
  "ambiguities": []
}
```

`decision` 只允许四种结果：`single` 表示一张表能完整回答，`multi` 表示用户需求必须组合多张表，`clarify` 表示存在多种合理业务口径需要用户确认，`no_match` 表示没有表满足必要条件。比起让 LLM 输出未经校准的小数分数，这里更应关注它找到了哪些字段、缺少什么信息以及为什么存在歧义。

例如同时存在“订单明细”和“流量归因”两张表时，用户问“分析苹果渠道引导销售额趋势”，LLM 应选择流量归因表；用户只问“分析苹果销售额趋势”时，实际成交金额和渠道引导成交金额都能解释“销售额”，LLM 应返回 `clarify`，而不是猜测一张表。用户明确要求对比两种销售额时，应返回 `multi`，然后进入第 12 章的跨表关联和粒度校验。

### 单表列数过多时

表数量少不代表 Schema 一定小。某张表有几百或上千列时，不应将全部字段详情一次性放入 prompt，也不应简单截断前 N 列。这时可以采用与 Skill 渐进式披露相同的分层加载方式：

```text
用户 query
  → 加载所有候选表的能力摘要
  → LLM 选择表和字段组
  → 只加载选中字段组的具体列
  → LLM 完成字段映射
  → 程序补充结构字段并校验
  → 绑定表和字段执行
```

选表阶段只展示宽表的能力摘要和字段组目录：

```json
{
  "table_id": "2001",
  "name": "商品经营宽表",
  "description": "商品日粒度的交易、流量、库存和营销经营数据",
  "grain": ["日期", "SKU"],
  "column_count": 860,
  "column_groups": [
    {
      "name": "基础维度",
      "description": "日期、SKU、SPU、品牌、类目和店铺"
    },
    {
      "name": "交易指标",
      "description": "成交金额、订单量、销量、客单价和退款金额"
    },
    {
      "name": "流量指标",
      "description": "曝光UV、点击UV、访问UV、渠道引导成交和转化率"
    },
    {
      "name": "库存指标",
      "description": "库存量、可售库存和周转天数"
    }
  ]
}
```

对于“分析近30天苹果的渠道引导销售额趋势”，LLM 先返回：

```json
{
  "selected_table": "2001",
  "required_column_groups": ["基础维度", "流量指标"]
}
```

系统再加载这两个字段组的具体列，LLM 最终只需映射 `stat_date`、`brand_name` 和 `guided_pay_amount` 等必要字段。字段组仍然过大时，可以每 50 至 100 列分成一批，每批都携带相同的 query 和表摘要让 LLM 召回相关列，合并各批结果后再进行一次最终字段选择。

这个过程可以通过类似 Skill 加载的元信息工具实现：

```text
list_user_tables()
get_table_profile(table_id)
list_table_column_groups(table_id)
get_table_columns(table_id, groups)
get_table_relations(table_ids)
```

LLM 首先看到的是表目录，只在需要时调用工具加载字段组和关系。渐进式加载只负责控制上下文，不代替最终校验。程序仍然要自动补充主键、时间字段、数据粒度字段、多表关联键、指标计算依赖字段和权限过滤字段，并确认 LLM 返回的表和字段都真实存在。

### 选表后的最小程序校验

即使候选表很少，也不应将全部决策权交给 LLM。程序至少要校验：LLM 返回的表属于当前用户、字段映射真实存在、`single` 只选择一张表、`multi` 选择多张表且存在可用关系、`clarify` 不会继续执行 SQL，以及最终执行只使用已选中的表。

是否能一次把 Schema 交给 LLM，应按元信息 token 数而不是单纯按表数量判断。可以将 Schema 元信息预算限制在模型有效上下文的 25% 至 30%；超过预算后，进入“表摘要 → 字段组 → 具体字段”的渐进式加载。当用户空间的表数量或字段总量继续增长时，再在 LLM 精选前增加候选召回层。

因此，这个问题的标准回答是：**用户空间的候选表很少且 Schema 元信息在 token 预算内时，可以直接由 LLM 读取详细元信息并输出 `single/multi/clarify/no_match` 选择结果；单表列数很多时，则采用类似 Skill 渐进式披露的“表摘要、字段组、具体字段”分层加载，最后由程序校验并绑定执行。**
