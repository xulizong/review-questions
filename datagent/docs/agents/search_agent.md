# Search Agent 找数 Agent

Search Agent 是 `AGENT=search-agent` 时的 master agent，代码入口是 `modules/agents/search_agents/main.py`，核心逻辑在 `modules/agents/search_agents/search_agent.py`。它负责两类能力：结构化找数，以及指标知识 RAG。结构化找数输出的是 Search DSL，不直接查表；知识 RAG 则直接生成面向用户的指标知识回答。

## 1. Agent 组成

| 名称 | 类型 | 功能 | Prompt / 依赖 |
| -- | -- | -- | -- |
| `search_agent` | `WorkflowAgent` | master，判断进入结构化找数还是知识 RAG | 无直接 prompt |
| `search_intent_agent` | `ReActAgent` | 判断 query 应走 `search_workflow_agent`、`search_rag_agent` 或未知 | `get_search_intent_agent_prompt()` |
| `search_workflow_agent` | `WorkflowAgent` | 结构化找数主流程 | 调用时间、指标、维度子 Agent |
| `search_ner_date_agent` | `ReActAgent` | 从 query 抽取时间描述 | `get_search_ner_date_agent_prompt()` |
| `search_nen_date_agent` | `ReActAgent` | 将时间描述归一成起止时间、粒度、实时/离线 | `get_search_nen_date_agent_prompt()` |
| `search_ner_ind_agent` | `ReActAgent` | 识别指标实体 | `get_search_ner_ind_agent_prompt()`，可调用 `search_ner_ind_rag_agent` |
| `search_ner_ind_rag_agent` | `ReActAgent` | 通过 RAG 候选辅助指标识别 | `get_search_ner_ind_rag_agent_prompt()` |
| `search_ner_dim_agent` | `ReActAgent` | 识别维度、维值和分组修饰 | `get_search_ner_dim_agent_prompt()` |
| `search_rag_agent` | `WorkflowAgent` | 指标知识问答主流程 | 调用 RAG NER 和总结 |
| `search_rag_ner_agent` | `ReActAgent` | 从问题中识别需要查知识的指标 | `get_search_rag_ner_agent_prompt()` |
| `search_rag_summary_agent` | `ChatAgent` | 基于召回知识生成回答 | `get_search_rag_summary_agent_prompt()` |

## 2. 结构化找数流程

`search_agent_func_workflow(req)` 会优先读取 `arguments.search_intent`。如果上游已经指定为 `search_workflow_agent`，它直接进入结构化找数；如果没有指定，则调用 `search_intent_agent` 判断。结构化找数的目标是把自然语言转换为时间、指标、维度、维值、权限视角和错误信息组成的 DSL。

`search_workflow_agent_func_workflow()` 首先调用 `get_auth_default_dim()` 读取用户的默认看数视角顺序，再通过权限服务获取用户可用维度组。没有权限时，它返回权限错误。随后流程调用 `search_ner_date_agent` 抽取时间表达，调用 `search_nen_date_agent` 做时间归一，并用 JimDB 缓存不涉及当前时刻的归一结果。接着调用 `search_ner_ind_agent` 识别指标，指标识别可借助 `search_ner_ind_rag_agent` 给出候选；每个指标名会通过 `search_nen_ind_tool()` 请求知识服务归一成指标实体。

如果某个指标带有修饰语，例如“3C 数码事业部”“按店铺”“按天”，流程会调用 `search_ner_dim_agent`。维度归一分两类：分组维度用 `group_search_nen_dim_tool()`，过滤维值用 `keyword_search_nen_dim_tool()`；这些函数会调用知识服务匹配 dimension 和 dimValue，并结合指标支持维度和用户权限做过滤。最终输出按原始 query 分组的候选结果，每个候选包含指标、时间、维度、分组和 errors。

```text
search_workflow_agent
    ↓
权限视角获取
    ↓
search_ner_date_agent
    ↓
search_nen_date_agent
    ↓
search_ner_ind_agent
    ↓
search_nen_ind_tool
    ↓
search_ner_dim_agent
    ↓
维度/维值归一与权限校验
    ↓
Search DSL
```

## 3. 指标知识 RAG 流程

`search_rag_agent_func_workflow()` 先调用 `search_rag_ner_agent`，识别用户问题关联的指标名。然后它调用 `search_rag_ind_tool()` 到知识服务召回指标知识，过滤低于 `similarity_indicator_score` 的结果。最后 workflow 构造“用户问题 + 参考知识”的 prompt，调用 `search_rag_summary_agent` 生成自然语言回答。

这个路径和结构化找数的差异很明确：结构化找数返回的是给 Query Agent 使用的 DSL，RAG 路径返回的是给用户看的知识解释。Datagent 在指标知识意图下会显式传 `search_intent=search_rag_agent`，避免 Search Agent 误入结构化找数。

## 4. 直接调用的工具函数

Search Agent 的 `tools/` 目录下多数函数不是 FunctionHub 工具，而是 workflow 直接 import 后调用。`search_nen_ind_tool()`、`group_search_nen_dim_tool()`、`keyword_search_nen_dim_tool()` 封装知识服务实体归一；`search_rag_ind_tool()` 和 `search_nen_ind_rag_tool()` 封装指标知识召回；`get_ind_default_dim()` 和 `get_auth_default_dim()` 读取 ES 或组织信息，用于默认维度和权限视角。

## 5. 示例

用户问“昨日 3C 数码事业部成交金额按天看”。Search Agent 会先确认这是结构化找数，然后时间节点抽取“昨日”，时间归一节点给出起止时间和粒度；指标节点识别“成交金额”并归一到指标 code；维度节点识别“3C 数码事业部”和“按天”，分别归一到事业部维值和 `dt` 分组。输出结果不是最终数据，而是一组候选 DSL，Query Agent 会继续使用它组装数据查询。

用户问“成交金额口径是什么”。如果进入 RAG 路径，`search_rag_ner_agent` 会识别“成交金额”，`search_rag_ind_tool()` 召回指标知识，`search_rag_summary_agent` 基于参考知识生成回答。如果召回为空，summary prompt 仍会收到“未获取到相关参考知识”，由模型按配置策略处理。
