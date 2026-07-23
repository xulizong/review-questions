# Datagent Intent Agent Prompt

本文解释当前 `datagent_intent_agent` 使用的意图识别 prompt 结构，以及它与 Datagent 代码的关系。完整原始 prompt 单独保存在 [datagent_intent_prompt_raw.md](datagent_intent_prompt_raw.md)，便于逐字查看和后续对照 LAF 配置。代码侧通过 `modules/agents/datagents/laf_clients.py` 中的 `get_datagent_intent_agent_prompt()` 从 LAF key `datagent_intent_agent_prompt` 获取线上 prompt。

## 1. Prompt 定位

`datagent_intent_agent` 的任务不是回答用户问题，而是把用户问题和历史上下文分类成 Datagent 可路由的意图。如果识别为 `datagent-query`，prompt 还要求继续识别二级 `query_operator`，并在必要时输出 `rewritten_query`。代码侧 `datagent_intent_agent_func_parse_llm_response()` 只强制检查 `has_intent` 和 `intent_type`，但 `datagent_func_workflow()` 后续会读取 `query_operator` 和 `rewritten_query`，因此 prompt 的字段约定会直接影响路由是否成功。

## 2. 一级意图

| intent_type | Prompt 定义 | 当前 datagent workflow 是否支持 | 说明 |
| -- | -- | -- | -- |
| `datagent-ge-analyze` | 指标分析、趋势、归因、洞察 | 否 | Prompt 已定义，但 `support_intents` 未包含，当前会被判为不支持 |
| `datagent-query` | 获取业务数据、指标数值、图表、维度枚举、排序筛选等 | 是 | 代码额外要求 `query_operator == "direct"` |
| `datagent-search-knowledge` | 指标定义、口径、SQL、计算逻辑、含义解释 | 是 | 路由到远程 `search_agent` 的 RAG 路径 |
| `datagent-business-knowledge` | 资源定位、操作指南、权限、负责人、归属映射 | 是 | 路由到本地 `get_business_knowledge` |
| `datagent-page-summary` | 当前页面总结、摘要或解读 | 是 | 路由到远程 `ge_ai_page_summary_agent` |
| `datagent-shop-diagnostic` | 固定前缀的店铺星级诊断 | 是 | 路由到本地 `shop_diagnostic_agent` |
| `datagent-chat-clarification` | 闲聊、问候、非数据类模糊输入 | 否 | Prompt 已定义，但当前会被判为不支持 |

## 3. 二级算子

`query_operator` 只在 `intent_type=datagent-query` 时输出。Prompt 定义了 `direct`、`calculation`、`filter`、`analysis`、`dimension_value` 五类；但当前 Datagent 代码只接受 `direct`，其他算子会在 `check_support_intent_and_rewrite()` 中被判为不支持并返回 `get_datagent_intent_reply()`。这说明 prompt 已经为更复杂的数据能力预留了分类，但 Datagent 当前只把基础直查转给 `query_agent`。

| query_operator | Prompt 语义 | 当前代码行为 |
| -- | -- | -- |
| `direct` | 基础直查、标准聚合、分组、排序、同环比、预定义指标查询 | 允许进入 `query_agent` |
| `calculation` | 二次运算、多指标逻辑计算、双时段/双对象对比 | 当前不支持 |
| `filter` | 基于指标数值阈值筛选或极值查找 | 当前不支持 |
| `analysis` | 分析洞察类，prompt 建议升级为 `datagent-ge-analyze` | 当前不支持 |
| `dimension_value` | 维度值枚举、字典、元数据验证 | 当前不支持 |

## 4. 改写逻辑

Prompt 要求当 `intent_type` 为 `datagent-query` 或 `datagent-ge-analyze` 且算子不是 `dimension_value` 时，判断是否需要改写。需要改写的典型原因是缺时间、缺主体、缺指标或存在代词指代。改写时同时处理实体/指标继承和时间补全：当前 query 有时间就使用当前时间；当前没有时间但历史最近一轮有时间，就继承历史时间；两者都没有时默认补“昨天”。

这部分与代码的连接点是 `rewritten_query`。如果 prompt 输出 `need_rewrite=true` 且给出 `rewritten_query`，Datagent 在 `datagent-query` 分支会优先把 `rewritten_query` 传给 `query_agent`。如果 `need_rewrite=false`，prompt 规范要求不要输出 `rewritten_query`，Datagent 会使用原 query。

## 5. 输出 JSON 约束

Prompt 要求输出标准 JSON，不要 Markdown。核心模板如下：

```json
{
  "has_intent": true,
  "intent_type": "datagent-query",
  "query_operator": "direct",
  "need_rewrite": false
}
```

当 `need_rewrite=true` 时才允许输出 `rewritten_query`：

```json
{
  "has_intent": true,
  "intent_type": "datagent-query",
  "query_operator": "direct",
  "need_rewrite": true,
  "rewritten_query": "昨天成交金额"
}
```

## 6. 关键判例

| 用户输入 | 预期意图 | 关键原因 |
| -- | -- | -- |
| `成交金额` | `datagent-query`, `direct`, 改写为 `昨天成交金额` | 孤立业务指标默认查昨天 |
| `GMV的口径是什么` | `datagent-search-knowledge` | 问定义、口径、计算逻辑 |
| `搜索词热点在哪里` | `datagent-business-knowledge` | 问操作路径，不是查数 |
| `店铺ID888888店铺星级：为我诊断下店铺星级` | `datagent-shop-diagnostic` | 前缀强制匹配 |
| `最近GMV如何` | `datagent-ge-analyze` | 定性分析和表现判断 |
| `昨天成交金额同比小于0的品牌有哪些` | `datagent-query`, `filter` | 指标阈值筛选，但当前代码不支持该算子 |

## 7. 当前实现风险

这份 prompt 的能力范围比当前 Datagent workflow 更宽。Prompt 会输出 `datagent-ge-analyze`、`datagent-chat-clarification`、`calculation`、`filter`、`dimension_value` 等分类，但 `datagent.py` 目前只允许 `datagent-query` 且 `query_operator=direct` 进入查数，其余会被判为不支持。后续如果要接入分析主管、维度枚举或筛选计算能力，需要同步修改 `support_intents` 和分支路由，而不是只改 prompt。

## 8. Prompt 原文位置

当前文档负责解释 prompt 的结构和它与代码的连接关系。逐字原文单独保存在 [datagent_intent_prompt_raw.md](datagent_intent_prompt_raw.md)，该文件直接来自本次提供的 prompt 附件，后续如果 LAF 中的 `datagent_intent_agent_prompt` 发生变化，应优先更新 raw 文档，再同步调整本文的意图、算子和兼容性说明。
