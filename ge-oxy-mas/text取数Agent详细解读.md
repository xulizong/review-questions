# Text取数Agent详细解读

> 本文基于 `master@4cf2ba6e` 的静态代码分析，未实际调用 LLM、数据工具或 Elasticsearch。
>
> 分析范围限定为：
>
> - `Text取数Agent` 的注册参数；
> - `ProgressiveReActAgent` 的执行循环；
> - 与该 Agent 直接相关的记忆机制；
> - `func_parse_llm_response_for_tool` 的解析、Judge 和后处理流程。
>
> 证据口径：Agent 和模型选择来自 **registration/config**；执行顺序、状态变化和降级行为来自 **implementation**；文中的业务数据与 JSON 示例均为帮助理解的教学示例，不代表已经运行验证。

## 1. 先建立一个简单的整体认识

`Text取数Agent` 不是只负责“生成 SQL”的单步 Agent，而是一个多轮执行控制器：它让 LLM 根据当前上下文决定调用什么工具，把工具结果写回上下文，再继续决定下一步，直到生成最终结论。

可以先把它理解成下面这条正常主链路；`trust_mode` 和最大轮数兜底属于后面单独说明的绕过路径：

```text
process_expert 准备任务、数据源和工具说明
    ↓
调用 Text取数Agent
    ↓
text2data_llm 决定下一步
    ├─ 调用 text2data_tool：查询数据
    ├─ 调用 py_coder_tool：加工临时文件
    ├─ 调用其他业务工具
    └─ 调用 summary_tool：提交最终总结
           ↓
       自定义解析器
           ├─ Judge
           ├─ 数值溯源
           ├─ 数值标签化
           ├─ 数据源和 Markdown 修复
           └─ 返回最终 ANSWER
    ↓
memory_llm 压缩本轮问答，供后续对话使用
```

需要先记住三个角色：

| 角色 | 负责什么 | 不负责什么 |
|---|---|---|
| `ProgressiveReActAgent` | 控制多轮 LLM 决策、工具执行、结果回填和终止 | 不直接生成 SQL，也不直接执行 Judge 规则 |
| `text2data_llm` | 根据 Prompt 和工具结果决定下一步，并生成工具调用 JSON 或最终总结 | 不直接执行工具 |
| `func_parse_llm_response_for_tool` | 将 LLM 输出解析为 `TOOL_CALL`、`ERROR_PARSE` 或 `ANSWER`，并处理 `summary_tool` | 不负责前面的数据查询本身 |

注册入口见 [`run_master_0130.py:406`](applications/master_agent/run_master_0130.py#L406)，上游 `process_expert` 的调用见 [`data_process_workflow.py:374`](applications/data_process/agents/data_process_workflow.py#L374)。

## 2. 注册参数逐项解读

当前注册代码如下：

```python
oxy.ProgressiveReActAgent(
    name="Text取数Agent",
    desc="Text取数Agent",
    llm_model="text2data_llm",
    llm_model_backup="text2data_llm_backup",
    llm_model_memory="memory_llm",
    prompt="",
    memory_prompt=memory_prompt,
    memory_max_tokens=1024 * 64,
    func_parse_llm_response=func_parse_llm_response_for_tool,
    is_retain_master_short_memory=True,
    max_react_rounds=20,
    retries=3,
    sub_agents=["node_judge_agent"],
    tools=[
        "text2data_tool",
        "py_coder_tool",
        "web_search_tool",
        "metric_analyze_tool",
        "judge_tool",
    ],
)
```

各参数的实际作用如下。

| 参数 | 当前作用 | 需要注意的边界 |
|---|---|---|
| `name` | Agent 在 MAS 中的唯一名称，上游通过这个名称调用它 | `process_expert` 使用 `callee="Text取数Agent"` 发起调用 |
| `desc` | 提供给其他 Agent 的简要能力说明 | 当前只写了名称，没有详细描述能力边界 |
| `llm_model` | ReAct 每轮的主决策模型，即 `text2data_llm` | 当前温度为 `0.1` |
| `llm_model_backup` | 主模型返回框架指定的友好错误文本时使用的备用模型 | 不是所有空输出、短输出或业务错误都会触发备用模型 |
| `llm_model_memory` | 最终答案成功后，负责压缩本轮问答的独立模型 | 不参与取数、工具选择、总结生成和 Judge |
| `prompt=""` | 构造 Agent 时会先采用框架默认 Prompt；实际调用时再由 `data_process_workflow` 动态传入场景 Prompt | `ProgressiveReActAgent` 每轮会用 `arguments["prompt"]` 覆盖自身 Prompt |
| `memory_prompt` | 指导 `memory_llm` 如何压缩“用户问题 + 最终答案” | 它不是业务 System Prompt |
| `memory_max_tokens` | 将每轮送入主模型的 `full_memory` 尽量限制在 65536 token 内 | tokenizer 不可用时会返回原消息，因此不是绝对硬上限 |
| `func_parse_llm_response` | 使用项目自定义解析器代替框架默认解析器 | `summary_tool`、Judge、数值溯源都从这里进入 |
| `is_retain_master_short_memory` | 额外加载由 `call_stack[:2]` 确定的上层会话历史到 `master_short_memory` | 当前执行循环没有读取这个字段，详见第 4.7 节 |
| `max_react_rounds=20` | 控制 ReAct 循环轮数 | 代码使用 `range(max_react_rounds + 1)`，实际最多进行 21 次循环内 LLM 决策 |
| `retries=3` | 未捕获异常逃出 `_execute` 时，框架最多执行 3 次整体尝试 | `ERROR_PARSE` 不属于外层异常重试，而是在同一次 ReAct 循环内纠正 |
| `sub_agents` | 将 `node_judge_agent` 声明为可调用子 Agent | 声明关系不会自动执行它 |
| `tools` | 声明该 Agent 可调用的工具权限 | `summary_tool` 不在这里，因为它不是一个真实注册工具 |

主模型、备用模型和记忆模型的注册见 [`run_master_0130.py:121`](applications/master_agent/run_master_0130.py#L121)，三个模型当前温度均为 `0.1`。

`node_judge_agent` 和 `judge_tool` 同时出现在配置中并不重复：前者根据本次任务生成动态审核规则和待评估上下文；随后解析器使用原始 `Text取数Agent` 请求显式调用 `judge_tool` 得到 `pass` 结果，因此 `Text取数Agent` 自身也需要拥有 `judge_tool` 的调用权限。实际调用见 [`llm_response_parser.py:642`](applications/data_process/util/llm_response_parser.py#L642)。

## 3. `ProgressiveReActAgent` 详细解读

### 3.1 它和普通 `ReActAgent` 的关系

两者底层都是同一个基本循环：

```text
LLM 决策
    ↓
解析输出
    ↓
执行工具
    ↓
把工具结果写回上下文
    ↓
再次调用 LLM
```

`ProgressiveReActAgent` 在普通 ReAct 基础上增加了以下能力：

1. 每轮刷新 workspace 和当前工具说明；
2. 通过 `get_tool_documentation` 按需切换工具详细文档；
3. 支持主模型和备用模型；
4. 支持异步自定义解析器；
5. 成功结束后可以调用记忆模型压缩问答；
6. 支持从压缩历史中读取旧问答。

普通 `ReActAgent` 的循环见 [`react_agent.py:320`](oxygent/oxy/agents/react_agent.py#L320)，渐进版的循环见 [`progressive_react_agent.py:494`](oxygent/oxy/agents/progressive_react_agent.py#L494)。

需要特别说明：`ProgressiveReActAgent` 并不意味着“自动逐个执行子 Agent”。这里的 Progressive 主要体现在上下文、workspace 和工具说明会随着执行过程逐步变化。

### 3.2 调用前由 `process_expert` 准备什么

`data_process_workflow` 在调用 `Text取数Agent` 前，会先完成以下准备：

1. 根据数据源判断是在线数据还是本地文件；
2. 根据“查数”或“分析”场景决定可用业务工具；
3. 构造 `tool_description_dict`；
4. 选出初始业务工具；
5. 动态拼装场景 Prompt；
6. 把数据源、知识、workspace、工具列表和 Prompt 一起传入 Agent。

核心参数形态如下：

```python
arguments = {
    "query": query,
    "dataset_list": dataset_list,
    "dataset_id_list": dataset_id_list,
    "dataset_markdown": dataset_markdown,
    "file_list": file_list,
    "file_markdown": file_markdown,
    "knowledge": knowledge,
    "now_time": now_time,
    "workspace": scene_workspace,
    "metric_priority_hint": metric_priority_hint,
    "prompt": scene_prompt,
    "tool_list": scene_tool_list,
    "current_tool_description": initial_tool_desc,
    "current_tool_name": initial_tool_name,
}
```

这部分实现在 [`data_process_workflow.py:305`](applications/data_process/agents/data_process_workflow.py#L305)。

### 3.3 每一轮送给 LLM 的上下文

每轮都会重新构造 `full_memory`：

```text
1. 动态 System Prompt
2. short_memory：跨调用历史问答
3. 当前 query
4. react_memory：本次调用已经发生的工具调用和纠错反馈
5. 按 memory_max_tokens 截断
```

对应结构可以近似理解为：

```python
full_memory = [
    {
        "role": "system",
        "content": "动态业务 Prompt + 当前工具详细说明 + workspace"
    },
    # 历史问答
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."},
    # 当前问题
    {
        "role": "user",
        "content": "我的最新问题是：...禁止自行计算..."
    },
    # 当前调用已经发生的 ReAct 轨迹
    {"role": "assistant", "content": "工具调用 JSON"},
    {"role": "user", "content": "Tool [...] execution result: ..."},
]
```

构造代码见 [`progressive_react_agent.py:509`](oxygent/oxy/agents/progressive_react_agent.py#L509)，token 截断见 [`progressive_react_agent.py:578`](oxygent/oxy/agents/progressive_react_agent.py#L578)。

截断策略是：

- 保留 System 消息；
- 在剩余 token 预算内，从后向前优先保留最近消息；
- 如果 System Prompt 本身已经超限，只保留第一条 System 消息；
- tokenizer 无法加载时，不截断。

实现见 [`tiktoken_utils.py:459`](applications/data_process/tools/tiktoken_utils.py#L459)。

### 3.4 一轮 LLM 决策之后有三种状态

主模型输出会交给 `func_parse_llm_response_for_tool`，解析器返回三种主要状态：

| 状态 | 含义 | `ProgressiveReActAgent` 的处理 |
|---|---|---|
| `TOOL_CALL` | LLM 想调用工具或 Agent | 执行调用，把结果包装成 Observation，进入下一轮 |
| `ERROR_PARSE` | JSON 格式、参数或业务校验不通过 | 把原输出和错误提示写入 `react_memory`，进入下一轮 |
| `ANSWER` | 已获得最终答案 | 可选执行记忆压缩，然后返回 `COMPLETED` |

状态处理见 [`progressive_react_agent.py:636`](oxygent/oxy/agents/progressive_react_agent.py#L636)。

外部工具自身返回 `FAILED` 时，当前循环不会自动结束整个 Agent，而是把失败响应同样包装成 Observation 交给下一轮 LLM，由模型决定换参数、换工具或结束，见 [`progressive_react_agent.py:749`](oxygent/oxy/agents/progressive_react_agent.py#L749)。

#### `TOOL_CALL` 示例

下面只是教学示意，具体参数以运行时工具 description 为准：

```json
{
  "tool_name": "text2data_tool",
  "arguments": {
    "query": "查询 2026-07-01 A 店销售额和订单量",
    "dataset_id": "dataset_demo_001",
    "title": "A店2026年7月1日销售数据"
  }
}
```

工具执行结果会被写成类似：

```text
Tool [text2data_tool] execution result:
销售额=120000，订单量=300
```

下一轮 LLM 可以同时看到原始工具调用和这条结果。

#### `ERROR_PARSE` 示例

如果 LLM 一次输出两个 JSON：

```text
{"tool_name":"text2data_tool", ...}
{"tool_name":"summary_tool", ...}
```

解析器会返回错误提示：

```text
输出中检测到多个 JSON 结构，每次只允许返回一个 JSON。
```

下一轮的上下文会包含：

```text
Assistant: <这是一段之前不合规的输出>
User: 输出中检测到多个 JSON 结构……
```

这样 LLM 能够根据错误提示重新输出。

#### `ANSWER` 示例

当解析器成功处理 `summary_tool` 后，会返回：

```python
LLMResponse(
    state=LLMState.ANSWER,
    output="最终处理后的业务结论",
)
```

Agent 随后结束本次 ReAct 循环。

### 3.5 “渐进式工具切换”是怎么工作的

业务 Prompt 声明“同一时刻只使用一个业务工具”。初始工具通常根据场景选择：

- 在线数据场景通常从 `text2data_tool` 开始；
- 本地文件场景通常从 `py_coder_tool` 开始；
- 命中指标分析路由时，可以优先从 `metric_analyze_tool` 开始。

假设当前工具是 `text2data_tool`，而下一步需要 Python 加工，LLM 先输出：

```json
{
  "tool_name": "get_tool_documentation",
  "arguments": {
    "target_tool": "py_coder_tool"
  }
}
```

这个调用不会进入外部 MAS 工具，而是在 `ProgressiveReActAgent` 内部完成：

```text
读取 target_tool
    ↓
从 tool_description_dict 获取 py_coder_tool 说明
    ↓
更新 shared_data.current_tool_name
    ↓
更新 shared_data.current_tool_description
    ↓
返回“已切换到工具 py_coder_tool”
```

下一轮重建 System Prompt 时，LLM 才会看到 `py_coder_tool` 的完整详细说明。

实现见 [`progressive_react_agent.py:683`](oxygent/oxy/agents/progressive_react_agent.py#L683)。

这里需要区分“设计约束”和“代码硬约束”：

- Prompt 要求先切换，再调用目标业务工具；
- 但普通外部工具执行分支没有检查 `tool_name == current_tool_name`；
- 自定义解析器也没有检查调用 `summary_tool` 前是否已经切换；
- 因此“当前只能调用一个业务工具”主要是 Prompt 约束，而不是不可绕过的执行闸门。

还有一个切换失败边界：目标工具虽然已注册、但当前场景的 `tool_description_dict` 中没有对应说明时，代码会先更新 `current_tool_name`，然后返回“未找到工具文档”，却不会回滚工具名，可能形成“新工具名 + 旧工具说明”的不一致状态，见 [`progressive_react_agent.py:693`](oxygent/oxy/agents/progressive_react_agent.py#L693)。

### 3.6 三类工具的区别

| 类型 | 例子 | 执行方式 |
|---|---|---|
| 内部系统工具 | `get_tool_documentation`、`retrieve_memory_detail` | 在 `ProgressiveReActAgent` 内部直接处理 |
| 外部工具或子 Agent | `text2data_tool`、`py_coder_tool`、`node_judge_agent` | 通过 `oxy_request.call()` 调用 |
| 虚拟结束协议 | `summary_tool` | 不执行真实工具，由自定义解析器截获 |

`summary_tool` 不在注册的 `tools` 列表中，但 `get_tool_documentation` 对它做了特殊放行。模型生成 `summary_tool` JSON 后，自定义解析器直接进入最终校验链，不会执行：

```python
oxy_request.call(callee="summary_tool")
```

### 3.7 主模型、备用模型和记忆模型

三个模型职责完全不同：

```text
text2data_llm
    └─ 每轮业务决策、工具选择、总结生成

text2data_llm_backup
    └─ 主模型返回 friendly_error_text 时备用

memory_llm
    └─ 最终 ANSWER 后压缩“用户问题 + 最终答案”
```

主备选择见 [`progressive_react_agent.py:597`](oxygent/oxy/agents/progressive_react_agent.py#L597)。

当前备用模型触发条件很窄：

```python
primary_failed = output_primary == friendly_error_text
```

也就是说，下面这些情况不一定会触发备用模型：

- 输出为空但不等于 `friendly_error_text`；
- 输出格式错误；
- 输出业务结论错误；
- 工具选择错误；
- Judge 不通过。

这些问题通常由自定义解析器和 ReAct 重试处理，而不是切换备用模型。

如果主、备模型都返回各自的 `friendly_error_text`，当前分支中的“改选主模型”条件实际上不可达，最终仍会选择备用模型的错误输出，再交给解析器处理。这是当前主备兜底的实现边界，见 [`progressive_react_agent.py:606`](oxygent/oxy/agents/progressive_react_agent.py#L606)。

### 3.8 轮数、外层重试和最终兜底

这里存在三个不同层次的“重试”，不要混在一起。

| 层次 | 触发原因 | 当前行为 |
|---|---|---|
| ReAct 下一轮 | 工具执行完成或 `ERROR_PARSE` | 保留当前 `react_memory`，再次调用 LLM |
| `retries=3` | 未捕获异常逃出整个 `_execute` | 框架最多执行 3 次整体尝试 |
| 最大轮数兜底 | ReAct 循环耗尽仍未得到 `ANSWER` | 收集 `react_memory` 中的 user-role Observation 和纠错反馈，额外调用一次主模型直接生成答案 |

最大轮数需要特别注意：

```python
for current_round in range(self.max_react_rounds + 1):
```

因此 `max_react_rounds=20` 实际最多进行 21 次循环内 LLM 决策。循环耗尽后还会额外调用一次主模型生成兜底答案。

兜底路径不会经过：

- `func_parse_llm_response_for_tool`；
- `summary_tool` 后处理；
- `node_judge_agent`；
- 数值溯源；
- 备用模型；
- `memory_llm` 压缩。

兜底实现见 [`progressive_react_agent.py:793`](oxygent/oxy/agents/progressive_react_agent.py#L793)。

另外，如果模型在工具调用 JSON 顶层输出 `"trust_mode": 1`，当前实现可以直接把工具 Observation 当作最终结果返回，也会绕过 `summary_tool`。默认 `trust_mode=False`，但模型字段仍可触发这条路径，见 [`progressive_react_agent.py:760`](oxygent/oxy/agents/progressive_react_agent.py#L760)。

## 4. Text取数Agent 的 Memory 机制

### 4.1 不要把所有 Memory 理解成同一个东西

当前链路中至少存在五类容易混淆的上下文或记忆：

| 名称 | 生命周期 | 内容 | 主要用途 |
|---|---|---|---|
| `short_memory` | 跨多次 Agent 调用 | 历史 query/answer | 让当前问题参考同一 Agent 会话的历史 |
| `master_short_memory` | 跨调用链历史 | 由 `call_stack[:2]` 拼出的会话历史 | 当前代码会加载，但没有注入主 ReAct 上下文 |
| `react_memory` | 当前一次 Text取数Agent 调用 | 工具调用、Observation、解析错误反馈 | 支撑本次多轮 ReAct |
| `full_memory` | 当前一轮 LLM 调用 | System + short memory + 当前问题 + react memory | 实际送给主/备 LLM，也供 Parser 和 Judge 使用 |
| `compressed_memory` | 当前调用结束后产生，供未来使用 | 当前 query 和最终 answer 的极简摘要 | 减少更早历史对上下文的占用 |

它们之间的关系如下：

```text
历史存储
    ↓
short_memory
    ┐
动态 System Prompt ─┐
当前 query ─────────┼─→ full_memory ─→ text2data_llm
react_memory ───────┘

最终 ANSWER
    ↓
memory_llm + memory_prompt
    ↓
compressed_memory
    ↓
写入历史，供未来 short_memory 使用
```

### 4.2 `short_memory`：跨调用历史

`data_process_workflow` 调用 `Text取数Agent` 时没有显式传入 `short_memory`，因此 `LocalAgent._pre_process()` 会查询历史存储。

查询条件包括：

- 当前应用的 history 索引；
- 当前 trace 链；
- 当前 `session_name`；
- 最近 `short_memory_size` 条记录。

当前 `Text取数Agent` 没有单独配置 `short_memory_size`，因此继承默认值 `10`，见 [`config.py:96`](oxygent/config.py#L96)。这里表示最近 10 条 history 记录，不一定等于用户界面中的 10 轮对话，因为一个用户请求可以执行多个 GraphPlan 步骤，每个步骤都可能单独调用一次 `Text取数Agent`。

对这条调用链而言，普通 `session_name` 由：

```python
self.caller + "__" + self.callee
```

生成，通常对应：

```text
process_expert__Text取数Agent
```

历史读取见 [`progressive_react_agent.py:138`](oxygent/oxy/agents/progressive_react_agent.py#L138)，`session_name` 定义见 [`oxy.py:125`](oxygent/schemas/oxy.py#L125)。

### 4.3 `react_memory`：本次调用的工作轨迹

每次进入 `_execute()`，都会创建新的空 `react_memory`：

```python
react_memory = Memory()
```

工具执行完成后写入：

```text
Assistant: 本轮工具调用 JSON
User: Tool [工具名] execution result: 工具结果
```

存在两个例外：当 `trust_mode` 提前返回时，本轮工具调用和 Observation 尚未写入 `react_memory`；成功的最终 `summary_tool` 输出直接变成 `ANSWER`，也不会再追加到 `react_memory`。

解析失败时写入：

```text
Assistant: LLM 原始输出
User: 解析器返回的错误或修订提示
```

因此，`react_memory` 的主要作用是让同一次调用中的下一轮 LLM 看见：

- 自己之前调用了什么；
- 工具返回了什么；
- 上一次输出为什么被拒绝；
- Judge 提出了什么修订意见。

写入逻辑见 [`progressive_react_agent.py:774`](oxygent/oxy/agents/progressive_react_agent.py#L774)。

正常结束时，`react_memory` 会放入 `OxyResponse.extra`。上游 `data_process_workflow` 会从中提取工具结果，并把最终输出作为 `conclusion`，见 [`data_process_workflow.py:379`](applications/data_process/agents/data_process_workflow.py#L379)。

### 4.4 `full_memory`：当前轮实际使用的上下文

`full_memory` 不是后续会话会重新恢复的长期对话记忆，而是每轮临时组装的消息快照。它写入 `oxy_request.arguments` 后，仍可能作为节点 input 被追踪存储持久化，见 [`base_oxy.py:536`](oxygent/oxy/base_oxy.py#L536)。

它同时服务三个地方：

1. 作为主模型和备用模型的输入；
2. 写入 `oxy_request.arguments["full_memory"]`；
3. 供 `summary_tool` 的 Judge 和数值溯源使用。

其中数值溯源接收当前 `full_memory`；Judge 则使用 `full_memory[1:]`，即去掉第一条 System Prompt 后，将剩余消息作为 `node_judge_agent` 的 `short_memory`，见 [`llm_response_parser.py:638`](applications/data_process/util/llm_response_parser.py#L638)。

需要注意：当前 LLM 刚生成的输出还不在本轮 `full_memory` 中。只有该输出被写入 `react_memory` 后，下一轮重建 `full_memory` 时才会出现。

### 4.5 `compressed_memory` 与 `memory_prompt`

只有解析器返回正常 `ANSWER` 后，才会调用：

```python
compressed_memory = await self._compress_memory(
    llm_response.output,
    oxy_request,
)
```

传给 `memory_llm` 的内容只有：

```text
User: 当前 oxy_request.get_query()
Assistant: 最终 llm_response.output
```

压缩模型不会直接读取：

- 本轮完整 `react_memory`；
- SQL；
- Python 代码；
- 所有工具调用详情；
- Judge 的完整过程。

如果这些内容被上游直接写进了最终 answer，压缩模型仍然能够看到，因为它接收的 Assistant 输入就是最终 answer。

`memory_prompt` 要求模型输出：

```json
{
  "query": "用户问题的一句话摘要",
  "answer": "最终答案的一句话摘要"
}
```

同时尽量保留日期、筛选条件、数据版本、SKU、排名和关键指标等信息。Prompt 见 [`memory_prompts.py:1`](applications/data_process/prompts/memory_prompts.py#L1)，调用见 [`progressive_react_agent.py:449`](oxygent/oxy/agents/progressive_react_agent.py#L449)。

压缩成功后，返回结果大致为：

```python
OxyResponse(
    output="最终业务结论",
    extra={
        "react_memory": [...],
        "compressed_memory": {
            "query": "用户查询2026年7月1日A店销售额和订单量",
            "answer": "助手返回销售额120000元、订单量300单"
        }
    }
)
```

框架保存历史时，会把 `extra` 合并到 history 中，见 [`base_agent.py:198`](oxygent/oxy/agents/base_agent.py#L198)。这依赖 `is_save_history=True` 且 Elasticsearch 可用；节点存储默认可以异步执行，因此静态代码不能保证答案返回后历史一定立即可读。

### 4.6 后续对话如何读取压缩记忆

当前 `is_discard_react_memory` 使用默认值 `True`，因此读取历史时：

- 最新一条历史使用原始 query/answer；
- 更早的历史如果有 `compressed_memory`，优先使用压缩内容；
- 不会恢复更早历史的完整 `react_memory`；
- 压缩失败或没有压缩内容时，回退到原始 query/answer。

实现见 [`progressive_react_agent.py:229`](oxygent/oxy/agents/progressive_react_agent.py#L229)。

#### 三次 Text取数Agent 调用示例

假设三次调用位于可关联的 trace 链中，并且历史已经成功写入和检索：

```text
第1次调用：
用户：查询7月1日A店销售额
答案：销售额为100万元

第2次调用：
用户：查询7月2日A店销售额
答案：销售额为120万元

第3次调用：
用户：比较这两天
```

第 3 次调用的 `full_memory` 大致是：

```text
System Prompt
  + 第1次调用的 compressed_memory
  + 第2次调用的原始 query/answer
  + 第3次调用的当前问题
  + 第3次调用中新产生的 react_memory
```

第 1、2 次调用的 SQL 和工具 Observation 不会完整恢复。真实项目中，这里的 query 也可能不是用户原话，而是当前 GraphPlan 步骤的 `title + step_command`。

### 4.7 `is_retain_master_short_memory=True` 的当前实际效果

这个配置会额外查询由 `call_stack[:2]` 拼出的会话历史，并写入：

```python
oxy_request.arguments["master_short_memory"]
```

但 `ProgressiveReActAgent` 组装 `full_memory` 时使用的是：

```python
oxy_request.get_short_memory()
```

而不是：

```python
oxy_request.get_short_memory(master_level=True)
```

因此，按照当前代码：

```text
master_short_memory 被加载
    ↓
保存在 arguments 中
    ↓
主 ReAct 上下文没有读取
```

也就是说，这个配置目前没有把这份上层调用链历史真正注入 `Text取数Agent` 的主模型输入。这里的会话名只是机械使用 `call_stack` 前两项，不保证其中一定包含 `master_agent`。字段加载见 [`local_agent.py:365`](oxygent/oxy/agents/local_agent.py#L365)，实际读取见 [`progressive_react_agent.py:541`](oxygent/oxy/agents/progressive_react_agent.py#L541)。

它还带来一个需要留意的缓存覆盖问题：

```text
先读取 Text取数Agent 自己的历史
    → 写入 shared_data["history_cache"]
    → short_memory 中记录较早历史的 MemoryID

再读取 master 历史
    → 再次写入同一个 shared_data["history_cache"]
    → 前一份 Text取数Agent 历史缓存被覆盖
```

而内部 `retrieve_memory_detail(memory_id)` 只会查询这一个 `history_cache`。因此，模型使用普通 `short_memory` 中的 Text 会话 MemoryID 回查详情时，该 ID 可能已经不在缓存中并返回 `not found`；即使命中，当前接口也只返回原始 query/answer，不恢复历史 ReAct 轨迹。覆盖写入见 [`progressive_react_agent.py:225`](oxygent/oxy/agents/progressive_react_agent.py#L225)，详情查询见 [`progressive_react_agent.py:375`](oxygent/oxy/agents/progressive_react_agent.py#L375)。

## 5. `func_parse_llm_response_for_tool` 详细解读

### 5.1 它在整个流程中的位置

每轮主模型或备用模型返回字符串后，`ProgressiveReActAgent` 会执行：

```python
llm_response = self.func_parse_llm_response(
    oxy_response.output,
    oxy_request,
)

if inspect.isawaitable(llm_response):
    llm_response = await llm_response
```

因此这个函数相当于 LLM 输出与执行框架之间的“路由器”：

```text
LLM 原始字符串
    ↓
func_parse_llm_response_for_tool
    ├─ TOOL_CALL：交给执行框架
    ├─ ERROR_PARSE：反馈给 LLM 重试
    └─ ANSWER：结束 ReAct
```

调用点见 [`progressive_react_agent.py:625`](oxygent/oxy/agents/progressive_react_agent.py#L625)，解析器入口见 [`llm_response_parser.py:359`](applications/data_process/util/llm_response_parser.py#L359)。

### 5.2 前置格式处理

解析器首先做两件事。

#### 去掉 Think 内容

如果输出中包含 `</think>`，只保留最后一个 `</think>` 后面的内容：

```text
<think>这里是模型思考</think>
{"tool_name":"text2data_tool", ...}
```

最终只解析 JSON 部分。

#### 拒绝同一轮多个 JSON

解析器先统计 JSON 块数量。如果同一次输出出现多个 JSON，则直接返回 `ERROR_PARSE`。

目的不是只为了解析方便，还为了维持“一轮只执行一个动作”的 ReAct 约束。

当检测到多 JSON 时，原始输出不会完整写入下一轮，而会替换为：

```text
<这是一段之前不合规的输出>
```

避免错误 JSON 中的数字、工具调用或 Summary 污染下一轮上下文。实现见 [`llm_response_parser.py:383`](applications/data_process/util/llm_response_parser.py#L383)。

JSON 提取规则为：

1. 优先提取 ```` ```json ... ``` ````；
2. 没有代码块时，截取第一个 `{` 到最后一个 `}`；
3. 再执行 `json.loads()`。

实现见 [`llm_response_parser.py:213`](applications/data_process/util/llm_response_parser.py#L213)。

### 5.3 普通工具分支

除 `summary_tool` 和 `py_coder_tool` 外，其他带 `tool_name` 的输出会直接返回：

```python
LLMResponse(
    state=LLMState.TOOL_CALL,
    output=tool_call_dict,
    ori_response=ori_response,
)
```

例如：

```json
{
  "tool_name": "text2data_tool",
  "arguments": {
    "query": "查询昨日销售额",
    "dataset_id": "dataset_demo_001",
    "title": "昨日销售额"
  }
}
```

解析器不会在这个分支检查：

- 工具名是否真实存在；
- 是否是当前 `current_tool_name`；
- 参数是否完整；
- SQL 是否正确；
- 工具是否适合当前任务。

这些问题会由 Prompt、权限检查或后续工具执行暴露。分支见 [`llm_response_parser.py:435`](applications/data_process/util/llm_response_parser.py#L435)。

### 5.4 `py_coder_tool` 专项校验

如果 `tool_name == "py_coder_tool"`，解析器会检查 `arguments.code` 中是否出现：

```python
pd.read_csv(
```

或：

```python
pd.read_excel(
```

通过示例：

```json
{
  "tool_name": "py_coder_tool",
  "arguments": {
    "code": "df = pd.read_csv('oxy://file_1.csv')"
  }
}
```

不通过示例：

```json
{
  "tool_name": "py_coder_tool",
  "arguments": {
    "code": "df = previous_tool_result"
  }
}
```

不通过时会返回 `ERROR_PARSE`，要求模型从 workspace 文件读取数据。

这是一个字符串正则校验，而不是 Python AST 或真实数据来源校验。因此：

- 注释中出现 `# pd.read_csv(` 也可能通过；
- 非 workspace 地址也可能通过；
- `pandas.read_csv()` 不会通过；
- `pd.read_parquet()` 不会通过。

实际校验见 [`llm_response_parser.py:313`](applications/data_process/util/llm_response_parser.py#L313)。

### 5.5 `summary_tool` 是虚拟结束协议

模型最终需要输出类似：

```json
{
  "tool_name": "summary_tool",
  "arguments": {
    "summary": "最终业务结论"
  }
}
```

解析器要求：

- `tool_name` 为 `summary_tool`；
- `arguments.summary` 存在且非空。

满足后，不会执行真实的 `summary_tool`，而是直接进入后处理链：

```text
候选 summary
    ↓
Step 0：AI Judge
    ↓
Step 1：数值溯源
    ↓
Step 2：数值标签化 LLM
    ↓
Step 3：数据源 ID 修复
    ↓
Step 4：Markdown 表格修复
    ↓
Step 5：数值标签渲染
    ↓
Step 6：“万万”替换为“亿”
    ↓
LLMState.ANSWER
```

主分支见 [`llm_response_parser.py:442`](applications/data_process/util/llm_response_parser.py#L442)。

### 5.6 `summary_tool` description 和 Judge Prompt 分别给谁

这两个 Prompt 的使用者不同：

| 内容 | 使用者 | 使用时机 | 作用 |
|---|---|---|---|
| `summary_tool` description | `text2data_llm` | 生成候选 summary 之前 | 告诉“写答案的人”如何写 |
| `node_judge_agent` Prompt | `judge_llm` | 候选 summary 生成之后 | 告诉“审核答案的人”如何检查 |

遵守当前业务 Prompt 时的标准顺序是：

```text
get_tool_documentation(summary_tool)
    ↓
summary_tool description 进入 Text取数Agent 的 System Prompt
    ↓
text2data_llm 生成候选 summary_tool JSON
    ↓
解析器截获 summary_tool
    ↓
node_judge_agent 使用自己的 Prompt 审核候选 summary
```

因此 `summary_tool` description 不是给 Judge 使用，也不是工具执行之后才使用。它在候选答案生成之前已经影响了 `text2data_llm`。

这仍然是 Prompt 规定的标准路径，不是解析器硬前置条件；模型直接输出合法的 `summary_tool` JSON 时，解析器同样会进入后处理链。

### 5.7 Step 0：AI Judge

Judge 链路如下：

```text
候选 summary
    ↓
_run_summary_post_judge_non_intrusive
    ↓
run_summary_post_judge
    ↓
node_judge_agent
    ↓
生成本次任务的动态 rubric 和 eval_cot
    ↓
judge_tool
    ↓
pass=true / pass=false
```

`node_judge_agent` 主要负责根据本次任务生成动态检查规则，不是最终执行规则的工具；真正输出 `pass` 的通常是 `judge_tool`。

调用代码见 [`llm_response_parser.py:614`](applications/data_process/util/llm_response_parser.py#L614)，`node_judge_agent` 注册见 [`run_master_0130.py:645`](applications/master_agent/run_master_0130.py#L645)。

主路径之外还兼容两类返回：`node_judge_agent` 直接返回英文 `pass` 字段，或者返回中文“是否通过”字段；完全无法识别的返回按不通过处理。调用异常或超时则相反，会降级放行。

Judge 使用的是 `node_judge_agent` 生成的压缩 `eval_cot`，不是把原始 `full_memory` 原封不动交给 `judge_tool`，因此压缩过程可能损失证据。Judge Prompt 还可以被 DUCC 远端配置替换，当前仓库只能确认本地默认文本，配置入口见 [`laf_instance.py:259`](extends/ducc/laf_instance.py#L259)。

另外，英文兼容分支使用 `bool(tool_call_dict["pass"])` 转换；如果上游错误返回字符串 `"false"` 而不是布尔值 `false`，Python 会把非空字符串判断为 `True`。这是当前实现的格式边界。

Judge 主要检查：

- 用户要求的业务对象是否一致；
- 时间范围是否一致；
- 指标口径和单位是否一致；
- 输出形态是否符合要求；
- 中间数据能否支撑最终结论；
- PP、百分比等表达是否正确；
- 空结果、截断数据是否如实说明。

Judge 的动态 Prompt 见 [`node_judge_prompt.py:1`](applications/master_agent/agent/ai_hosting_add/node_judge_prompt.py#L1)，静态规则见 [`node_judge_prompt.py:62`](applications/master_agent/agent/ai_hosting_add/node_judge_prompt.py#L62)。

#### Judge 不通过后发生什么

Judge 不通过时，解析器返回 `ERROR_PARSE`，并把修订意见交给下一轮 `Text取数Agent`。

下一轮 LLM 会同时看到：

- 原来的 query；
- 原来的工具结果；
- 上一次候选 summary；
- Judge 失败原因；
- 要求局部修复或重新执行数据步骤的提示。

如果只是表达或格式错误，可以复用已有数据重写 summary；如果涉及查询、筛选、聚合、计算或口径错误，反馈明确要求重新调用相应 Data/SQL/Code 工具。

#### Judge 实际只执行一次

当前实现会在第一次调用 Judge 前执行：

```python
oxy_request.shared_data["not_pass_judge"] = not_pass_judge + 1
```

下一次进入 `summary_tool` 时，只要：

```python
not_pass_judge >= 1
```

就跳过 Judge。

所以一次 `Text取数Agent` 执行内：

- 第一次 Judge 不通过，修改后的第二版 summary 不会再次 Judge；
- 第一次 Judge 通过，但后面的数值溯源失败，第二版 summary 也不会再次 Judge；
- Judge 调用异常并降级放行后，后续同样不会再 Judge；
- 成功返回最终 `ANSWER` 后，才会重置 Judge 计数。

这部分见 [`llm_response_parser.py:151`](applications/data_process/util/llm_response_parser.py#L151)。

### 5.8 Step 1：数值溯源

数值溯源的目标是检查候选 summary 中的数字是否能在 `full_memory` 中找到来源。

例如：

```text
full_memory 中出现：
销售额=100

候选 summary：
销售额为120
```

`120` 找不到来源，第一次会返回 `ERROR_PARSE`。

校验会跳过 Summary 中的部分非业务数字：

- 日期；
- Markdown 章节号；
- 列表序号；
- `<datasource>...</datasource>`；
- `<jmtCharts>...</jmtCharts>`。

匹配规则包括：

- 约 `1%` 的相对误差；
- `0.01` 的绝对误差兜底；
- 正负数绝对值匹配，例如工具返回 `-7.1`，总结写“下降 7.1%”。

实现见 [`number_source_validator.py:60`](applications/data_process/tools/number_source_validator.py#L60)。

#### 两次校验策略

```text
第1次数值溯源失败
    → ERROR_PARSE
    → LLM 重新生成 summary

第2次仍失败
    → 记录告警
    → 放行，继续后处理
```

第二次校验前，会删除历史中上一版 `summary_tool` JSON 和紧随其后的错误提示，避免上一版幻觉数字反过来成为“可信来源”。

需要注意两个边界：

1. 数字来源不是只检查工具 Observation，而是检查 `full_memory` 中所有非 System 消息，因此当前 query 和历史问答中的数字也可能被视为来源；
2. “第几次”按照历史 `summary_tool` 尝试次数统计，如果第一版 Summary 是被 Judge 拒绝的，第二版的数值溯源也可能直接被视为第二次尝试。

对应代码见 [`llm_response_parser.py:461`](applications/data_process/util/llm_response_parser.py#L461) 和 [`number_source_validator.py:200`](applications/data_process/tools/number_source_validator.py#L200)。

### 5.9 Step 2：数值标签化

数值溯源通过或被放行后，会调用独立的：

```text
extract_tag_llm
```

将自然语言数字转换为结构化数值标签，例如：

```text
销售额为 123456 元
```

转换为类似：

```text
销售额为 {num: 123456, format: "unit"} 元
```

这样后面的 `NumberTagProcessor` 可以统一处理原值、百分比、小数位和万/亿展示。

这个 LLM 与 `text2data_llm` 解耦，调用见 [`number_tag_llm_formatter.py:53`](applications/data_process/tools/number_tag_llm_formatter.py#L53)。

标签化失败时会沿用原始 summary，不阻塞主链路。当前用于验证“标签化结果中是否仍存在裸数字”的 `validate_summary_result()` 调用已经被注释，因此标签化失败后仍可能直接进入后续流程，见 [`llm_response_parser.py:536`](applications/data_process/util/llm_response_parser.py#L536)。

### 5.10 Step 3 到 Step 6：格式与展示处理

| 步骤 | 当前实际行为 | 失败策略 |
|---|---|---|
| 数据源 ID 修复 | 删除数值大于 `10000` 的整个 `<datasource>...</datasource>` 标签 | 异常时保留原文 |
| Markdown 修复 | 根据表格最大列数补齐表头和分隔行 | 异常时保留原文 |
| 数值标签处理 | 将 `{num: ...}` 标签渲染为最终数字展示 | 处理器整体异常时保留当前文本；单个标签无效、`num=null` 或格式化失败时可能替换为 `-` |
| 文本替换 | 将“万万”或“万 万”替换为“亿” | 直接字符串替换 |

这些步骤分别见：

- [`post_validation.py:44`](applications/data_process/tools/post_validation.py#L44)；
- [`markdown_repair.py:4`](applications/data_process/tools/markdown_repair.py#L4)；
- [`number_tag_processor.py:74`](applications/data_process/tools/number_tag_processor.py#L74)；
- [`llm_response_parser.py:583`](applications/data_process/util/llm_response_parser.py#L583)。

除明确返回 `ERROR_PARSE` 的 Judge 和第一次数字溯源外，大部分后处理异常都采用“记录日志并放行原文”的策略。

## 6. 一个完整的简单例子

下面是教学示例，用于说明控制流程，不代表 `text2data_tool` 的完整真实参数协议。

### 6.1 用户问题

```text
查询 2026-07-01 A 店的销售额和订单量，只输出表格，不要分析。
```

### 6.2 准备阶段

`data_process_workflow` 判断这是在线数据查询场景，准备：

```text
初始工具：text2data_tool
候选工具：text2data_tool、py_coder_tool、summary_tool
当前工具 description：text2data_tool 的详细说明
动态 Prompt：先取数，最后使用 summary_tool
```

如果当前请求还启用了指标分析或联网检索，候选列表还可能包含 `metric_analyze_tool`、`web_search_tool`。

### 6.3 第 1 轮：调用数据工具

`text2data_llm` 输出：

```json
{
  "tool_name": "text2data_tool",
  "arguments": {
    "query": "查询2026-07-01 A店销售额和订单量",
    "dataset_id": "dataset_demo_001",
    "title": "A店2026年7月1日销售数据"
  }
}
```

解析器返回 `TOOL_CALL`，框架执行数据工具。

假设工具返回：

```text
日期=2026-07-01
门店=A店
销售额=120000
订单量=300
```

这些结果进入 `react_memory`。

### 6.4 第 2 轮：切换到 summary_tool

LLM 已经获得所需数据，输出：

```json
{
  "tool_name": "get_tool_documentation",
  "arguments": {
    "target_tool": "summary_tool"
  }
}
```

`ProgressiveReActAgent` 更新：

```text
current_tool_name = summary_tool
current_tool_description = summary_tool 的详细说明
```

### 6.5 第 3 轮：生成第一版候选总结

下一轮 `text2data_llm` 根据 `summary_tool` description 输出：

```json
{
  "tool_name": "summary_tool",
  "arguments": {
    "summary": "| 日期 | 门店 | 销售额 | 订单量 |\n|---|---|---:|---:|\n| 2026-07-01 | A店 | 120000 | 300 |\n\nA店当天销售表现很好。"
  }
}
```

解析器截获 `summary_tool`，不会调用真实工具。

### 6.6 Judge 拒绝第一版

用户要求“只输出表格，不要分析”，但候选 Summary 添加了：

```text
A店当天销售表现很好。
```

Judge 可以据此返回：

```json
{
  "pass": false,
  "violations": [
    {
      "rule": "用户要求只输出表格，不要分析",
      "reason": "Summary 在表格后增加了主观评价"
    }
  ]
}
```

解析器将 Judge 意见转换成 `ERROR_PARSE`。下一轮 `Text取数Agent` 会看到原始数据和这条修订意见。

这里不需要重新查询数据，因为错误只涉及输出形态。

### 6.7 第 4 轮：局部修改总结

LLM 重新输出：

```json
{
  "tool_name": "summary_tool",
  "arguments": {
    "summary": "| 日期 | 门店 | 销售额 | 订单量 |\n|---|---|---:|---:|\n| 2026-07-01 | A店 | 120000 | 300 |"
  }
}
```

这一版不会再次执行 Judge，因为当前实现一次 Agent 执行内最多 Judge 一次。

随后：

1. 数值溯源检查 `120000` 和 `300` 是否出现在 `full_memory`；
2. `extract_tag_llm` 对数字进行标签化；
3. Markdown 修复器检查表格列数；
4. `NumberTagProcessor` 渲染最终数字；
5. 返回 `LLMState.ANSWER`。

标签化的中间结果可能类似：

```text
| 2026-07-01 | A店 | {num: 120000, format: "origin"} | {num: 300, format: "origin"} |
```

渲染完成后再返回普通 Markdown：

```text
| 日期 | 门店 | 销售额 | 订单量 |
|---|---|---:|---:|
| 2026-07-01 | A店 | 120000 | 300 |
```

### 6.8 成功结束后的记忆压缩

最终答案产生后，`memory_llm` 接收：

```text
User:
查询 2026-07-01 A 店的销售额和订单量，只输出表格，不要分析。

Assistant:
最终表格
```

可能压缩为：

```json
{
  "query": "用户要求以表格查询2026年7月1日A店销售额和订单量且不要分析",
  "answer": "助手返回A店当日销售额120000元、订单量300单的表格"
}
```

该内容作为 `compressed_memory` 保存，供未来历史读取使用。

### 6.9 如果 Judge 指出的是数据问题

假设 Judge 发现：

```text
用户要求 2026-07-01，但工具结果实际是 2026-07-02。
```

这时下一轮不应只修改 Summary 中的日期，而应：

```text
get_tool_documentation(text2data_tool)
    ↓
重新调用 text2data_tool
    ↓
获得正确日期的数据
    ↓
get_tool_documentation(summary_tool)
    ↓
重新生成 summary_tool
```

所以“Judge 不通过后数据一定不会重新生成”并不准确。是否重新取数取决于错误是表达问题还是数据、筛选、计算、口径问题。

## 7. 当前实现中最重要的边界

| 边界 | 当前实际行为 | 影响 |
|---|---|---|
| 工具切换 | 主要依赖 Prompt，没有硬校验当前工具名 | 模型理论上可以绕过 `get_tool_documentation` 直接调用已授权工具 |
| `summary_tool` | 虚拟解析协议，不是真实工具 | 正常主路径的最终答案依赖自定义解析器识别；`trust_mode` 和最大轮数兜底可以绕过 |
| Judge 次数 | 一次 Agent 执行内最多运行一次 | 修订后的第二版 Summary 不会再次 Judge |
| Judge 异常 | 降级放行 | Judge 不是绝对强制保证 |
| 数值溯源来源 | 检查所有非 System 消息 | 用户问题和历史问答里的数字也可能被当作来源 |
| 数值溯源重试 | 历史中已出现一次 `summary_tool` 后，本次失败即记录后放行 | 统计的是 Summary 总尝试次数，不是数值校验失败次数 |
| 数值溯源空证据 | memory 为空或没有可提取数字时直接放行 | “通过”不一定代表找到了真实工具来源 |
| 标签化 LLM | 主要通过非空和长度下限判断输出 | 仍可能改写数字之外的正文内容 |
| 标签化合规检查 | 当前调用被注释 | 标签化失败后可能携带裸数字继续返回 |
| `master_short_memory` | 被加载但未进入 `full_memory` | `is_retain_master_short_memory=True` 当前没有实现预期的上层调用链历史注入 |
| 历史详情缓存 | master 历史查询会覆盖普通历史的 `history_cache` | 使用普通历史 MemoryID 调用 `retrieve_memory_detail` 时可能查不到 |
| 最大轮次兜底 | 把所有 user-role ReAct 消息交给主模型直出 | Observation 和 `ERROR_PARSE` 提示会混在一起，并绕过 `summary_tool`、Judge、数值溯源和记忆压缩 |
| 外层 `retries` | 只处理逃出 `_execute` 的异常 | 不等同于 ReAct 的业务纠错轮次 |

## 8. 最后用一句话区分各模块

下面描述的是正常 `summary_tool` 主路径；提前 `trust_mode` 和最大轮数兜底不遵循全部步骤。

```text
ProgressiveReActAgent：
负责“循环怎么跑、工具怎么执行、结果怎么回填”。

业务 Prompt 与工具 description：
负责“text2data_llm 应该怎么做、怎么写”。

func_parse_llm_response_for_tool：
负责“LLM 这次输出代表调用工具、需要重试，还是可以结束”。

summary_tool：
是提交最终结论的虚拟协议。

node_judge_agent + judge_tool：
负责对候选总结生成规则并做一次后置审核。

memory_prompt + memory_llm：
负责在任务结束后压缩本轮问答，供未来对话使用。
```
